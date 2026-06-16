# 06 - ISP core P/PE design

Dette dokument beskriver ISP core-laget i H6-Net.

## Designmål

Providerlaget skal levere transport, MPLS/EPL og separat ISP-management. Kundens site-routing og kundens internet breakout skal fortsat ligge på CE-laget.

Providerlaget består af:

| Enhed | Rolle | Loopback | Funktion |
| --- | --- | --- | --- |
| ISP-R1 / PE1 | PE-side/xconnect-endepunkt | 1.1.1.1/32 | Dot1Q + xconnect mod PE2 |
| P1 | P-router + ISP NAT breakout | 2.2.2.2/32 | MPLS transit og internet/NAT for ISP-management |
| P2 | P-router | 3.3.3.3/32 | MPLS transit |
| ISP-R2 / PE2 | PE-side/xconnect-endepunkt | 4.4.4.4/32 | Dot1Q + xconnect mod PE1 |

## OSPF core

Providerlaget bruger `router ospf 10`.

| Enhed | Router-id | OSPF net |
| --- | --- | --- |
| ISP-R1 / PE1 | 1.1.1.1 | 1.1.1.1/32, 10.0.12.0/30, 10.0.13.0/30 |
| P1 | 2.2.2.2 | 2.2.2.2/32, 10.0.12.0/30, 10.0.24.0/30, 10.0.23.0/30 |
| P2 | 3.3.3.3 | 3.3.3.3/32, 10.0.13.0/30, 10.0.34.0/30, 10.0.23.0/30 |
| ISP-R2 / PE2 | 4.4.4.4 | 4.4.4.4/32, 10.0.24.0/30, 10.0.34.0/30 |

P1 annoncerer default route i OSPF for ISP-udstyret:

```text
router ospf 10
 default-information originate
```

Det betyder, at P2, PE1 og PE2 kan sende management-/internettrafik mod P1 uden at bruge CE-CORE-R1.

## MPLS/LDP

P1, P2, ISP-R1 og ISP-R2 har MPLS aktiveret på provider links.

P1 og P2 har:

```text
mpls ldp router-id Loopback0 force
```

Det anbefales at standardisere samme kommando på PE-routerne, så LDP router-id altid er Loopback0.

## BFD

Provider links bruger BFD med:

```text
bfd interval 50 min_rx 50 multiplier 3
```

OSPF har `bfd all-interfaces`, så OSPF kan reagere hurtigere på linkfejl.

## Interfaceplan

| Enhed | Interface | Beskrivelse | IP | MPLS | BFD |
| --- | --- | --- | --- | --- | --- |
| ISP-R1 / PE1 | Gi0/0 | CORE-TO-P1 | 10.0.12.1/30 | Ja | Ja |
| ISP-R1 / PE1 | Gi0/2 | CORE-TO-P2 | 10.0.13.1/30 | Ja | Ja |
| P1 | Gi0/0 | CORE-TO-PE1 | 10.0.12.2/30 | Ja | Ja |
| P1 | Gi0/1 | CORE-TO-PE2 | 10.0.24.1/30 | Ja | Ja |
| P1 | Gi0/2 | ISP-INTERNET/NAT-OUTSIDE | DHCP | Nej | Nej |
| P2 | Gi0/0 | CORE-TO-PE1 | 10.0.13.2/30 | Ja | Ja |
| P2 | Gi0/1 | CORE-TO-PE2 | 10.0.34.1/30 | Ja | Ja |
| ISP-R2 / PE2 | Gi0/0 | CORE-TO-P1 | 10.0.24.2/30 | Ja | Ja |
| ISP-R2 / PE2 | Gi0/2 | CORE-TO-P2 | 10.0.34.2/30 | Ja | Ja |

## ISP NAT breakout på P1

P1 fungerer nu som ISP-sidens NAT breakout.

Aktuel model:

```text
interface GigabitEthernet0/0
 description CORE-TO-PE1
 ip nat inside
!
interface GigabitEthernet0/1
 description CORE-TO-PE2
 ip nat inside
!
interface GigabitEthernet0/2
 ip address dhcp
 ip nat outside
!
ip nat inside source list 1 interface GigabitEthernet0/2 overload
access-list 1 permit any
```

Dette er adskilt fra CE-CORE-R1, som fortsat er kundens/site internet breakout.

### Oprydning på NAT ACL

`access-list 1 permit any` virker til lab-test, men bør erstattes med en mere præcis ACL, når SW-CE management er færdigtestet.

Anbefalet:

```text
ip access-list standard NAT-ISP-MGMT
 permit 1.1.1.1
 permit 2.2.2.2
 permit 3.3.3.3
 permit 4.4.4.4
 permit 10.0.12.0 0.0.0.3
 permit 10.0.13.0 0.0.0.3
 permit 10.0.24.0 0.0.0.3
 permit 10.0.34.0 0.0.0.3
 permit 10.10.10.0 0.0.0.255
 permit 10.20.10.0 0.0.0.255
```

## SW-CE management via PE

SW-CE skal ikke bruge CE-CORE-R1 som gateway. Gateway skal ligge på PE-siden.

| Management-net | Gateway | Gateway-enhed |
| --- | --- | --- |
| 10.10.10.0/24 | 10.10.10.1 | ISP-R1 / PE1 |
| 10.20.10.0/24 | 10.20.10.1 | ISP-R2 / PE2 |

PE-routerne skal annoncere managementnettene i OSPF 10, så P1 kan route retur til SW-CE-nettene.

## Standardisering / cleanup

| Punkt | Status | Anbefaling |
| --- | --- | --- |
| ISP-R1 filnavn vs hostname | Fil hedder ISP-PE1, hostname er ISP-R1 | Vælg én navnestandard |
| ISP-R2 filnavn vs hostname | Fil hedder ISP-PE2, hostname er ISP-R2 | Vælg én navnestandard |
| LDP router-id på PE-routere | Ikke konsekvent dokumenteret | Tilføj `mpls ldp router-id Loopback0 force` |
| P1 NAT ACL | `access-list 1 permit any` | Stram til `NAT-ISP-MGMT` efter test |
| OSPF 10.0.23.0/30 på P1/P2 | Network statement findes, interface ikke fundet | Fjern eller dokumentér linket |
