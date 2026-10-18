# FortiGate 与 CSR IPsec ECMP 实验手册

Oct 18, 2026 · @voight

## 实验目标与设计要点

FortiGate 与 CSR2、CSR3 之间各建 3 条 IKEv2 路由型 IPsec 隧道（共 6 条），每条隧道上跑一个 eBGP 邻居，业务流量按 ECMP 分担到 3 条隧道上，再用两端服务器做 iperf3 压测。

- **隧道端点用物理口地址，BGP 用隧道 overlay 地址**：端点是直连网段，overlay 是独立的 172.16.x.x/30，不存在递归路由问题。
- **同一对端点只能建一条隧道**：所以 CSR 物理口加两个 secondary 地址，3 条隧道各用一个源地址，FortiGate 侧每个口一个地址不变。
- **路由型（VTI）对接**：思科 sVTI 的选择子固定 0/0，FortiGate phase2 也保持 0/0；感兴趣流量由路由决定，不写 ACL。
- **ECMP**：FortiGate 开 `ebgp-multipath`，CSR 配 `maximum-paths 3`；两端都按源+目的 IP 哈希，同一对 IP 只走一条隧道。
- **防泄漏**：两端 prefix-list 只收发业务网段，不收发 192.168.x 和 172.16.x；各自配一条 distance 254 的黑洞路由兜底。
- **CSR2 与 CSR3 互通**经 FortiGate 转发，依赖 Zone 的 `intrazone allow`。



## 地址规划

### Underlay（隧道端点）

| 设备 | 接口 | 主地址 | Secondary |
| --- | --- | --- | --- |
| FortiGate | port1 | 192.168.10.1/24 | — |
| CSR2 | Gi1 | 192.168.10.2/24 | 192.168.10.3、192.168.10.4 |
| FortiGate | port2 | 192.168.20.1/24 | — |
| CSR3 | Gi2 | 192.168.20.2/24 | 192.168.20.3、192.168.20.4 |

### 隧道与 overlay（BGP 邻居地址）

| FortiGate phase1 | 端点（FGT ↔ CSR） | CSR 接口 | 互联网段 | FGT 隧道 IP | CSR 隧道 IP |
| --- | --- | --- | --- | --- | --- |
| CSR2-T1 | 192.168.10.1 ↔ 192.168.10.2 | CSR2 Tunnel1 | 172.16.10.0/30 | 172.16.10.1 | 172.16.10.2 |
| CSR2-T2 | 192.168.10.1 ↔ 192.168.10.3 | CSR2 Tunnel2 | 172.16.10.4/30 | 172.16.10.5 | 172.16.10.6 |
| CSR2-T3 | 192.168.10.1 ↔ 192.168.10.4 | CSR2 Tunnel3 | 172.16.10.8/30 | 172.16.10.9 | 172.16.10.10 |
| CSR3-T1 | 192.168.20.1 ↔ 192.168.20.2 | CSR3 Tunnel1 | 172.16.20.0/30 | 172.16.20.1 | 172.16.20.2 |
| CSR3-T2 | 192.168.20.1 ↔ 192.168.20.3 | CSR3 Tunnel2 | 172.16.20.4/30 | 172.16.20.5 | 172.16.20.6 |
| CSR3-T3 | 192.168.20.1 ↔ 192.168.20.4 | CSR3 Tunnel3 | 172.16.20.8/30 | 172.16.20.9 | 172.16.20.10 |

### BGP 与业务网段

| 设备 | AS | Router-ID | 业务网段 | 业务接口 |
| --- | --- | --- | --- | --- |
| FortiGate | 65000 | 10.255.0.1 | 10.0.0.0/24 | port3 · 10.0.0.1 |
| CSR2 | 65002 | 10.255.0.2 | 10.2.2.0/24 | Gi2 · 10.2.2.1 |
| CSR3 | 65003 | 10.255.0.3 | 10.3.3.0/24 | Gi3 · 10.3.3.1 |

### 压测服务器

