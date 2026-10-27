# Versa VOS as an Inter-AS Option C PE: BGP Labeled Unicast + VPNv4 L3VPN with Cisco IOS XE

Eniz Aksoy · 27 Oct 2026 · CML lab notes. Companion files: `config/` (VOS set config), `outputs/` (verbatim CLI outputs), `captures/` (pcap), `docs/` (Word/PDF with colour-coded CLI).

VOS 23.1.2 works as an Inter-AS Option C PE: eBGP labeled unicast to a Cisco ASBR, multihop eBGP VPNv4 to a remote Cisco PE, and an L3VPN VRF with end-to-end forwarding, verified in the lab on 26–27 Oct 2026.

## Purpose

This article shows how to run a standalone VOS appliance as an Inter-AS Option C PE against Cisco IOS XE. VOS exchanges BGP labeled unicast (BGP-LU) with a Cisco ASBR over a plain Ethernet link and runs a multihop eBGP VPNv4 session to a remote Cisco PE in another AS. One customer VRF (RED) is used to prove end-to-end L3VPN forwarding and to document the label stacks on the wire. The configuration was verified on VOS 23.1.2 (builds 23.1.2-J and 23.1.2-B, commit d43d52f), configured from the CLI without Director, against Cisco Catalyst 8000V IOS XE 17.16.01a, on 26–27 Oct 2026. EVPN is out of scope.

## Reference Topology

Three ASes: AS 65001 and AS 65002 are Cisco IOS XE with IS-IS + LDP inside each AS and iBGP-LU with next-hop-self; VOS is the only node in AS 65003.

```text
            AS 65001                            AS 65002
  +---------------------------+       +---------------------------+
  |  PE1 ----------- ASBR1    |       |  ASBR2 ----------- PE2    |
  |  1.1.1.1        2.2.2.2   |=======|  3.3.3.3         4.4.4.4  |
  |  IS-IS + LDP              |eBGP-LU|  IS-IS + LDP              |
  |  iBGP-LU, next-hop-self   |       |  iBGP-LU, next-hop-self   |
  +-----+---------------------+       +-----+-----------------+---+
        | Gi3                               | Gi1             | Gi3
        |                                   | 10.2.35.1/30    |
     Host A                                 |               Host B
     172.16.30.10/24                        | eBGP-LU       172.16.40.10/24
     gw 172.16.30.1 (VRF VLAN30)            | family mpls   gw 172.16.40.1 (VRF VLAN40)
                                            | 0x8847, no GRE
                                            |
                                            | vni-0/1 10.2.35.2/30 (family mpls)
                              +-------------+-------------+
                              |  VOS 23.1.2 (PE)  AS 65003|
                              |  lo0 5.5.5.5              |
                              |  no IGP, no LDP           |
                              +-------------+-------------+
                                            | vni-0/3 192.168.107.1/24
                                            |
                                         Host C
                                         192.168.107.10/24
                                         gw 192.168.107.1 (VRF RED)

Service session:  PE1 lo0 1.1.1.1 <---- multihop eBGP VPNv4 (lo0 to lo0) ----> VOS lo0 5.5.5.5
Label path:       PE1 learns 5.5.5.5/32 via iBGP-LU from ASBR1, ASBR1 via eBGP-LU from ASBR2, ASBR2 via eBGP-LU from VOS
VRF RED loopbacks: PE1 Lo101 192.168.101.1/32, PE2 Lo101 192.168.101.2/32
```

**Host connections**

| Host | Attached to | IP address | Gateway | VRF (RD) | Route target |
| --- | --- | --- | --- | --- | --- |
| Host A | PE1 Gi3 (untagged) | 172.16.30.10/24 | 172.16.30.1 (PE1 Gi3) | VLAN30 (1.1.1.1:30) | export 65000:30, import 65000:30 65000:40 65000:110 |
| Host B | PE2 Gi3 (untagged) | 172.16.40.10/24 | 172.16.40.1 (PE2 Gi3) | VLAN40 (4.4.4.4:40) | export 65000:40, import 65000:30 65000:40 65000:110 |
| Host C | VOS vni-0/3.0 | 192.168.107.10/24 | 192.168.107.1 (VOS) | RED (5.5.5.5:1) | 65000:1 (plus 30/40/110 with the optional RT leaking in 4d) |
| PE1 RED loopback | Loopback101 | 192.168.101.1/32 | — | RED (1.1.1.1:1) | 65000:1 |
| PE2 RED loopback | Loopback101 | 192.168.101.2/32 | — | RED (4.4.4.4:1) | 65000:1 |

Hosts A and B reach Host C only after the optional RT leaking in section 4d; the base configuration connects Host C with the PE RED loopbacks.

| Node | AS | Loopback | Role | Link to next hop |
| --- | --- | --- | --- | --- |
| PE1 | 65001 | 1.1.1.1 | PE, VRF RED (RD 1.1.1.1:1, RT 65000:1), VPNv4 peer of VOS | IS-IS + LDP + iBGP-LU to ASBR1 |
| ASBR1 | 65001 | 2.2.2.2 | ASBR | 10.0.23.0/30 eBGP-LU to ASBR2, no LDP |
| ASBR2 | 65002 | 3.3.3.3 | ASBR, BGP-LU peer of VOS | Gi1 10.2.35.1/30 to VOS vni-0/1 10.2.35.2/30 |
| PE2 | 65002 | 4.4.4.4 | PE, VRF RED 192.168.101.2/32 (reached via PE1) | IS-IS + LDP + iBGP-LU to ASBR2 |
| VOS | 65003 | 5.5.5.5 | PE, VRF RED (RD 5.5.5.5:1, RT 65000:1) | vni-0/1.0 10.2.35.2/30 eBGP-LU to ASBR2 |

