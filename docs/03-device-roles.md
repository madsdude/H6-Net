# 03 - Device roles

Dette dokument beskriver rollerne for P-, PE- og CE-enheder i H6-Net.

## Rollemodel

| Rolle | Enheder | Ansvar | Protokoller |
| --- | --- | --- | --- |
| CORE CE / internet breakout | CE-CORE-R1 | Central routing, NAT og default route til sites | BGP AS65000, OSPF 100, NAT |
| Site CE1 | CE-AAH-R1, CE-KBH-R1, CE-ODE-R1 | Primær CE på site | eBGP til CORE, iBGP til CE2, OSPF 100 under migration |
| Site CE2 | CE-AAH-R2, CE-KBH-R2, CE-ODE-R2 | Sekundær CE på site | eBGP til CORE, iBGP til CE1, OSPF 100 under migration |
| PE-side | ISP-R1 | Terminerer VLAN subinterfaces og xconnect | OSPF 10, MPLS/LDP, xconnect, BFD |
| P-router | P1, P2 | Ren MPLS transit i provider core | OSPF 10, MPLS/LDP, BFD |
| Remote PE | Ikke uploadet | Modsat xconnect-endepunkt | Forventet OSPF/MPLS/LDP/xconnect |

## CE-CORE-R1

CE-CORE-R1 er kundens centrale router og internet breakout.

Funktioner:

- Loopback/router-id: `11.11.11.11/32`
- BGP AS: `65000`
- eBGP mod AAH, KBH og ODE
- Sender default route til sites med `default-originate`
- Bruger route-maps til at gøre primær path bedre end sekundær path
- NAT overload via `GigabitEthernet0/1`
- Midlertidig OSPF 100 under migration

## Site CE-routere

Hvert site har to CE-routere.

| Site | CE1 | CE2 | AS |
| --- | --- | --- | --- |
| AAH | CE-AAH-R1 | CE-AAH-R2 | 65010 |
| KBH | CE-KBH-R1 | CE-KBH-R2 | 65020 |
| ODE | CE-ODE-R1 | CE-ODE-R2 | 65030 |

CE1 og CE2 på samme site har iBGP imellem sig over et interconnect /30-net. De annoncerer hver deres loopback ind i BGP.

## ISP-R1 / PE-side

Filen hedder `configs/Today-configs/ISP-PE1`, men selve configens hostname er `ISP-R1`.

ISP-R1 har følgende funktioner:

- OSPF 10 mod P1 og P2
- MPLS på provider links
- BFD på provider links
- Trunk fra SW-DIST-1 på Gi0/1
- Dot1Q subinterfaces pr. EPL service
- `xconnect 4.4.4.4 <VC-ID> encapsulation mpls`

## P1 og P2

P1 og P2 er rene provider core-routere. De skal ikke kende kundens BGP-ruter. De skal kun transportere MPLS labels og have reachability til PE loopbacks.

Begge bruger:

```text
mpls label protocol ldp
mpls ip
mpls ldp router-id Loopback0 force
router ospf 10
bfd all-interfaces
```

## Standardprincip

CE-laget og ISP-laget skal holdes adskilt:

- ISP core skal ikke køre kundens BGP AS'er.
- P-routere skal ikke have CE/site-ruter.
- EPL/xconnect skal kun levere Layer 2 transport.
- CE-routerne står for Layer 3, BGP path selection, default route og NAT-flow.
