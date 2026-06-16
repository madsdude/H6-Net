# 06 - ISP core P/PE design

Dette dokument beskriver ISP core-laget i H6-Net.

## Designmål

Providerlaget skal kun levere transport. Det skal ikke kende kundens site-routing ud over den Layer 2 transport, der leveres via xconnect.

Providerlaget består af:

| Enhed | Rolle | Loopback | Funktion |
| --- | --- | --- | --- |
| ISP-R1 | PE-side/xconnect-endepunkt | 1.1.1.1/32 | Dot1Q + xconnect mod remote PE |
| P1 | P-router | 2.2.2.2/32 | MPLS transit |
| P2 | P-router | 3.3.3.3/32 | MPLS transit |
| Remote PE | Mangler config | 4.4.4.4/32 | Modsat xconnect-endepunkt |

## OSPF core

Providerlaget bruger `router ospf 10`.

| Enhed | Router-id | OSPF net |
| --- | --- | --- |
| ISP-R1 | 1.1.1.1 | 1.1.1.1/32, 10.0.12.0/30, 10.0.13.0/30 |
| P1 | 2.2.2.2 | 2.2.2.2/32, 10.0.12.0/30, 10.0.24.0/30, 10.0.23.0/30 |
| P2 | 3.3.3.3 | 3.3.3.3/32, 10.0.13.0/30, 10.0.34.0/30, 10.0.23.0/30 |

## MPLS/LDP

P1, P2 og ISP-R1 har MPLS aktiveret på provider links. P1 og P2 har også:

```text
mpls ldp router-id Loopback0 force
```

Det anbefales at standardisere samme kommando på ISP-R1, så LDP router-id altid er Loopback0.

## BFD

Provider links bruger BFD med:

```text
bfd interval 50 min_rx 50 multiplier 3
```

OSPF har `bfd all-interfaces`, så OSPF kan reagere hurtigere på linkfejl.

## Interfaceplan

| Enhed | Interface | Beskrivelse | IP | MPLS | BFD |
| --- | --- | --- | --- | --- | --- |
| ISP-R1 | Gi0/0 | CORE-TO-P1 | 10.0.12.1/30 | Ja | Ja |
| ISP-R1 | Gi0/2 | CORE-TO-P2 | 10.0.13.1/30 | Ja | Ja |
| P1 | Gi0/0 | CORE-TO-PE1 | 10.0.12.2/30 | Ja | Ja |
| P1 | Gi0/1 | CORE-TO-PE2 | 10.0.24.1/30 | Ja | Ja |
| P2 | Gi0/0 | CORE-TO-PE1 | 10.0.13.2/30 | Ja | Ja |
| P2 | Gi0/1 | CORE-TO-PE2 | 10.0.34.1/30 | Ja | Ja |

## Standardisering / cleanup

| Punkt | Status | Anbefaling |
| --- | --- | --- |
| ISP-R1 filnavn vs hostname | Fil hedder ISP-PE1, hostname er ISP-R1 | Vælg én navnestandard |
| Remote PE config | Mangler i Today-configs | Upload/dokumentér PE2/remote PE |
| LDP router-id på ISP-R1 | Ikke fundet | Tilføj `mpls ldp router-id Loopback0 force` |
| OSPF 10.0.23.0/30 på P1/P2 | Network statement findes, interface ikke fundet | Fjern eller dokumentér linket |