PE2's RED loopback is reached through PE1; VOS has no VPNv4 session to PE2.

## Prerequisites

VOS 23.1.x is required; the BGP-LU and MPLS-on-Ethernet knobs used here are not available on 22.1.4 \[Inference from CLI checks on 22.1.4\].

| Side | Requirement |
| --- | --- |
| VOS | Release 23.1.x (verified 23.1.2); standalone CLI configuration; no additional license needed for this setup |
| VOS | One routed interface towards the ASBR with `family inet` and `family mpls` |
| Cisco ASBR | `mpls bgp forwarding` on the interface facing VOS; `send-label` on the eBGP IPv4 session to VOS |
| Cisco PE | Multihop eBGP VPNv4 to the VOS loopback (`ebgp-multihop`, `update-source Loopback0`, `send-community extended`) |
| Cisco core | A labeled path to 5.5.5.5/32 end to end: eBGP-LU between ASBRs, iBGP-LU with next-hop-self inside each AS |
| Both | Matching route targets on the VRF (65000:1 in this lab) |

## Configuration

### 4a. ASBR2 (IOS XE)

ASBR2 runs eBGP-LU to VOS on a plain Ethernet interface; `mpls bgp forwarding` enables MPLS on that interface without LDP.

```text
interface GigabitEthernet1
 description to VOS vni-0/1
 ip address 10.2.35.1 255.255.255.252
 mpls bgp forwarding
!
router bgp 65002
 bgp router-id 3.3.3.3
 no bgp default ipv4-unicast
 neighbor 4.4.4.4 remote-as 65002
 neighbor 4.4.4.4 update-source Loopback0
 neighbor 10.0.23.1 remote-as 65001
 neighbor 10.2.35.2 remote-as 65003
 address-family ipv4
  network 4.4.4.4 mask 255.255.255.255
  neighbor 4.4.4.4 activate
  neighbor 4.4.4.4 next-hop-self
  neighbor 4.4.4.4 send-label
  neighbor 10.0.23.1 activate
  neighbor 10.0.23.1 send-label
  neighbor 10.2.35.2 activate
  neighbor 10.2.35.2 send-label
 exit-address-family
```

### 4b. PE1 (IOS XE)

PE1 peers VPNv4 directly with the VOS loopback over multihop eBGP; the next hop 5.5.5.5 is resolved through the iBGP-LU route learned from ASBR1.

```text
router bgp 65001
 bgp router-id 1.1.1.1
 no bgp default ipv4-unicast
 neighbor 2.2.2.2 remote-as 65001
 neighbor 2.2.2.2 update-source Loopback0
 neighbor 5.5.5.5 remote-as 65003
 neighbor 5.5.5.5 ebgp-multihop 255
 neighbor 5.5.5.5 update-source Loopback0
 address-family ipv4
  neighbor 2.2.2.2 activate
  neighbor 2.2.2.2 send-label
 exit-address-family
 address-family vpnv4
  neighbor 5.5.5.5 activate
  neighbor 5.5.5.5 send-community extended
 exit-address-family
 address-family ipv4 vrf RED
  redistribute connected
 exit-address-family
!
vrf definition RED
 rd 1.1.1.1:1
 address-family ipv4
  route-target export 65000:1
  route-target import 65000:1
```

### 4c. VOS (set format)

The complete L3VPN-relevant VOS configuration is below; this is the verified set.

```text
set interfaces lo0 unit 0 family inet address 5.5.5.5
set interfaces vni-0/0 enable true
set interfaces vni-0/0 unit 100 vlan-id 100
set interfaces vni-0/0 unit 100 family inet address 192.168.105.1/24
set interfaces vni-0/1 enable true
set interfaces vni-0/1 unit 0 family inet address 10.2.35.2/30
set interfaces vni-0/1 unit 0 family mpls
set interfaces vni-0/3 enable true
set interfaces vni-0/3 unit 0 family inet address 192.168.107.1/24
set routing-instances CORE instance-type virtual-router
set routing-instances CORE policy-options redistribution-policy TO-LU term LO match protocol direct
set routing-instances CORE policy-options redistribution-policy TO-LU term LO action accept
set routing-instances CORE policy-options redistribute-to-bgp-labeled-unicast TO-LU
set routing-instances CORE mpls-vpn-core
set routing-instances CORE interfaces [ lo0.0 vni-0/1.0 ]
set routing-instances CORE routing-options mpls-vpn-local-router-address 5.5.5.5
set routing-instances CORE protocols bgp 1 router-id 5.5.5.5
set routing-instances CORE protocols bgp 1 local-as as-number 65003
set routing-instances CORE protocols bgp 1 group LU type external
set routing-instances CORE protocols bgp 1 group LU family inet labeled-unicast
set routing-instances CORE protocols bgp 1 group LU peer-as 65002
set routing-instances CORE protocols bgp 1 group LU neighbor 10.2.35.1
set routing-instances CORE protocols bgp 1 group VPN type external
set routing-instances CORE protocols bgp 1 group VPN family inet-vpn unicast
set routing-instances CORE protocols bgp 1 group VPN peer-as 65001
set routing-instances CORE protocols bgp 1 group VPN local-address 5.5.5.5
set routing-instances CORE protocols bgp 1 group VPN neighbor 1.1.1.1
set routing-instances RED instance-type vrf
set routing-instances RED policy-options redistribution-policy DIRECT term T1 match protocol direct
set routing-instances RED policy-options redistribution-policy DIRECT term T1 action accept
set routing-instances RED policy-options redistribute-to-bgp DIRECT
set routing-instances RED mpls-vpn-core-instance CORE
set routing-instances RED interfaces [ vni-0/0.100 vni-0/3.0 ]
set routing-instances RED route-distinguisher 5.5.5.5:1
set routing-instances RED vrf-both-target target:65000:1
set orgs org LAB appliance-owner
set orgs org LAB available-routing-instances [ CORE RED ]
set orgs org LAB owned-routing-instances [ CORE RED ]
set orgs org LAB traffic-identification using [ lo0.0 vni-0/0.100 vni-0/1.0 vni-0/3.0 ]
set orgs org-services LAB objects zones MPLS-ZONE interface-list [ lo0.0 vni-0/0.100 vni-0/1.0 vni-0/3.0 ]
```