| 服务器 | 接入 | 地址 | 网关 |
| --- | --- | --- | --- |
| Server-FGT | FortiGate port3 | 10.0.0.10–13/24 | 10.0.0.1 |
| Server-CSR2 | CSR2 Gi2 | 10.2.2.10–13/24 | 10.2.2.1 |
| Server-CSR3 | CSR3 Gi3 | 10.3.3.10–13/24 | 10.3.3.1 |

每台服务器 4 个地址，4×4 = 16 对源/目的 IP，才能让按 IP 哈希的 ECMP 把流量分到 3 条隧道上。

## 加密参数与两端对应关系

两端 phase1 和 phase2 都是 DES-SHA256、DH14，预共享密钥 `CC!E!nfr4`（FortiGate 写成 `"CC!E!nfr4"` 带引号，思科不加引号）。

| 项目 | 取值 | FortiGate | 思科 |
| --- | --- | --- | --- |
| IKE 版本 | IKEv2 | phase1-interface `ike-version 2` | `crypto ikev2 proposal / policy / profile` |
| IKE 加密 / 完整性 / DH | DES / SHA256 / 14 | phase1 `proposal des-sha256` + `dhgrp 14` | `crypto ikev2 proposal`：`encryption des`、`integrity sha256`、`group 14` |
| 认证 | 预共享密钥 | phase1 `psksecret` + `remote-gw` | `crypto ikev2 keyring` + profile 的 `match identity` |
| IKE SA 寿命 | 86400 s | phase1 `keylife` | ikev2 profile `lifetime` |
| ESP 加密 / 完整性 | DES / SHA256 | phase2-interface `proposal des-sha256` | `crypto ipsec transform-set esp-des esp-sha256-hmac` |
| PFS | DH14 | phase2 `pfs enable` + `dhgrp 14` | ipsec profile `set pfs group14` |
| IPsec SA 寿命 | 43200 s | phase2 `keylifeseconds` | ipsec profile `set security-association lifetime seconds` |
| 选择子 | 0/0 ↔ 0/0 | phase2 默认 0/0 | sVTI 隐含 0/0 |
| 隧道接口 | — | 与 phase1 同名的接口 + `ip` / `remote-ip` | `interface TunnelX` + `tunnel protection ipsec profile` |

思科每条隧道单独一个 ikev2 profile 和 ipsec profile，用 `match address local` 按本端 secondary 地址区分。

如果 IOS-XE 版本在 17.7 及以上，配置 DES 前需要先执行 `crypto engine compliance shield disable`，然后重启。

## FortiGate 完整配置

FortiOS 7.x，按段顺序粘贴；phase1 必须先于隧道接口地址配置，否则隧道接口还不存在。

