# 🌐 FlexVPN Multi-Access Hub — One Config, Every VPN Type

!\[Cisco](https://img.shields.io/badge/Cisco-IOS--XE-1BA0D7?style=flat\&logo=cisco\&logoColor=white)
!\[FlexVPN](https://img.shields.io/badge/FlexVPN-IKEv2-blue?style=flat)
!\[ISE](https://img.shields.io/badge/Cisco-ISE%20RADIUS-orange?style=flat)
!\[Status](https://img.shields.io/badge/Status-Phased%20Build-yellow?style=flat)

A single **FlexVPN hub** (IOS-XE) that terminates **every major VPN type** into one enterprise core — dynamic multi-spoke (DMVPN-style), a site-to-site **ASA over VTI IPsec**, and **AnyConnect remote access** — all from **one unified IKEv2 configuration**, with **Cisco ISE (RADIUS)** authenticating every connection.

The whole point: *one flexible hub config accepts many connection types, and ISE decides who is approved and what they can reach.*

\---

## Why This Lab

Most VPN labs build one technology in isolation. Real enterprises aggregate **branches, partner firewalls, and remote workers** into one core. This lab proves that a single FlexVPN (IKEv2) hub can serve all three roles at once — and that AnyConnect does **not** require an ASA (the router terminates it directly over IKEv2). It ties together every VPN skill — DMVPN, site-to-site IPsec, remote access, and RADIUS identity — into one coherent design.

\---

## Topology

```
   INSIDE (Düsseldorf HQ — protected, behind the hub)
   ┌────────────────┐  ┌──────────────┐   ┌───────────────────┐
   │  Internal LAN  │  │  DMZ segment  │  │  ISE / RADIUS     │
   │  VPC8–VPC11    │  │ Server4–7     │  │   (inside)        │
   │ 10.20.20.0/24  │  │ 10.10.10.0/24 │  │  10.30.30.5       │
   └──────┬─────────┘  └──────┬────────┘  └─────────┬─────────┘
          │                   │                     │
       ┌──┴───────────────────┴─────────────────────┴────┐
       │              Sw-local (IOSvL2)                  │
       └───────────────────────┬─────────────────────────┘
                                │ e0/0  inside
                    ┌───────────┴────────────┐
                    │   HQ-DU — FlexVPN HUB  │  ← IOS-XE, IKEv2, EIGRP
                    │   IKEv2 · EIGRP · AAA  │    RADIUS client to ISE
                    │   e0/0 inside · e0/1 out
                    └───────────┬────────────┘
                                │ e0/1  outside
                        ╔═══════╧═══════╗
                        ║   Internet    ║   transport / "outside"
                        ║   (cloud)     ║   192.168.91.0/24
                        ╚═══╤══════╤═══╤═╝
             ┌──────────────┘      │   └──────────────┐
             │                     │                  │
   ┌─────────┴─────────┐  ┌────────┴────────┐  ┌──────┴──────────┐
   │  DMVPN spokes     │  │  ASA (Hamburg)  │  │  AnyConnect RA  │
   │  Stuttgart (S3)   │  │  Site-to-Site   │  │  Remote Pc      │
   │  Köln      (S2)   │  │  VTI IPsec IPv4 │  │  (Nürnberg)     │
   │  München   (S1)   │  │                 │  │  over IKEv2     │
   └───────────────────┘  └─────────────────┘  └─────────────────┘

   Auth flow:  remote device → (IKEv2) → HQ-DU hub → (RADIUS) → ISE (inside)
   Remote devices NEVER reach ISE directly; the hub proxies RADIUS to it.
```

**Roles**

|Element|Role|Technology|
|-|-|-|
|HQ-DU|FlexVPN concentrator (Düsseldorf)|IOS-XE, IKEv2, virtual-template, EIGRP, AAA→ISE|
|Sw-local|inter-segment switch|IOSvL2, carries LAN + DMZ + ISE to the hub|
|München (S1)|branch spoke|FlexVPN/IKEv2 dynamic spoke|
|Köln (S2)|branch spoke|FlexVPN/IKEv2 dynamic spoke|
|Stuttgart (S3)|branch spoke|FlexVPN/IKEv2 dynamic spoke|
|Hamburg (ASA)|partner firewall|site-to-site over VTI IPsec (ASA 9.7+)|
|Remote Pc (Nürnberg)|remote user|AnyConnect over IKEv2 to the hub (no ASA)|
|ISE|identity|RADIUS auth + authorization for every tunnel|

\---

## Addressing Plan

**Transport / "outside" — VMnet2 `192.168.91.0/24`**

|Device|Interface|IP|
|-|-|-|
|HQ-DU hub — outside|e0/1|192.168.91.129|
|Hamburg ASA — outside|e0/0|192.168.91.10|
|München (Spokes1) — outside|e0/0|192.168.91.21|
|Köln (Spokes2) — outside|e0/0|192.168.91.22|
|Stuttgart (Spokes3) — outside|e0/0|192.168.91.23|
|Remote Pc (AnyConnect)|eth0|192.168.91.128|

**Behind the HUB — the resources**

|Segment|Subnet|Host|
|-|-|-|
|Internal LAN|10.20.20.0/24|10.20.20.100|
|DMZ|10.10.10.0/24|10.10.10.100|
|Services (ISE)|10.30.30.0/24|**ISE = 10.30.30.5**|

> ISE sits on an \*\*inside\*\* services segment — never on the outside/transport network. The hub reaches it internally and acts as the RADIUS client for every tunnel.

**Tunnel / overlay**

|Overlay|Subnet|Notes|
|-|-|-|
|FlexVPN/DMVPN|172.16.0.0/24|hub .1, spokes get dynamic .x|
|AnyConnect pool|172.16.10.0/24|remote users draw from here|
|ASA VTI|172.16.20.0/30|hub .1, ASA .2|

**Behind each spoke / ASA (their LANs)**

|Site|LAN|
|-|-|
|München (Spokes1)|192.168.1.0/24|
|Köln (Spokes2)|192.168.2.0/24|
|Stuttgart (Spokes3)|192.168.3.0/24|
|Hamburg (ASA)|192.168.30.0/24|

\---

## Design Notes

> 🧠 \*\*FlexVPN vs DMVPN.\*\* FlexVPN is the IKEv2-based evolution of DMVPN. Rather than run both frameworks, this hub uses \*\*FlexVPN\*\* as the single framework — its spokes still give the DMVPN-style \*\*dynamic multipoint + spoke-to-spoke\*\*, but everything shares one IKEv2 config style. That is what lets one hub config also accept the ASA peer and AnyConnect users.

> 🧠 \*\*AnyConnect without an ASA.\*\* The AnyConnect client speaks \*\*IKEv2/IPsec\*\*, not only SSL. So the IOS-XE hub terminates remote-access users directly — no ASA in the remote-access path. The ASA instead appears as a \*\*site\*\* (VTI IPsec), which is the honest, correct way to include it.

> 🧠 \*\*ISE as the single gatekeeper.\*\* Every incoming tunnel — spoke, ASA, or AnyConnect user — is authenticated by ISE over RADIUS. ISE decides \*if\* the peer/user is allowed and \*what\* they get (IP pool, routes, authorization), so access is identity-scoped, not "connect and you're in."

\---

## Build Phases

This lab is built in phases — each one is verified end-to-end before the next, so you are never debugging six VPN types at once.

|Phase|What|Status|
|-|-|-|
|1|HUB + SW + LAN/DMZ reachable locally, ISE reachable|⬜|
|2|FlexVPN + Spoke1 → spoke reaches LAN/DMZ (EIGRP)|⬜|
|3|München + Stuttgart (multi-spoke + spoke-to-spoke)|📝 config ready|
|4|Hamburg ASA via VTI IPsec → reaches resources|📝 config ready|
|5|AnyConnect RA on the hub (Nürnberg → pool → resources)|📝 config ready|
|6|ISE (RADIUS) authenticates every tunnel|📝 config ready|

\---

## Phase 1 — Hub, Switch, LAN/DMZ

Goal: the hub routes between the internal LAN, DMZ, and services (ISE) segments — **before any VPN**.

```cisco
! ===== HQ-DU — FlexVPN hub — base + internal =====
hostname HQ-DU
ip routing

! Outside (transport / Internet cloud)
interface Ethernet0/1
 description OUTSIDE-INTERNET
 ip address 192.168.91.129 255.255.255.0
 no shutdown

! Inside toward Sw-local (sub-interfaces: LAN + DMZ + ISE/services)
interface Ethernet0/0
 no shutdown
interface Ethernet0/0.20
 description INTERNAL-LAN
 encapsulation dot1Q 20
 ip address 10.20.20.1 255.255.255.0
interface Ethernet0/0.10
 description DMZ
 encapsulation dot1Q 10
 ip address 10.10.10.1 255.255.255.0
interface Ethernet0/0.30
 description SERVICES-ISE
 encapsulation dot1Q 30
 ip address 10.30.30.1 255.255.255.0
```

```cisco
! ===== Sw-local (IOSvL2) — carry LAN + DMZ + ISE to the hub =====
hostname Sw-local
vlan 10
 name DMZ
vlan 20
 name LAN
vlan 30
 name SERVICES
! Trunk to the hub (HQ-DU e0/0)
interface Ethernet2/1
 description TO-HQ-DU
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
! DMZ servers (Server4–7)
interface range Ethernet0/1 - 3
 switchport mode access
 switchport access vlan 10
! Internal LAN (VPC8–11)
interface range Ethernet1/1 - 3
 switchport mode access
 switchport access vlan 20
! ISE (services)
interface Ethernet0/0
 description ISE
 switchport mode access
 switchport access vlan 30
```

**Verify Phase 1**

```cisco
! From HQ-DU (the hub):
ping 10.20.20.100        ! an Internal LAN host (VPC8–11)
ping 10.10.10.100        ! a DMZ server (Server4–7)
ping 10.30.30.5          ! ISE (inside services segment)
! A LAN VPC should reach a DMZ server through the hub (inter-segment routing)
```

\---

## Phase 2 — FlexVPN Hub + First Spoke (Köln) — EIGRP overlay

Goal: the first spoke (Köln / Spokes2) builds an IKEv2 tunnel to HQ-DU and reaches the LAN/DMZ, with EIGRP over the overlay. Build one spoke first, then clone for München and Stuttgart in Phase 3.

```cisco
! ===== HUB — FlexVPN (IKEv2) =====
! (pre-shared key shown for Phase 2; ISE/EAP comes in Phase 6)

crypto ikev2 keyring FLEX-KR
 peer ANY
  address 0.0.0.0 0.0.0.0
  pre-shared-key FLEXKEY123

crypto ikev2 profile FLEX-HUB
 match identity remote address 0.0.0.0
 authentication local pre-share
 authentication remote pre-share
 keyring local FLEX-KR
 virtual-template 1

crypto ipsec profile FLEX-IPSEC
 set ikev2-profile FLEX-HUB

interface Loopback0
 ip address 172.16.0.1 255.255.255.255

interface Virtual-Template1 type tunnel
 ip unnumbered Loopback0
 tunnel source Ethernet0/1
 tunnel protection ipsec profile FLEX-IPSEC

router eigrp 100
 network 172.16.0.1 0.0.0.0
 network 10.20.20.0 0.0.0.255
 network 10.10.10.0 0.0.0.255
 network 10.30.30.0 0.0.0.255
```

```cisco
! ===== Köln (Spokes2) — FlexVPN spoke =====
hostname Koln
crypto ikev2 keyring SP-KR
 peer HQDU
  address 192.168.91.129
  pre-shared-key FLEXKEY123

crypto ikev2 profile SP-PROFILE
 match identity remote address 192.168.91.129 255.255.255.255
 authentication local pre-share
 authentication remote pre-share
 keyring local SP-KR

crypto ipsec profile SP-IPSEC
 set ikev2-profile SP-PROFILE

interface Tunnel0
 ip address 172.16.0.12 255.255.255.0
 tunnel source Ethernet0/0
 tunnel destination 192.168.91.129
 tunnel protection ipsec profile SP-IPSEC

interface Ethernet0/0
 description OUTSIDE-INTERNET
 ip address 192.168.91.22 255.255.255.0
interface Ethernet0/1
 description KOLN-LAN
 ip address 192.168.2.1 255.255.255.0

router eigrp 100
 network 172.16.0.0 0.0.0.255
 network 192.168.2.0 0.0.0.255
```

**Verify Phase 2**

```cisco
! On HQ-DU (hub):
show crypto ikev2 sa            ! IKEv2 SA up with Köln
show crypto ipsec sa            ! IPsec SA / encrypted packets
show ip eigrp neighbors         ! EIGRP adjacency over the tunnel
! On Köln (spoke):
ping 10.20.20.100 source 192.168.2.1     ! reach the Internal LAN through the tunnel
ping 10.10.10.100 source 192.168.2.1     ! reach the DMZ
```

\---

## Phase 3 — München + Stuttgart spokes (multi-spoke + spoke-to-spoke)

Goal: add the other two spokes (clone Köln), then optionally enable **direct spoke-to-spoke** with NHRP redirect/shortcut.

**München (Spokes1)** — same as Köln, change: outside `.21`, tunnel `172.16.0.11`, LAN `192.168.1.0/24`.
**Stuttgart (Spokes3)** — same as Köln, change: outside `.23`, tunnel `172.16.0.13`, LAN `192.168.3.0/24`.

```cisco
! ===== München (Spokes1) — deltas from the Köln config =====
hostname Munchen
interface Tunnel0
 ip address 172.16.0.11 255.255.255.0
interface Ethernet0/0
 ip address 192.168.91.21 255.255.255.0
interface Ethernet0/1
 ip address 192.168.1.1 255.255.255.0
router eigrp 100
 network 172.16.0.0 0.0.0.255
 network 192.168.1.0 0.0.0.255
```

### Spoke-to-spoke — two ways

**By default (hub relay):** with EIGRP, the hub advertises every spoke's LAN to all spokes, so München reaches Stuttgart **through the hub** — works with zero extra config. Verify:

```cisco
Munchen# traceroute 192.168.3.1        ! path goes via HUB (172.16.0.1) → relay
```

**Direct spoke-to-spoke (FlexVPN Phase 3):** to build a direct tunnel that bypasses the hub, add NHRP redirect on the hub's virtual-template and shortcut on the spokes:

```cisco
! HUB virtual-template:
interface Virtual-Template1 type tunnel
 ip nhrp redirect

! Each SPOKE tunnel:
interface Tunnel0
 ip nhrp shortcut
```

```cisco
Munchen# traceroute 192.168.3.1        ! now goes DIRECT to Stuttgart (no hub)
show ip nhrp                            ! shows the dynamic spoke-to-spoke mapping
```

> 🧠 \*\*FlexVPN ≠ DMVPN here.\*\* You do \*\*not\*\* use the classic DMVPN `no ip split-horizon eigrp` / `no ip next-hop-self eigrp` commands. Hub-relay spoke-to-spoke works out of the box via EIGRP; direct spoke-to-spoke is enabled with \*\*NHRP redirect (hub) + shortcut (spoke)\*\* — the modern FlexVPN Phase 3 mechanism. Prove the difference with `traceroute` before and after.

\---

## Phase 4 — Hamburg ASA via VTI IPsec

Goal: the Hamburg ASA (9.7+) builds a **route-based VTI IPsec** tunnel to HQ-DU and its LAN (192.168.30.0/24) reaches the DMZ/LAN. VTI = the tunnel is a real interface you route over (static routing here, since the ASA isn't in EIGRP).

```cisco
! ===== Hamburg ASA (9.7+) — VTI IPsec to HQ-DU =====

interface GigabitEthernet0/0
 nameif outside
 security-level 0
 ip address 192.168.91.10 255.255.255.0
interface GigabitEthernet0/1
 nameif inside
 security-level 100
 ip address 192.168.30.1 255.255.255.0

! IKEv2 policy + proposal
crypto ikev2 policy 10
 encryption aes-256
 integrity sha256
 group 14
 prf sha256
 lifetime seconds 86400
crypto ikev2 enable outside

! IPsec proposal + profile
crypto ipsec ikev2 ipsec-proposal ASA-PROP
 protocol esp encryption aes-256
 protocol esp integrity sha-256
crypto ipsec profile ASA-VTI-PROFILE
 set ikev2 ipsec-proposal ASA-PROP

! Tunnel group (PSK to the hub's outside IP)
tunnel-group 192.168.91.129 type ipsec-l2l
tunnel-group 192.168.91.129 ipsec-attributes
 ikev2 remote-authentication pre-shared-key FLEXKEY123
 ikev2 local-authentication pre-shared-key FLEXKEY123

! The VTI interface (route-based)
interface Tunnel1
 nameif vti-hub
 ip address 172.16.20.2 255.255.255.252
 tunnel source interface outside
 tunnel destination 192.168.91.129
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile ASA-VTI-PROFILE

! Route the internal subnets across the VTI
route vti-hub 10.10.10.0 255.255.255.0 172.16.20.1     ! DMZ
route vti-hub 10.20.20.0 255.255.255.0 172.16.20.1     ! LAN
route outside 0.0.0.0 0.0.0.0 192.168.91.1             ! (transport default, if needed)
```

```cisco
! ===== HQ-DU hub — VTI side toward the ASA =====
crypto ikev2 keyring ASA-KR
 peer HAMBURG
  address 192.168.91.10
  pre-shared-key FLEXKEY123

crypto ikev2 profile ASA-PROFILE
 match identity remote address 192.168.91.10 255.255.255.255
 authentication local pre-share
 authentication remote pre-share
 keyring local ASA-KR

crypto ipsec profile ASA-IPSEC
 set ikev2-profile ASA-PROFILE

interface Tunnel20
 ip address 172.16.20.1 255.255.255.252
 tunnel source Ethernet0/1
 tunnel destination 192.168.91.10
 tunnel protection ipsec profile ASA-IPSEC

! Reach the ASA LAN, and advertise it into EIGRP so the inside knows it
ip route 192.168.30.0 255.255.255.0 172.16.20.2
router eigrp 100
 network 172.16.20.0 0.0.0.3
 redistribute static metric 10000 100 255 1 1500
```

**Verify Phase 4**

```cisco
! ASA:
show crypto ikev2 sa                       ! IKEv2 SA up to the hub
show interface Tunnel1                       ! vti-hub up/up
ping 10.10.10.100                            ! reach a DMZ server across the VTI
! Hub:
show crypto ikev2 sa | include 192.168.91.10
ping 192.168.30.1                            ! reach the ASA inside
```

> 🧠 \*\*Why VTI (route-based) instead of crypto-map (policy-based):\*\* VTI gives the ASA a real tunnel \*interface\* you route over — cleaner, supports dynamic routing, and matches the FlexVPN "everything is an interface" model. Crypto-map (policy-based) would work on older ASA but ties encryption to ACLs and doesn't route as cleanly.

\---

## Phase 5 — AnyConnect Remote Access on the Hub (no ASA)

Goal: the Remote Pc (Nürnberg) connects with **AnyConnect over IKEv2** directly to HQ-DU — no ASA — gets an IP from a local pool, and reaches the DMZ/LAN. Users authenticate with **EAP** (username/password), which Phase 6 points at ISE.

```cisco
! ===== HQ-DU hub — FlexVPN Remote Access (AnyConnect over IKEv2) =====

! AAA (local for now; Phase 6 switches this to ISE/RADIUS)
aaa new-model
aaa authentication login RA-AUTH local
aaa authorization network RA-AUTHZ local
username vpnuser password 0 VpnPass123

! Address pool for remote clients
ip local pool RA-POOL 172.16.10.10 172.16.10.100

! IKEv2 local authorization policy (what the client gets)
crypto ikev2 authorization policy RA-POLICY
 pool RA-POOL
 dns 208.67.222.222
 route set interface
 def-domain lab.local

! IKEv2 proposal/policy (AnyConnect-compatible)
crypto ikev2 proposal RA-PROP
 encryption aes-cbc-256
 integrity sha256
 group 14
crypto ikev2 policy RA-IKEPOL
 proposal RA-PROP

! The hub authenticates with a certificate (RSA-SIG); client uses EAP
crypto ikev2 profile RA-PROFILE
 match identity remote any
 authentication local rsa-sig
 authentication remote eap query-identity
 pki trustpoint HQDU-TP
 aaa authentication eap RA-AUTH
 aaa authorization group eap list RA-AUTHZ RA-POLICY
 virtual-template 10

crypto ipsec profile RA-IPSEC
 set ikev2-profile RA-PROFILE

! Virtual-template the remote users clone from
interface Virtual-Template10 type tunnel
 ip unnumbered Ethernet0/0.20
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile RA-IPSEC

! (Trustpoint HQDU-TP = a self-signed or CA cert on the hub for the
!  server side of EAP — generate with: crypto pki enroll / self-signed)
```

**AnyConnect client (Remote Pc) profile — force IKEv2/IPsec**

```xml
<!-- AnyConnect XML profile snippet -->
<PrimaryProtocol>IPsec</PrimaryProtocol>
<!-- connect to the hub's outside IP -->
<HostAddress>192.168.91.129</HostAddress>
```

Username: `vpnuser`  Password: `VpnPass123` (local now; ISE in Phase 6).

**Verify Phase 5**

```cisco
! Hub, after the client connects:
show crypto ikev2 sa                         ! SA for the remote user
show crypto session                          ! virtual-access up
show ip local pool RA-POOL                   ! an address handed out
! From the Remote Pc (once connected):
ping 10.20.20.100                            ! reach the Internal LAN
ping 10.10.10.100                            ! reach the DMZ
```

> 🧠 \*\*AnyConnect without an ASA:\*\* the AnyConnect client speaks IKEv2/IPsec, so the IOS-XE hub terminates it directly. The hub authenticates with a certificate (rsa-sig); the user authenticates with EAP (username/password) — which Phase 6 validates against ISE. This is "FlexVPN Remote Access."

\---

## Phase 6 — ISE (RADIUS) Authenticates Everything

Goal: swap the local AAA to **ISE over RADIUS**, so AnyConnect users (and, if desired, spoke/peer auth) are validated by ISE and scoped by policy. The hub is the **RADIUS client**; ISE never talks to the remote devices directly.

```cisco
! ===== HQ-DU hub — point AAA at ISE =====
radius server ISE
 address ipv4 192.168.91.5 auth-port 1812 acct-port 1813
 key WATER

aaa group server radius AZG-ISE
 server name ISE

! Switch the RA auth from local → ISE
aaa authentication login RA-AUTH group AZG-ISE
aaa authorization network RA-AUTHZ group AZG-ISE local
aaa accounting network default start-stop group AZG-ISE

! (The RA-PROFILE from Phase 5 now uses these ISE-backed lists)
```

**ISE side**

```
1. Administration → Network Resources → Network Devices → + Add
     Name: HQ-DU
     IP Address: 192.168.91.6   (the hub's ISE-facing interface)
     RADIUS Shared Secret: WATER

2. Administration → Identity Management → Identities → Users → + Add
     Username: vpnuser
     Password: VpnPass123

3. Policy → Policy Sets → (default or a custom set)
     Allowed Protocols: include EAP-MSCHAPv2 / PEAP for the AnyConnect users
     Authorization: PermitAccess (or a scoped dACL/VLAN result)

4. Operations → RADIUS → Live Logs  (watch auth in real time)
```

**Verify Phase 6**

```cisco
! Hub — quick RADIUS reachability test:
test aaa group AZG-ISE vpnuser VpnPass123 legacy
! then connect AnyConnect and watch:
show crypto ikev2 sa
```

```
ISE → Operations → RADIUS → Live Logs:
  green Pass for vpnuser, EAP method, Authorization result ✅
```

> 🧠 \*\*The hub is the single RADIUS client.\*\* Every remote identity (AnyConnect users here; spoke/peer auth if you extend it) is authenticated by the hub \*on behalf of\* the network, querying ISE inside. ISE decides who is allowed and what they get — identity-scoped access, not "connect and you're in."

> ⚠️ \*\*Lab note — ISE placement:\*\* in this build ISE sits on the management/transport subnet (192.168.91.0/24) as a simplification, with the hub reaching it at 192.168.91.5. In production, ISE belongs on an isolated inside services segment (e.g. 10.30.30.0/24); the hub would proxy RADIUS to it there. Functionally identical — the remote devices never reach ISE directly either way.

\---

## Troubleshooting Log

*Document each real issue and its fix here as you build — this is the most valuable part of the writeup.*

\---

## Key Concepts Reference

* **FlexVPN** — a unified IKEv2 VPN framework on IOS-XE; one config style serves site-to-site, hub-and-spoke (dynamic multipoint), and remote access.
* **IKEv2** — the key-exchange/authentication protocol underneath FlexVPN; more flexible and robust than IKEv1.
* **Virtual-Template / Virtual-Access** — the hub clones a per-peer tunnel interface from one template, which is how one config serves many spokes/users.
* **VTI (Virtual Tunnel Interface)** — route-based IPsec; the tunnel is a real interface you route over (used for the ASA site).
* **AnyConnect over IKEv2** — the client connecting to a router (not an ASA) for remote access.
* **RADIUS / ISE** — centralized authentication and authorization for every tunnel; identity decides access.

\---