What each block does:

- `interfaces`: `family mpls` on vni-0/1 unit 0 enables MPLS on the Ethernet link to ASBR2. The loopback address is entered without a prefix length.
- `CORE`: a `virtual-router` that carries both the labeled-unicast session (group LU, `family inet labeled-unicast`, peer-as 65002) and the VPNv4 session (group VPN, `family inet-vpn unicast`, peer-as 65001, sourced from 5.5.5.5).
- `mpls-vpn-core` + `mpls-vpn-local-router-address 5.5.5.5`: make CORE the MPLS VPN core instance and 5.5.5.5 the PE address used as the VPNv4 next hop.
- `redistribute-to-bgp-labeled-unicast TO-LU`: advertises the directly connected loopback 5.5.5.5/32 to ASBR2 with a label. VOS advertises implicit-null for its own loopback.
- `RED`: a `vrf` instance bound to the core via `mpls-vpn-core-instance CORE`, with RD 5.5.5.5:1 and RT 65000:1 in both directions. `redistribute-to-bgp DIRECT` exports the connected RED subnets into VPNv4.
- `orgs`: the org must own every routing-instance, and every interface must appear in `traffic-identification` and in a zone. Otherwise the commit is refused with `routing-instance CORE must be defined as a owned-routing-instance for an org`.

### 4d. Optional: route-target leaking to other Cisco VRFs

To reach additional Cisco VRFs (for example RT 65000:30, 65000:40 and 65000:110), replace `vrf-both-target` with explicit import and export lists.

```text
delete routing-instances RED vrf-both-target
set routing-instances RED vrf-import-target "target:65000:1 target:65000:30 target:65000:40 target:65000:110"
set routing-instances RED vrf-export-target "target:65000:1 target:65000:30 target:65000:40 target:65000:110"
```

The value is one string of space-separated targets. The `[ a b ]` list form is accepted by the parser but results in no change. With this configuration PE1 shows the VOS prefix with `Extended Community: RT:65000:1 RT:65000:30 RT:65000:40 RT:65000:110`, and one VPN label per destination VRF is used on the wire.

## Verification

### 5a. BGP sessions and labels

Both VOS sessions are established: VPNv4 to PE1 (17 prefixes received) and labeled unicast to ASBR2 (2 prefixes received). Outputs are verbatim, trimmed.

VOS:

```text
admin@versa-cli> show bgp neighbor brief
routing-instance: CORE
Neighbor        V  MsgRcvd   MsgSent    Uptime     State/PfxRcd  PfxSent AS
1.1.1.1         4  103       93         00:36:06   17            4       65001
10.2.35.1       4  85        85         00:36:06   2             2       65002

admin@versa-cli> show route routing-instance CORE
BGP    N/A  +1.1.1.1/32          0.0.0.0         00:36:37 Indirect
BGP    N/A  +4.4.4.4/32          0.0.0.0         00:36:37 Indirect
local  N/A  +5.5.5.5/32          0.0.0.0         00:36:47 directly connected
conn   N/A  +10.2.35.0/30        0.0.0.0         00:36:41 vni-0/1.0
```

ASBR2 holds a BGP-LU label for 5.5.5.5/32 with out-label `imp-null`, so it pops its own label towards VOS:

```text
ASBR2#show bgp ipv4 unicast labels
   Network          Next Hop      In label/Out label
   1.1.1.1/32       10.0.23.1       3002/2000
   5.5.5.5/32       10.2.35.2       3004/imp-null
ASBR2#show mpls forwarding-table 5.5.5.5 32
Local      Outgoing   Prefix           Bytes Label   Outgoing   Next Hop
3004       Pop Label  5.5.5.5/32       20089         Gi1        10.2.35.2
```

PE1 learns 5.5.5.5/32 with a label from ASBR1 over iBGP-LU, which lets it resolve the VPNv4 next hop:

```text
PE1#show bgp ipv4 unicast 5.5.5.5
  65002 65003
    2.2.2.2 (metric 20) from 2.2.2.2 (2.2.2.2)
      mpls labels in/out nolabel/2003
```

### 5b. VRF RED routes

VOS imports both Cisco RED loopbacks with next hop 1.1.1.1, and PE1 imports the VOS RED subnet with next hop 5.5.5.5 and VPN label 24706.

```text
admin@versa-cli> show route routing-instance RED
BGP    N/A  +192.168.101.1/32    1.1.1.1         00:01:05 Indirect
BGP    N/A  +192.168.101.2/32    1.1.1.1         00:01:05 Indirect
conn   N/A  +192.168.105.0/24    0.0.0.0         00:01:09 vni-0/0.100
conn   N/A  +192.168.107.0/24    0.0.0.0         00:01:04 vni-0/3.0
```

```text
PE1#show bgp vpnv4 unicast vrf RED 192.168.107.0
  65003, imported path from 5.5.5.5:1:192.168.107.0/24 (global)
    5.5.5.5 (metric 20) (via default) from 5.5.5.5 (5.5.5.5)
      Origin incomplete, localpref 100, valid, external, best
      Extended Community: RT:65000:1
      mpls labels in/out nolabel/24706
PE1#show ip route vrf RED bgp
B     192.168.107.0/24 [20/0] via 5.5.5.5, 00:10:26
```