```
config system interface
    edit "port1"
        set mode static
        set ip 192.168.10.1 255.255.255.0
        set allowaccess ping
    next
    edit "port2"
        set mode static
        set ip 192.168.20.1 255.255.255.0
        set allowaccess ping
    next
    edit "Lo0"
        set vdom "root"
        set type loopback
        set ip 10.255.0.1 255.255.255.255
        set allowaccess ping
    next
    edit "port3"
        set mode static
        set ip 10.0.0.1 255.255.255.0
        set allowaccess ping
        set role lan
        set description "To-Server-FGT"
    next
end

# v4-ecmp-mode：按源+目的 IP 哈希；auxiliary-session：回程走另一条隧道时不丢包
config system settings
    set v4-ecmp-mode source-dest-ip-based
    set auxiliary-session enable
end

config vpn ipsec phase1-interface
    edit "CSR2-T1"
        set interface "port1"
        set ike-version 2
        set peertype any
        set proposal des-sha256
        set dhgrp 14
        set keylife 86400
        set remote-gw 192.168.10.2
        set dpd on-idle
        set dpd-retryinterval 10
        set psksecret "CC!E!nfr4"
    next
    edit "CSR2-T2"
        set interface "port1"
        set ike-version 2
        set peertype any
        set proposal des-sha256
        set dhgrp 14
        set keylife 86400
        set remote-gw 192.168.10.3
        set dpd on-idle
        set dpd-retryinterval 10
        set psksecret "CC!E!nfr4"
    next
    edit "CSR2-T3"
        set interface "port1"
        set ike-version 2
        set peertype any
        set proposal des-sha256
        set dhgrp 14
        set keylife 86400
        set remote-gw 192.168.10.4
        set dpd on-idle
        set dpd-retryinterval 10
        set psksecret "CC!E!nfr4"
    next
    edit "CSR3-T1"
        set interface "port2"
        set ike-version 2
        set peertype any
        set proposal des-sha256
        set dhgrp 14
        set keylife 86400
        set remote-gw 192.168.20.2
        set dpd on-idle
        set dpd-retryinterval 10
        set psksecret "CC!E!nfr4"
    next
    edit "CSR3-T2"
        set interface "port2"
        set ike-version 2
        set peertype any
        set proposal des-sha256
        set dhgrp 14
        set keylife 86400
        set remote-gw 192.168.20.3
        set dpd on-idle
        set dpd-retryinterval 10
        set psksecret "CC!E!nfr4"
    next
    edit "CSR3-T3"
        set interface "port2"
        set ike-version 2
        set peertype any
        set proposal des-sha256
        set dhgrp 14
        set keylife 86400
        set remote-gw 192.168.20.4
        set dpd on-idle
        set dpd-retryinterval 10
        set psksecret "CC!E!nfr4"
    next
end

config vpn ipsec phase2-interface
    edit "CSR2-T1-p2"
        set phase1name "CSR2-T1"
        set proposal des-sha256
        set pfs enable
        set dhgrp 14
        set keylifeseconds 43200
        set auto-negotiate enable
    next
    edit "CSR2-T2-p2"
        set phase1name "CSR2-T2"
        set proposal des-sha256
        set pfs enable
        set dhgrp 14
        set keylifeseconds 43200
        set auto-negotiate enable
    next
    edit "CSR2-T3-p2"
        set phase1name "CSR2-T3"
        set proposal des-sha256
        set pfs enable
        set dhgrp 14
        set keylifeseconds 43200
        set auto-negotiate enable
    next
    edit "CSR3-T1-p2"
        set phase1name "CSR3-T1"
        set proposal des-sha256
        set pfs enable
        set dhgrp 14
        set keylifeseconds 43200
        set auto-negotiate enable
    next
    edit "CSR3-T2-p2"
        set phase1name "CSR3-T2"
        set proposal des-sha256
        set pfs enable
        set dhgrp 14
        set keylifeseconds 43200
        set auto-negotiate enable
    next
    edit "CSR3-T3-p2"
        set phase1name "CSR3-T3"
        set proposal des-sha256
        set pfs enable
        set dhgrp 14
        set keylifeseconds 43200
        set auto-negotiate enable
    next
end

config system interface
    edit "CSR2-T1"
        set ip 172.16.10.1 255.255.255.255
        set remote-ip 172.16.10.2 255.255.255.252
        set allowaccess ping
    next
    edit "CSR2-T2"
        set ip 172.16.10.5 255.255.255.255
        set remote-ip 172.16.10.6 255.255.255.252
        set allowaccess ping
    next
    edit "CSR2-T3"
        set ip 172.16.10.9 255.255.255.255
        set remote-ip 172.16.10.10 255.255.255.252
        set allowaccess ping
    next
    edit "CSR3-T1"
        set ip 172.16.20.1 255.255.255.255
        set remote-ip 172.16.20.2 255.255.255.252
        set allowaccess ping
    next
    edit "CSR3-T2"
        set ip 172.16.20.5 255.255.255.255
        set remote-ip 172.16.20.6 255.255.255.252
        set allowaccess ping
    next
    edit "CSR3-T3"
        set ip 172.16.20.9 255.255.255.255
        set remote-ip 172.16.20.10 255.255.255.252
        set allowaccess ping
    next
end

config system zone
    edit "VPN-Zone"
        set interface "CSR2-T1" "CSR2-T2" "CSR2-T3" "CSR3-T1" "CSR3-T2" "CSR3-T3"
        set intrazone allow
    next
end

config firewall address
    edit "FGT-LAN"
        set subnet 10.0.0.0 255.255.255.0
    next
    edit "CSR2-LAN"
        set subnet 10.2.2.0 255.255.255.0
    next
    edit "CSR3-LAN"
        set subnet 10.3.3.0 255.255.255.0
    next
end
config firewall addrgrp
    edit "Remote-LANs"
        set member "CSR2-LAN" "CSR3-LAN"
    next
end

config firewall policy
    edit 0
        set name "VPN-to-LAN"
        set srcintf "VPN-Zone"
        set dstintf "port3"
        set srcaddr "Remote-LANs"
        set dstaddr "FGT-LAN"
        set action accept
        set schedule "always"
        set service "ALL"
        set logtraffic all
    next
    edit 0
        set name "LAN-to-VPN"
        set srcintf "port3"
        set dstintf "VPN-Zone"
        set srcaddr "FGT-LAN"
        set dstaddr "Remote-LANs"
        set action accept
        set schedule "always"
        set service "ALL"
        set logtraffic all
    next
end

config router static
    edit 0
        set dst 10.2.2.0 255.255.255.0
        set blackhole enable
        set distance 254
    next
    edit 0
        set dst 10.3.3.0 255.255.255.0
        set blackhole enable
        set distance 254
    next
end

config router prefix-list
    edit "FROM-CSR2"
        config rule
            edit 1
                set prefix 10.2.2.0 255.255.255.0
                unset ge
                unset le
            next
        end
    next
    edit "FROM-CSR3"
        config rule
            edit 1
                set prefix 10.3.3.0 255.255.255.0
                unset ge
                unset le
            next
        end
    next
    edit "TO-CSR"
        config rule
            edit 1
                set prefix 10.0.0.0 255.255.255.0
                unset ge
                unset le
            next
            edit 2
                set prefix 10.2.2.0 255.255.255.0
                unset ge
                unset le
            next
            edit 3
                set prefix 10.3.3.0 255.255.255.0
                unset ge
                unset le
            next
        end
    next
end

config router bgp
    set as 65000
    set router-id 10.255.0.1
    set ebgp-multipath enable
    config neighbor
        edit "172.16.10.2"
            set remote-as 65002
            set keep-alive-timer 10
            set holdtime-timer 30
            set soft-reconfiguration enable
            set prefix-list-in "FROM-CSR2"
            set prefix-list-out "TO-CSR"
        next
        edit "172.16.10.6"
            set remote-as 65002
            set keep-alive-timer 10
            set holdtime-timer 30
            set soft-reconfiguration enable
            set prefix-list-in "FROM-CSR2"
            set prefix-list-out "TO-CSR"
        next
        edit "172.16.10.10"
            set remote-as 65002
            set keep-alive-timer 10
            set holdtime-timer 30
            set soft-reconfiguration enable
            set prefix-list-in "FROM-CSR2"
            set prefix-list-out "TO-CSR"
        next
        edit "172.16.20.2"
            set remote-as 65003
            set keep-alive-timer 10
            set holdtime-timer 30
            set soft-reconfiguration enable
            set prefix-list-in "FROM-CSR3"
            set prefix-list-out "TO-CSR"
        next
        edit "172.16.20.6"
            set remote-as 65003
            set keep-alive-timer 10
            set holdtime-timer 30
            set soft-reconfiguration enable
            set prefix-list-in "FROM-CSR3"
            set prefix-list-out "TO-CSR"
        next
        edit "172.16.20.10"
            set remote-as 65003
            set keep-alive-timer 10
            set holdtime-timer 30
            set soft-reconfiguration enable
            set prefix-list-in "FROM-CSR3"
            set prefix-list-out "TO-CSR"
        next
    end
    config network
        edit 1
            set prefix 10.0.0.0 255.255.255.0
        next
    end
end
```

