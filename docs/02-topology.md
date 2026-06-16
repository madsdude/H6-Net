# 02 - Topologi

Dette dokument beskriver den fysiske og logiske topologi i H6-Net baseret på `configs/Today-configs`.

## Overordnet logisk topologi

```text
                         Internet / upstream
                                |
                           CE-CORE-R1
                           Gi0/1 NAT outside
                                |
                  Gi0/0 trunk mod EPL/ISP layer
                                |
                    SW-DIST / EPL access layer
                                |
                  ISP-R1 / PE-side xconnect
                         /                  \
                       P1                    P2
                         \                  /
                       Remote PE 4.4.4.4
                                |
              EPL VLANs mod AAH / KBH / ODE sites
```

## Providertransport

Providerlaget består i de uploadede configs af:

| Enhed | Rolle | Loopback | Funktion |
| --- | --- | --- | --- |
| ISP-R1 | PE-side / xconnect-endepunkt | 1.1.1.1/32 | Dot1Q subinterfaces og MPLS xconnect |
| P1 | Provider core | 2.2.2.2/32 | MPLS/LDP transit |
| P2 | Provider core | 3.3.3.3/32 | MPLS/LDP transit |
| Remote PE | Ikke fundet som config | 4.4.4.4/32 | Xconnect peer for ISP-R1 |

`ISP-R1` har xconnect mod `4.4.4.4` på flere VLAN/VC-ID'er. Det betyder, at der bør findes en remote PE-router med loopback `4.4.4.4`, men den config ligger ikke i `Today-configs`.

## Provider links

| Link | Side A | Side B | Net | Bemærkning |
| --- | --- | --- | --- | --- |
| ISP-R1 - P1 | ISP-R1 Gi0/0 `10.0.12.1` | P1 Gi0/0 `10.0.12.2` | 10.0.12.0/30 | OSPF 10, MPLS, BFD |
| ISP-R1 - P2 | ISP-R1 Gi0/2 `10.0.13.1` | P2 Gi0/0 `10.0.13.2` | 10.0.13.0/30 | OSPF 10, MPLS, BFD |
| P1 - Remote PE | P1 Gi0/1 `10.0.24.1` | Remote PE forventet `10.0.24.2` | 10.0.24.0/30 | P1 config viser link mod PE2 |
| P2 - Remote PE | P2 Gi0/1 `10.0.34.1` | Remote PE forventet `10.0.34.2` | 10.0.34.0/30 | P2 config viser link mod PE2 |

## CE-/kundelag

```text
                CE-CORE-R1 AS65000
                 /       |       \
              AAH       KBH       ODE
           AS65010    AS65020    AS65030
          CE1--CE2   CE1--CE2   CE1--CE2
```

Hvert site har to CE-routere:

- CE1 er primær site-router.
- CE2 er sekundær site-router.
- CE1 og CE2 har et interconnect /30-net mellem sig.
- Begge CE-routere peeringer mod CE-CORE-R1 med eBGP.
- CE1 og CE2 på samme site peeringer med iBGP.

## CORE trunk/subinterfaces

CE-CORE-R1 bruger `GigabitEthernet0/0` som trunk mod EPL-laget. De aktive nye subinterfaces er:

| VLAN | CORE subinterface | Site | CE | CORE IP |
| --- | --- | --- | --- | --- |
| 101 | Gi0/0.101 | AAH | CE1 primary | 10.10.0.1/30 |
| 102 | Gi0/0.102 | AAH | CE2 secondary | 10.10.0.5/30 |
| 201 | Gi0/0.201 | KBH | CE1 primary | 10.20.0.1/30 |
| 202 | Gi0/0.202 | KBH | CE2 secondary | 10.20.0.5/30 |
| 301 | Gi0/0.301 | ODE | CE1 primary | 10.30.0.1/30 |
| 302 | Gi0/0.302 | ODE | CE2 secondary | 10.30.0.5/30 |

Følgende gamle EPL VLAN-subinterfaces findes stadig på CE-CORE-R1, men er shutdown og uden IP:

```text
100, 110, 200, 210, 300, 310
```

Disse kan fjernes, når migrationen er dokumenteret og verificeret færdig.