### 5c. Data plane

All pings between the host behind VOS and the Cisco RED loopbacks succeed in both directions.

| Source | Destination | Result |
| --- | --- | --- |
| Host 192.168.107.10 (behind VOS) | PE1 RED 192.168.101.1 | 4/4 |
| Host 192.168.107.10 (behind VOS) | PE2 RED 192.168.101.2 | 4/4 |
| PE1 `ping vrf RED 192.168.107.10 source 192.168.101.1` | Host 192.168.107.10 | 5/5 |

Label stacks captured on the VOS-ASBR2 link (EtherType 0x8847, plain Ethernet, no GRE):

| Direction | Label stack on the wire | Meaning |
| --- | --- | --- |
| VOS to PE1 RED | \[3002 \| 1005\] | 3002 = ASBR2's BGP-LU label for 1.1.1.1/32; 1005 = PE1's VPN label for RED |
| PE1 RED to VOS | \[24706\] | VOS's VPN label for RED only; ASBR2 popped its transport label 3004 because VOS advertised implicit-null for 5.5.5.5/32 |

Label values differ per lab (a first run showed \[3002 | 1001\] out and \[24704\] in); the stack structure is identical.

The PE1 traceroute and `show ip cef vrf RED 192.168.107.10 detail` in section 5e are the data-plane proof of the Option C stack: per-hop labels 2003/24706 then 3004/24706, and the CEF recursion 5.5.5.5 label 24706 via 2.2.2.2 label 2003.

### 5d. How to read the label stack

Outbound from VOS, the top label is the BGP-LU transport label that ASBR2 allocated for the remote PE loopback, and the bottom label is the VPN label the remote PE advertised in VPNv4. Inbound, only the VPN label that VOS advertised arrives, because VOS signals implicit-null for its own loopback and ASBR2 pops the transport label (penultimate hop popping).

### 5e. Full command outputs

Raw console outputs from the verified lab, one block per device; only pagination commands, timestamps and the management address were removed.

VOS:

```text
admin@versa-cli> show bgp neighbor brief | nomore
routing-instance: CORE

Neighbor        V  MsgRcvd   MsgSent    Uptime     State/PfxRcd  PfxSent AS
1.1.1.1         4  137       129        00:51:37   17            4       65001
10.2.35.1       4  119       120        00:51:37   2             2       65002

admin@versa-cli> show interfaces brief | nomore
NAME         MAC                OPER  ADMIN  TENANT  VRF     IP
-----------------------------------------------------------------------------------------
eth-0/0      52:54:00:cd:63:8a  up    up     0       global  <mgmt-ip>
lo0          n/a                up    up     -       -
lo0.0        n/a                up    up     1       CORE    5.5.5.5/32
vni-0/0      52:54:00:44:f7:82  up    up     -       -
vni-0/0.100  52:54:00:44:f7:82  up    up     1       RED     192.168.105.1/24
vni-0/1      52:54:00:9a:92:65  up    up     -       -
vni-0/1.0    52:54:00:9a:92:65  up    up     1       CORE    10.2.35.2/30
vni-0/2      52:54:00:10:39:4e  up    up     -       -
vni-0/2.106  52:54:00:10:39:4e  up    up     1       EVI106
vni-0/3      52:54:00:61:1a:6b  up    up     -       -
vni-0/3.0    52:54:00:61:1a:6b  up    up     1       RED     192.168.107.1/24
vni-0/4      52:54:00:59:e6:e7  down  down   -       -
vni-0/5      52:54:00:b1:ec:81  down  down   -       -
vni-0/6      52:54:00:db:9a:e5  down  down   -       -

admin@versa-cli> show route routing-instance CORE | nomore

Routes for Routing instance : CORE  AFI: ipv4  SAFI: unicast

Prot   Type  Dest Address/Mask   Next-hop        Age      Interface name     Nexthop name
----   ----  -----------------   --------        ---      --------------    ---------------
BGP    N/A  +1.1.1.1/32          0.0.0.0         00:51:57 Indirect
BGP    N/A  +4.4.4.4/32          0.0.0.0         00:51:57 Indirect
local  N/A  +5.5.5.5/32          0.0.0.0         00:52:07 directly connected
conn   N/A  +10.2.35.0/30        0.0.0.0         00:52:01 vni-0/1.0
local  N/A  +10.2.35.2/32        0.0.0.0         00:52:01 directly connected


Routes for Routing instance : CORE  AFI: ipv6  SAFI: unicast

admin@versa-cli> show route routing-instance RED | nomore

Routes for Routing instance : RED  AFI: ipv4  SAFI: unicast

Prot   Type  Dest Address/Mask   Next-hop        Age      Interface name     Nexthop name
----   ----  -----------------   --------        ---      --------------    ---------------
BGP    N/A  +172.16.30.0/24      1.1.1.1         00:41:06 Indirect
BGP    N/A  +172.16.40.0/24      1.1.1.1         00:41:06 Indirect
BGP    N/A  +172.16.100.0/24     1.1.1.1         00:41:06 Indirect
BGP    N/A  +192.168.101.1/32    1.1.1.1         00:41:05 Indirect
BGP    N/A  +192.168.101.2/32    1.1.1.1         00:41:05 Indirect
conn   N/A  +192.168.105.0/24    0.0.0.0         00:04:14 vni-0/0.100
local  N/A  +192.168.105.1/32    0.0.0.0         00:04:14 directly connected
conn   N/A  +192.168.107.0/24    0.0.0.0         00:41:04 vni-0/3.0
local  N/A  +192.168.107.1/32    0.0.0.0         00:41:04 directly connected


Routes for Routing instance : RED  AFI: ipv6  SAFI: unicast

admin@versa-cli> show route table l3vpn.ipv4.unicast routing-instance CORE receive-protocol bgp 192.168.101.1/32 exact | nomore

Routes for Routing instance : CORE  AFI: ipv4  SAFI: unicast

Routing entry for 192.168.101.1/32
   Peer Address       : 1.1.1.1
   Route Distinguisher: 1.1.1.1:1
   Next-hop           : 1.1.1.1
   VPN Label          : 1005
   Local Preference   : 100
   AS Path            : 65001
   AS Path(. notation): 65001
   Origin             : Incomplete
   MED                : 0
   Community          : N/A
   Extended community : target:65000:1
   Scalable group tag : 0
   Preference         : Default
   Weight             : 0
   Best               : true

admin@versa-cli> show route table l3vpn.ipv4.unicast routing-instance CORE advertising-protocol bgp 192.168.107.0/24 exact | nomore

Routes for Routing instance : CORE  AFI: ipv4  SAFI: unicast

Routing entry for 192.168.107.0/24
   Peer Address       : 1.1.1.1
   Route Distinguisher: 5.5.5.5:1
   Next-hop           : 5.5.5.5
   VPN Label          : 24706
   Local Preference   : 0
   AS Path            : 65003
   AS Path(. notation): 65003
   Origin             : Incomplete
   MED                : 0
   Community          : N/A
   Extended community : target:65000:1 target:65000:30 target:65000:40 target:65000:110

admin@versa-cli> ping 10.2.35.1 routing-instance CORE count 3
PING 10.2.35.1 (10.2.35.1) 56(84) bytes of data.
64 bytes from 10.2.35.1: icmp_seq=1 ttl=255 time=1.94 ms
64 bytes from 10.2.35.1: icmp_seq=2 ttl=255 time=1.67 ms
64 bytes from 10.2.35.1: icmp_seq=3 ttl=255 time=1.98 ms

--- 10.2.35.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2002ms
rtt min/avg/max/mdev = 1.672/1.865/1.983/0.146 ms
```