压测时如果 CPU 吃紧，可以把两条策略的 `logtraffic all` 改成 `disable`。

## CSR2 完整配置

IOS-XE（CSR1000v / Catalyst 8000v）。3 条 sVTI 各用一个本端地址作源，ikev2 profile 用 `match address local` 区分。

```
hostname CSR2
!
interface GigabitEthernet1
 description To-FortiGate-port1
 ip address 192.168.10.2 255.255.255.0
 ip address 192.168.10.3 255.255.255.0 secondary
 ip address 192.168.10.4 255.255.255.0 secondary
 no shutdown
!
interface Loopback0
 ip address 10.255.0.2 255.255.255.255
!
interface GigabitEthernet2
 description To-Server-CSR2
 ip address 10.2.2.1 255.255.255.0
 no shutdown
!
crypto ikev2 proposal FGT-PROP
 encryption des
 integrity sha256
 group 14
!
crypto ikev2 policy FGT-POL
 proposal FGT-PROP
!
crypto ikev2 keyring FGT-KEY
 peer FGT
  address 192.168.10.1
  pre-shared-key CC!E!nfr4
 !
!
crypto ikev2 profile FGT-PROF1
 match address local 192.168.10.2
 match identity remote address 192.168.10.1 255.255.255.255
 identity local address 192.168.10.2
 authentication remote pre-share
 authentication local pre-share
 keyring local FGT-KEY
 lifetime 86400
 dpd 10 3 on-demand
!
crypto ikev2 profile FGT-PROF2
 match address local 192.168.10.3
 match identity remote address 192.168.10.1 255.255.255.255
 identity local address 192.168.10.3
 authentication remote pre-share
 authentication local pre-share
 keyring local FGT-KEY
 lifetime 86400
 dpd 10 3 on-demand
!
crypto ikev2 profile FGT-PROF3
 match address local 192.168.10.4
 match identity remote address 192.168.10.1 255.255.255.255
 identity local address 192.168.10.4
 authentication remote pre-share
 authentication local pre-share
 keyring local FGT-KEY
 lifetime 86400
 dpd 10 3 on-demand
!
crypto ipsec transform-set FGT-TS esp-des esp-sha256-hmac
 mode tunnel
!
crypto ipsec profile FGT-IPSEC1
 set transform-set FGT-TS
 set pfs group14
 set security-association lifetime seconds 43200
 set ikev2-profile FGT-PROF1
!
crypto ipsec profile FGT-IPSEC2
 set transform-set FGT-TS
 set pfs group14
 set security-association lifetime seconds 43200
 set ikev2-profile FGT-PROF2
!
crypto ipsec profile FGT-IPSEC3
 set transform-set FGT-TS
 set pfs group14
 set security-association lifetime seconds 43200
 set ikev2-profile FGT-PROF3
!
interface Tunnel1
 description To-FGT-CSR2-T1
 ip address 172.16.10.2 255.255.255.252
 ip mtu 1400
 ip tcp adjust-mss 1360
 tunnel source 192.168.10.2
 tunnel mode ipsec ipv4
 tunnel destination 192.168.10.1
 tunnel protection ipsec profile FGT-IPSEC1
!
interface Tunnel2
 description To-FGT-CSR2-T2
 ip address 172.16.10.6 255.255.255.252
 ip mtu 1400
 ip tcp adjust-mss 1360
 tunnel source 192.168.10.3
 tunnel mode ipsec ipv4
 tunnel destination 192.168.10.1
 tunnel protection ipsec profile FGT-IPSEC2
!
interface Tunnel3
 description To-FGT-CSR2-T3
 ip address 172.16.10.10 255.255.255.252
 ip mtu 1400
 ip tcp adjust-mss 1360
 tunnel source 192.168.10.4
 tunnel mode ipsec ipv4
 tunnel destination 192.168.10.1
 tunnel protection ipsec profile FGT-IPSEC3
!
ip route 10.0.0.0 255.255.255.0 Null0 254
ip route 10.3.3.0 255.255.255.0 Null0 254
!
ip prefix-list FROM-FGT seq 10 permit 10.0.0.0/24
ip prefix-list FROM-FGT seq 20 permit 10.3.3.0/24
ip prefix-list TO-FGT seq 10 permit 10.2.2.0/24
!
router bgp 65002
 bgp router-id 10.255.0.2
 bgp log-neighbor-changes
 neighbor FGT peer-group
 neighbor FGT remote-as 65000
 neighbor FGT timers 10 30
 neighbor 172.16.10.1 peer-group FGT
 neighbor 172.16.10.1 description FGT-via-Tunnel1
 neighbor 172.16.10.5 peer-group FGT
 neighbor 172.16.10.5 description FGT-via-Tunnel2
 neighbor 172.16.10.9 peer-group FGT
 neighbor 172.16.10.9 description FGT-via-Tunnel3
 !
 address-family ipv4
  network 10.2.2.0 mask 255.255.255.0
  maximum-paths 3
  neighbor FGT soft-reconfiguration inbound
  neighbor FGT prefix-list FROM-FGT in
  neighbor FGT prefix-list TO-FGT out
  neighbor 172.16.10.1 activate
  neighbor 172.16.10.5 activate
  neighbor 172.16.10.9 activate
 exit-address-family
!
end
```

## CSR3 完整配置

结构与 CSR2 相同：对 FortiGate 的口是 Gi2，服务器接在 Gi3，AS 65003。

```
hostname CSR3
!
interface GigabitEthernet2
 description To-FortiGate-port2
 ip address 192.168.20.2 255.255.255.0
 ip address 192.168.20.3 255.255.255.0 secondary
 ip address 192.168.20.4 255.255.255.0 secondary
 no shutdown
!
interface Loopback0
 ip address 10.255.0.3 255.255.255.255
!
interface GigabitEthernet3
 description To-Server-CSR3
 ip address 10.3.3.1 255.255.255.0
 no shutdown
!
crypto ikev2 proposal FGT-PROP
 encryption des
 integrity sha256
 group 14
!
crypto ikev2 policy FGT-POL
 proposal FGT-PROP
!
crypto ikev2 keyring FGT-KEY
 peer FGT
  address 192.168.20.1
  pre-shared-key CC!E!nfr4
 !
!
crypto ikev2 profile FGT-PROF1
 match address local 192.168.20.2
 match identity remote address 192.168.20.1 255.255.255.255
 identity local address 192.168.20.2
 authentication remote pre-share
 authentication local pre-share
 keyring local FGT-KEY
 lifetime 86400
 dpd 10 3 on-demand
!
crypto ikev2 profile FGT-PROF2
 match address local 192.168.20.3
 match identity remote address 192.168.20.1 255.255.255.255
 identity local address 192.168.20.3
 authentication remote pre-share
 authentication local pre-share
 keyring local FGT-KEY
 lifetime 86400
 dpd 10 3 on-demand
!
crypto ikev2 profile FGT-PROF3
 match address local 192.168.20.4
 match identity remote address 192.168.20.1 255.255.255.255
 identity local address 192.168.20.4
 authentication remote pre-share
 authentication local pre-share
 keyring local FGT-KEY
 lifetime 86400
 dpd 10 3 on-demand
!
crypto ipsec transform-set FGT-TS esp-des esp-sha256-hmac
 mode tunnel
!
crypto ipsec profile FGT-IPSEC1
 set transform-set FGT-TS
 set pfs group14
 set security-association lifetime seconds 43200
 set ikev2-profile FGT-PROF1
!
crypto ipsec profile FGT-IPSEC2
 set transform-set FGT-TS
 set pfs group14
 set security-association lifetime seconds 43200
 set ikev2-profile FGT-PROF2
!
crypto ipsec profile FGT-IPSEC3
 set transform-set FGT-TS
 set pfs group14
 set security-association lifetime seconds 43200
 set ikev2-profile FGT-PROF3
!
interface Tunnel1
 description To-FGT-CSR3-T1
 ip address 172.16.20.2 255.255.255.252
 ip mtu 1400
 ip tcp adjust-mss 1360
 tunnel source 192.168.20.2
 tunnel mode ipsec ipv4
 tunnel destination 192.168.20.1
 tunnel protection ipsec profile FGT-IPSEC1
!
interface Tunnel2
 description To-FGT-CSR3-T2
 ip address 172.16.20.6 255.255.255.252
 ip mtu 1400
 ip tcp adjust-mss 1360
 tunnel source 192.168.20.3
 tunnel mode ipsec ipv4
 tunnel destination 192.168.20.1
 tunnel protection ipsec profile FGT-IPSEC2
!
interface Tunnel3
 description To-FGT-CSR3-T3
 ip address 172.16.20.10 255.255.255.252
 ip mtu 1400
 ip tcp adjust-mss 1360
 tunnel source 192.168.20.4
 tunnel mode ipsec ipv4
 tunnel destination 192.168.20.1
 tunnel protection ipsec profile FGT-IPSEC3
!
ip route 10.0.0.0 255.255.255.0 Null0 254
ip route 10.2.2.0 255.255.255.0 Null0 254
!
ip prefix-list FROM-FGT seq 10 permit 10.0.0.0/24
ip prefix-list FROM-FGT seq 20 permit 10.2.2.0/24
ip prefix-list TO-FGT seq 10 permit 10.3.3.0/24
!
router bgp 65003
 bgp router-id 10.255.0.3
 bgp log-neighbor-changes
 neighbor FGT peer-group
 neighbor FGT remote-as 65000
 neighbor FGT timers 10 30
 neighbor 172.16.20.1 peer-group FGT
 neighbor 172.16.20.1 description FGT-via-Tunnel1
 neighbor 172.16.20.5 peer-group FGT
 neighbor 172.16.20.5 description FGT-via-Tunnel2
 neighbor 172.16.20.9 peer-group FGT
 neighbor 172.16.20.9 description FGT-via-Tunnel3
 !
 address-family ipv4
  network 10.3.3.0 mask 255.255.255.0
  maximum-paths 3
  neighbor FGT soft-reconfiguration inbound
  neighbor FGT prefix-list FROM-FGT in
  neighbor FGT prefix-list TO-FGT out
  neighbor 172.16.20.1 activate
  neighbor 172.16.20.5 activate
  neighbor 172.16.20.9 activate
 exit-address-family
!
end
```