ASBR2:

```text
ASBR2#show bgp ipv4 unicast summary | begin Neighbor
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
4.4.4.4         4        65002      96     102       13    0    0 01:26:11        0
10.0.23.1       4        65001      97     100       13    0    0 01:26:18        1
10.2.35.2       4        65003     117     117       13    0    0 00:50:27        2
ASBR2#show bgp ipv4 unicast labels
   Network          Next Hop      In label/Out label
   1.1.1.1/32       10.0.23.1       3002/2000
   4.4.4.4/32       10.2.34.2       3001/nolabel
   5.5.5.5/32       10.2.35.2       3004/imp-null
   10.2.35.0/30     10.2.35.2       3005/imp-null
ASBR2#show mpls forwarding-table
Local      Outgoing   Prefix           Bytes Label   Outgoing   Next Hop
Label      Label      or Tunnel Id     Switched      interface
3000       Pop Label  10.0.23.1/32     0             Gi3        10.0.23.1
3001       Pop Label  4.4.4.4/32       283730        Gi2        10.2.34.2
3002       2000       1.1.1.1/32       327527        Gi3        10.0.23.1
3003       Pop Label  10.2.35.2/32     0             Gi1        10.2.35.2
3004       Pop Label  5.5.5.5/32       26304         Gi1        10.2.35.2
3005       Pop Label  10.2.35.0/30     0             aggregate
ASBR2#show ip route 5.5.5.5
Routing entry for 5.5.5.5/32
  Known via "bgp 65002", distance 20, metric 0
  Tag 65003, type external
  Last update from 10.2.35.2 00:50:57 ago
  Routing Descriptor Blocks:
  * 10.2.35.2, from 10.2.35.2, 00:50:57 ago
      opaque_ptr 0x76B9A55ED9A8
      Route metric is 0, traffic share count is 1
      AS Hops 1
      Route tag 65003
      MPLS label: implicit-null
ASBR2#ping 5.5.5.5 source 3.3.3.3 repeat 3
Type escape sequence to abort.
Sending 3, 100-byte ICMP Echos to 5.5.5.5, timeout is 2 seconds:
Packet sent with a source address of 3.3.3.3
...
Success rate is 0 percent (0/3)
```

PE1:

```text
PE1#show bgp ipv4 unicast | begin Network
     Network          Next Hop            Metric LocPrf Weight Path
 r>i  1.1.1.1/32       2.2.2.2                 20    100      0 i
 *>i  4.4.4.4/32       2.2.2.2                 20    100      0 65002 i
 *>i  5.5.5.5/32       2.2.2.2                  0    100      0 65002 65003 ?
 *>i  10.2.35.0/30     2.2.2.2                  0    100      0 65002 65003 ?
PE1#show bgp vpnv4 unicast vrf RED | begin Network
     Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 1.1.1.1:1 (default for vrf RED)
 *>   192.168.101.1/32 0.0.0.0                  0         32768 ?
 *>   192.168.101.2/32 4.4.4.4                  0             0 65002 ?
 *>   192.168.105.0    5.5.5.5                                0 65003 ?
 *>   192.168.107.0    5.5.5.5                                0 65003 ?
PE1#show ip route vrf RED | begin Gateway
Gateway of last resort is not set
      192.168.101.0/32 is subnetted, 2 subnets
C        192.168.101.1 is directly connected, Loopback101
B        192.168.101.2 [20/0] via 4.4.4.4, 01:23:50
B     192.168.105.0/24 [20/0] via 5.5.5.5, 00:01:07
B     192.168.107.0/24 [20/0] via 5.5.5.5, 00:38:10
PE1#show ip cef vrf RED 192.168.107.10 detail
192.168.107.0/24, epoch 0, flags [rib defined all labels]
  recursive via 5.5.5.5 label 24706
    recursive via 2.2.2.2 label 2003
      nexthop 10.1.12.2 GigabitEthernet2
PE1#ping vrf RED 192.168.107.10 source 192.168.101.1 repeat 5
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.107.10, timeout is 2 seconds:
Packet sent with a source address of 192.168.101.1
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 4/5/9 ms
PE1#traceroute vrf RED 192.168.107.10 source 192.168.101.1 numeric timeout 1 probe 1
Type escape sequence to abort.
Tracing the route to 192.168.107.10
VRF info: (vrf in name/id, vrf out name/id)
  1 10.1.12.2 [MPLS: Labels 2003/24706 Exp 0] 6 msec
  2 10.0.23.2 [MPLS: Labels 3004/24706 Exp 0] 5 msec
  3  *
  4 192.168.107.10 5 msec
```