## 压测服务器与 iperf3 压测

三台服务器各配 4 个地址，跑 `stress-test.sh` 同时发起 16 对 IP 的流量，结束后汇总每对速率与总计。网卡名以 `ens192` 为例，先用 `ip -br addr` 确认。

### 服务器地址（临时，重启失效）

```
# Server-FGT
ip addr add 10.0.0.11/24 dev ens192
ip addr add 10.0.0.12/24 dev ens192
ip addr add 10.0.0.13/24 dev ens192
ip route replace 10.0.0.0/8 via 10.0.0.1 dev ens192

# Server-CSR2
ip addr add 10.2.2.11/24 dev ens192
ip addr add 10.2.2.12/24 dev ens192
ip addr add 10.2.2.13/24 dev ens192
ip route replace 10.0.0.0/8 via 10.2.2.1 dev ens192

# Server-CSR3
ip addr add 10.3.3.11/24 dev ens192
ip addr add 10.3.3.12/24 dev ens192
ip addr add 10.3.3.13/24 dev ens192
ip route replace 10.0.0.0/8 via 10.3.3.1 dev ens192
```

.10 若尚未配置，先补上。永久配置：Ubuntu 写进 netplan 的 `addresses` 和 `routes`；RHEL 系用 `nmcli con mod <连接名> +ipv4.addresses "…"` 和 `+ipv4.routes "10.0.0.0/8 <网关>"`。

### 压测步骤

1. 三台服务器安装 iperf3 和 python3，放行 TCP/UDP 5201–5248（或临时关闭 firewalld/ufw）。
2. 压测前在 CSR 上执行 `show platform hardware throughput level`；未授权的 CSR1000v、评估许可证的 FortiGate VM 都会限速，测出的是许可证上限。
3. 接收端启动服务端：`./stress-test.sh server`（48 个端口，CSR2 和 CSR3 可同时往同一台打）。
4. 发送端执行，例如在 Server-CSR2 上：
   - TCP：`./stress-test.sh tcp fgt 60 4`（16 对 IP、每对 4 条流、60 秒）
   - UDP：`./stress-test.sh udp fgt 20M 60`（每对 20 Mbit/s，共 320 Mbit/s）
   - 反向或双向：前面加 `EXTRA=-R` 或 `EXTRA=--bidir`
5. 打流期间看三条隧道是否都有流量：FortiGate `get vpn ipsec tunnel summary`，CSR `show interfaces Tunnel1 | include rate`（Tunnel2、3 同样）。
6. 故障切换：打流中 `shutdown` 一条 Tunnel，记录中断时间；按 30 秒 hold time 约中断 30 秒，开 BFD 可压到秒级以内。
7. 结束：`./stress-test.sh stop`；JSON 原始结果在 `./iperf-logs`。