Host C:

```text
host-c:~$ ip -4 addr show eth0 | grep inet; ip route
    inet 192.168.107.10/24 scope global eth0
default via 192.168.107.1 dev eth0
192.168.107.0/24 dev eth0 scope link  src 192.168.107.10
host-c:~$ ping -c 4 192.168.101.1
PING 192.168.101.1 (192.168.101.1): 56 data bytes
64 bytes from 192.168.101.1: seq=0 ttl=42 time=3.282 ms
64 bytes from 192.168.101.1: seq=1 ttl=42 time=3.004 ms
64 bytes from 192.168.101.1: seq=2 ttl=42 time=3.191 ms
64 bytes from 192.168.101.1: seq=3 ttl=42 time=3.113 ms
--- 192.168.101.1 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 3.004/3.147/3.282 ms
host-c:~$ ping -c 4 192.168.101.2
PING 192.168.101.2 (192.168.101.2): 56 data bytes
64 bytes from 192.168.101.2: seq=0 ttl=42 time=5.277 ms
64 bytes from 192.168.101.2: seq=1 ttl=42 time=4.779 ms
64 bytes from 192.168.101.2: seq=2 ttl=42 time=4.555 ms
64 bytes from 192.168.101.2: seq=3 ttl=42 time=4.494 ms
--- 192.168.101.2 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 4.494/4.776/5.277 ms
host-c:~$ traceroute -n -w 1 -q 1 192.168.101.1
traceroute to 192.168.101.1 (192.168.101.1), 30 hops max, 46 byte packets
 1  192.168.107.1  0.923 ms
 2  10.2.35.1  7.075 ms
 3  10.0.23.1  4.568 ms
 4  192.168.101.1  71.877 ms
```

### 5f. Host-to-host tests and packet captures

Host C (behind VOS) and Host A / Host B (behind the Cisco PEs) reach each other with 0% loss in both directions, and the captures on the VOS–ASBR2 and ASBR1–ASBR2 links show the expected two-label stacks on the wire. These tests use the RT leaking from section 4d (RED imports 65000:30 and 65000:40).

**Host C → Host A and Host B**

```text
host-c:~$ ip -4 addr show eth0 | grep inet
    inet 192.168.107.10/24 scope global eth0
host-c:~$ ping -c 4 172.16.30.10
PING 172.16.30.10 (172.16.30.10): 56 data bytes
64 bytes from 172.16.30.10: seq=0 ttl=42 time=4.922 ms
64 bytes from 172.16.30.10: seq=1 ttl=42 time=3.790 ms
64 bytes from 172.16.30.10: seq=2 ttl=42 time=3.511 ms
64 bytes from 172.16.30.10: seq=3 ttl=42 time=3.897 ms
--- 172.16.30.10 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
host-c:~$ ping -c 4 172.16.40.10
PING 172.16.40.10 (172.16.40.10): 56 data bytes
64 bytes from 172.16.40.10: seq=0 ttl=42 time=5.997 ms
64 bytes from 172.16.40.10: seq=1 ttl=42 time=5.771 ms
64 bytes from 172.16.40.10: seq=2 ttl=42 time=5.181 ms
64 bytes from 172.16.40.10: seq=3 ttl=42 time=5.479 ms
--- 172.16.40.10 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
host-c:~$ traceroute -n -w 1 -q 1 172.16.30.10
traceroute to 172.16.30.10 (172.16.30.10), 30 hops max, 46 byte packets
 1  192.168.107.1  0.972 ms      (VOS vni-0/3, VRF RED)
 2  10.2.35.1  6.192 ms          (ASBR2 Gi1)
 3  10.0.23.1  5.768 ms          (ASBR1)
 4  172.16.30.1  8.026 ms        (PE1 Gi3, VRF VLAN30)
 5  172.16.30.10  3.005 ms       (Host A)
host-c:~$ traceroute -n -w 1 -q 1 172.16.40.10
traceroute to 172.16.40.10 (172.16.40.10), 30 hops max, 46 byte packets
 1  192.168.107.1  0.685 ms      (VOS)
 2  10.2.35.1  6.576 ms          (ASBR2)
 3  10.0.23.1  6.924 ms          (ASBR1)
 4  10.1.12.1  6.970 ms          (PE1: re-exports the 65000:40 route into VLAN30)
 5  10.1.12.2  6.486 ms          (ASBR1)
 6  10.0.23.2  6.388 ms          (ASBR2)
 7  172.16.40.1  21.286 ms       (PE2 Gi3, VRF VLAN40)
 8  172.16.40.10  5.489 ms       (Host B)
```