同一对 IP 无论开多少条 TCP 流都只走一条隧道，所以不能只用 `-P` 加并发来测 ECMP。

## 验证与排错

按“隧道 → 隧道互联地址 → BGP → 路由表 → 业务互通”的顺序逐层检查；BGP 卡在 Active 时，先确认对应隧道两端的 /30 地址能互 ping。

### FortiGate

```
get vpn ipsec tunnel summary                # 6 条隧道 up，rx/tx 计数
execute ping-options source 172.16.10.1
execute ping 172.16.10.2                    # 逐条测隧道互联地址
get router info bgp summary                 # 6 个邻居 Established
get router info routing-table bgp           # 10.2.2.0/24、10.3.3.0/24 各 3 条下一跳
get router info bgp network 10.2.2.0/24     # multipath 标记
get system performance status               # 压测时 CPU、内存、会话数

# IKE 排错
diagnose vpn ike log-filter rem-addr4 192.168.10.3
diagnose debug application ike -1
diagnose debug enable
```

### CSR2 / CSR3

```
show crypto ikev2 sa
show crypto ipsec sa | include interface|pkts encaps|pkts decaps
ping 172.16.10.1 source Tunnel1             # CSR3 用 172.16.20.x
show ip bgp summary
show ip route 10.0.0.0                      # 应有 3 条下一跳
show ip cef exact-route 10.2.2.10 10.0.0.10 # 某条流走哪条隧道
show platform hardware throughput level
show platform hardware qfp active datapath utilization summary

debug crypto ikev2
debug crypto ipsec                          # 结束 undebug all
```

### 常见问题

| 现象 | 常见原因 | 处理 |
| --- | --- | --- |
| phase1 起不来 | 加密参数或 PSK 不一致；思科不认 DES | 核对 DES-SHA256/DH14 与密钥；IOS-XE 17.7+ 先 `crypto engine compliance shield disable` 并重启 |
| phase1 up、phase2 down | 选择子不一致，或 PFS 不一致 | 两端都保持 0/0；FortiGate `pfs enable` 对应思科 `set pfs group14` |
| 某条隧道找错 profile | ikev2 profile 没按本端地址区分 | 每个 profile 都写 `match address local` + `identity local address` |
| BGP 停在 Active | 隧道没通，或邻居地址 / AS 写错 | 先 ping 隧道 /30 对端，再核对 neighbor 和 remote-as |
| 路由表只有 1 条下一跳 | 没开多路径，或 3 条路径属性不一致 | FortiGate `ebgp-multipath enable`，CSR `maximum-paths 3` |
| 压测只有一条隧道有流量 | 源/目的 IP 对太少 | 按 4×4 地址跑脚本，不要只加 `-P` |
| 吞吐远低于预期 | 许可证限速 | 检查 CSR throughput level 与 FortiGate VM 许可证 |
| 大包不通、小包通 | MTU / MSS | 隧道 `ip mtu 1400`、`ip tcp adjust-mss 1360`；UDP 用 `-l 1300` |