The Host B path hairpins through PE1 because VOS has a VPNv4 session only with PE1: PE1 imports 172.16.40.0/24 (RT 65000:40) into VRF VLAN30 and re-exports it with RT 65000:30 and itself as next hop, which is the copy VOS installs. A VPNv4 session to PE2 (or a route reflector) would give the direct path.

**Host A → Host C and Host B → Host C**

```text
host-a:~$ ip -4 addr show eth0 | grep inet
    inet 172.16.30.10/24 scope global eth0
host-a:~$ ip route
default via 172.16.30.1 dev eth0
172.16.30.0/24 dev eth0 scope link  src 172.16.30.10
host-a:~$ ping -c 4 192.168.107.10
PING 192.168.107.10 (192.168.107.10): 56 data bytes
64 bytes from 192.168.107.10: seq=0 ttl=42 time=3.185 ms
64 bytes from 192.168.107.10: seq=1 ttl=42 time=3.763 ms
64 bytes from 192.168.107.10: seq=2 ttl=42 time=3.404 ms
64 bytes from 192.168.107.10: seq=3 ttl=42 time=3.378 ms
--- 192.168.107.10 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
host-a:~$ traceroute -n -w 1 -q 1 192.168.107.10
traceroute to 192.168.107.10 (192.168.107.10), 30 hops max, 46 byte packets
 1  172.16.30.1  0.908 ms        (PE1 Gi3)
 2  10.1.12.2  5.047 ms          (ASBR1)
 3  10.0.23.2  6.122 ms          (ASBR2)
 4  *                            (VOS: label-switched, no ICMP time-exceeded)
 5  192.168.107.10  3.474 ms     (Host C)

host-b:~$ ip -4 addr show eth0 | grep inet
    inet 172.16.40.10/24 scope global eth0
host-b:~$ ping -c 4 192.168.107.10
PING 192.168.107.10 (192.168.107.10): 56 data bytes
64 bytes from 192.168.107.10: seq=0 ttl=42 time=5.377 ms
64 bytes from 192.168.107.10: seq=1 ttl=42 time=5.065 ms
64 bytes from 192.168.107.10: seq=2 ttl=42 time=5.245 ms
64 bytes from 192.168.107.10: seq=3 ttl=42 time=5.295 ms
--- 192.168.107.10 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
host-b:~$ traceroute -n -w 1 -q 1 192.168.107.10
traceroute to 192.168.107.10 (192.168.107.10), 30 hops max, 46 byte packets
 1  172.16.40.1  0.817 ms        (PE2 Gi3)
 2  10.2.34.1  63.870 ms         (ASBR2)
 3  10.0.23.1  6.719 ms          (ASBR1)
 4  10.1.12.1  6.883 ms          (PE1, hairpin)
 5  10.1.12.2  13.568 ms         (ASBR1)
 6  10.0.23.2  8.785 ms          (ASBR2)
 7  *                            (VOS)
 8  192.168.107.10  4.717 ms     (Host C)
```

**Capture on the VOS–ASBR2 link (VOS vni-0/1 ↔ ASBR2 Gi1)** while Host C pinged Host A, Host B and PE1 Lo101. Every frame is plain Ethernet 0x8847; no GRE or IP tunnel header.

| Direction | Flow | Label stack on the wire | Meaning |
| --- | --- | --- | --- |
| VOS → ASBR2 | 192.168.107.10 → 172.16.30.10 (Host A) | \[3002 \| 1007\] | 3002 = ASBR2's BGP-LU label for 1.1.1.1; 1007 = PE1's VPN label for VRF VLAN30 |
| VOS → ASBR2 | 192.168.107.10 → 172.16.40.10 (Host B) | \[3002 \| 1012\] | 1012 = PE1's VPN label for 172.16.40.0/24 imported into VLAN30 |
| VOS → ASBR2 | 192.168.107.10 → 192.168.101.1 (PE1 Lo101) | \[3002 \| 1005\] | 1005 = PE1's VPN label for VRF RED |
| ASBR2 → VOS | any → 192.168.107.10 | \[24706\] | 24706 = VOS's VPN label for RED; the LU label is already popped by ASBR2 (PHP, imp-null) |

```text
$ tshark -r vos_asbr2_hosts.pcap -Y icmp -T fields -e frame.number -e eth.type -e mpls.label -e mpls.bottom -e ip.src -e ip.dst -e icmp.type
8   0x8847  3002,1007  0,1  192.168.107.10  172.16.30.10    8
9   0x8847  24706      1    172.16.30.10    192.168.107.10  0
15  0x8847  3002,1012  0,1  192.168.107.10  172.16.40.10    8
16  0x8847  24706      1    172.16.40.10    192.168.107.10  0
23  0x8847  3002,1005  0,1  192.168.107.10  192.168.101.1   8
24  0x8847  24706      1    192.168.101.1   192.168.107.10  0

$ tshark -r vos_asbr2_hosts.pcap -Y "frame.number==8 || frame.number==9" -V
Frame 8: 106 bytes on wire
Ethernet II, Src: 52:54:00:9a:92:65 (VOS vni-0/1), Dst: 52:54:00:7a:3e:aa (ASBR2 Gi1)
    Type: MPLS label switched packet (0x8847)
MultiProtocol Label Switching Header, Label: 3002, Exp: 0, S: 0, TTL: 63
MultiProtocol Label Switching Header, Label: 1007, Exp: 0, S: 1, TTL: 63
Internet Protocol Version 4, Src: 192.168.107.10, Dst: 172.16.30.10
Internet Control Message Protocol
    Type: Echo (ping) request (8)
Frame 9: 102 bytes on wire
Ethernet II, Src: 52:54:00:7a:3e:aa (ASBR2 Gi1), Dst: 52:54:00:9a:92:65 (VOS vni-0/1)
    Type: MPLS label switched packet (0x8847)
MultiProtocol Label Switching Header, Label: 24706, Exp: 0, S: 1, TTL: 61
Internet Protocol Version 4, Src: 172.16.30.10, Dst: 192.168.107.10
Internet Control Message Protocol
    Type: Echo (ping) reply (0)
```

**Capture on the ASBR1–ASBR2 link** for the same pings. The Host B flow crosses this link twice in each direction, which is the PE1 hairpin seen in the traceroute.

| Direction | Flow | Label stack | Meaning |
| --- | --- | --- | --- |
| ASBR2 → ASBR1 | Host C → Host A | \[2000 \| 1007\] | 2000 = ASBR1's BGP-LU label for 1.1.1.1 (ASBR2 swapped 3002 → 2000) |
| ASBR1 → ASBR2 | Host A → Host C | \[3004 \| 24706\] | 3004 = ASBR2's BGP-LU label for 5.5.5.5 |
| ASBR2 → ASBR1 | Host C → Host B, first pass | \[2000 \| 1012\] | toward PE1 |
| ASBR1 → ASBR2 | Host C → Host B, second pass | \[3001 \| 4005\] | 3001 = ASBR2's LU label for 4.4.4.4; 4005 = PE2's VPN label for VRF VLAN40 |
| ASBR2 → ASBR1 | Host B → Host C, first pass | \[2000 \| 1016\] | 1016 = PE1's VPN label for 192.168.107.0/24 re-exported into VLAN30 |
| ASBR1 → ASBR2 | Host B → Host C, second pass | \[3004 \| 24706\] | toward VOS |

```text
$ tshark -r asbr1_asbr2_hosts.pcap -Y icmp -T fields -e frame.number -e mpls.label -e ip.src -e ip.dst -e icmp.type
11  2000,1007   192.168.107.10  172.16.30.10    8
12  3004,24706  172.16.30.10    192.168.107.10  0
18  2000,1012   192.168.107.10  172.16.40.10    8
19  3001,4005   192.168.107.10  172.16.40.10    8
20  2000,1016   172.16.40.10    192.168.107.10  0
21  3004,24706  172.16.40.10    192.168.107.10  0
33  2000,1005   192.168.107.10  192.168.101.1   8
34  3004,24706  192.168.101.1   192.168.107.10  0
```

The transport labels match ASBR2's own tables in 5e: `show bgp ipv4 unicast labels` lists 1.1.1.1/32 in 3002 / out 2000, 4.4.4.4/32 in 3001, 5.5.5.5/32 in 3004 / imp-null, and `show mpls forwarding-table` shows 3002 → 2000 via Gi3 and 3004 → Pop Label via Gi1.

**VOS view of the two labels.** The advertised Host C prefix carries VPN label 24706, the label seen on every frame coming back from ASBR2; the Host B prefix is received from PE1 (next hop 1.1.1.1, label 1012, AS path 65001 65002), which is why Host C to Host B traffic goes to PE1 first.

```text
admin@versa-cli> show route table l3vpn.ipv4.unicast routing-instance CORE advertising-protocol bgp 192.168.107.0/24 exact | nomore

Routes for Routing instance : CORE  AFI: ipv4  SAFI: unicast

Routing entry for 192.168.107.0/24
   Peer Address       : 1.1.1.1
   Route Distinguisher: 5.5.5.5:1
   Next-hop           : 5.5.5.5
   VPN Label          : 24706
   Local Preference   : 0
   AS Path            : 65003
   Origin             : Incomplete
   MED                : 0
   Extended community : target:65000:1 target:65000:30 target:65000:40 target:65000:110

admin@versa-cli> show route table l3vpn.ipv4.unicast routing-instance CORE receive-protocol bgp 172.16.40.0/24 exact | nomore

Routes for Routing instance : CORE  AFI: ipv4  SAFI: unicast

Routing entry for 172.16.40.0/24
   Peer Address       : 1.1.1.1
   Route Distinguisher: 4.4.4.4:40
   Next-hop           : 1.1.1.1
   VPN Label          : 1012
   Local Preference   : 100
   AS Path            : 65001 65002
   Origin             : Incomplete
   MED                : 0
   Extended community : target:65000:40
   Best               : true
```

## Known limitations / caveats

Transit L3VPN traffic works; the open items below were observed in the same lab.

1. Packets from the MPLS side addressed to VOS's own VRF interface IP (for example PE1 `ping vrf RED 192.168.105.1` or 192.168.107.1) are dropped (0/5). VOS packet trace shows that the lookup after the label pop lands in the core VR's FIB (NO\_ROUTE). Transit traffic to hosts behind VOS is not affected. Workaround for testing: ping a host behind VOS, not the VOS interface.
2. `global-vrf-id` / `announce-global-vrf-id-label true` on the VRF had no effect on the advertised VPN label.
3. `family inet labeled-unicast` and MPLS on Ethernet are not available on VOS 22.1.4; this setup needs 23.1 or later \[Inference from CLI checks on 22.1.4\].
4. CLI traps: the loopback address takes no prefix length; `set <leaf-list> [ x ]` replaces the whole list, so rewrite org, zone and VRF interface lists in full; every routing-instance must be owned by an org; a vni cannot have `unit 0` next to an access bridge unit (error `Only one can be defined`).
5. VOS's loopback is not pingable from outside; use an interface or host address for reachability tests. Observed evidence: ASBR2 ping 5.5.5.5 source 3.3.3.3 gives 0/3, while BGP-LU on that same loopback is Established and labeled traffic to it is forwarded.
