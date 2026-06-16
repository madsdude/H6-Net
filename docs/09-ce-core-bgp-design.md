# 09 - CE/CORE BGP design

Dette dokument beskriver BGP-designet mellem CE-CORE-R1 og sites.

## AS-plan

| Enhed/site | AS | Router-id/loopback |
| --- | --- | --- |
| CE-CORE-R1 | 65000 | 11.11.11.11 |
| AAH | 65010 | 22.22.22.22 / 22.22.22.23 |
| KBH | 65020 | 33.33.33.33 / 33.33.33.34 |
| ODE | 65030 | 44.44.44.44 / 44.44.44.45 |

## BGP-princip

- CE-CORE-R1 har eBGP til CE1 og CE2 på hvert site.
- CE1 og CE2 på samme site har iBGP imellem sig.
- CE-CORE-R1 annoncerer default route til sites med `default-originate`.
- Sites annoncerer deres loopbacks ind i BGP med `network <loopback> mask 255.255.255.255`.
- Primær path får bedre local preference end sekundær path på CORE.
- Default route til primær CE får lavere BGP metric end sekundær CE.

## CORE BGP neighbors

| Site | Neighbor | Beskrivelse | Remote AS | Rolle |
| --- | --- | --- | --- | --- |
| AAH | 10.10.0.2 | AAH-CE1-PRIMARY | 65010 | Primary |
| AAH | 10.10.0.6 | AAH-CE2-SECONDARY | 65010 | Secondary |
| KBH | 10.20.0.2 | KBH-CE1-PRIMARY | 65020 | Primary |
| KBH | 10.20.0.6 | KBH-CE2-SECONDARY | 65020 | Secondary |
| ODE | 10.30.0.2 | ODE-CE1-PRIMARY | 65030 | Primary |
| ODE | 10.30.0.6 | ODE-CE2-SECONDARY | 65030 | Secondary |

## Site iBGP neighbors

| Site | CE1 | CE2 | Interconnect |
| --- | --- | --- | --- |
| AAH | 10.10.0.9 | 10.10.0.10 | 10.10.0.8/30 |
| KBH | 10.20.0.9 | 10.20.0.10 | 10.20.0.8/30 |
| ODE | 10.30.0.9 | 10.30.0.10 | 10.30.0.8/30 |

## Route-maps på CE-CORE-R1

CE-CORE-R1 bruger route-maps til at styre primær/sekundær path.

```text
route-map DEFAULT-PRIMARY permit 10
 set metric 0
!
route-map DEFAULT-SECONDARY permit 10
 set metric 200
!
route-map SITE-PRIMARY-IN permit 10
 set local-preference 200
!
route-map SITE-SECONDARY-IN permit 10
 set local-preference 100
```

Effekt:

| Retning | Route-map | Effekt |
| --- | --- | --- |
| CORE -> CE1 | DEFAULT-PRIMARY | Default route ser bedst ud via CE1 |
| CORE -> CE2 | DEFAULT-SECONDARY | Default route ser sekundær ud via CE2 |
| Site -> CORE via CE1 | SITE-PRIMARY-IN | CORE foretrækker site-ruter via CE1 |
| Site -> CORE via CE2 | SITE-SECONDARY-IN | CORE bruger CE2 som backup |

## Standard site-template

For et nyt site bruges samme model:

```text
router bgp <SITE-AS>
 bgp router-id <CE-loopback>
 bgp log-neighbor-changes
 network <CE-loopback> mask 255.255.255.255
 neighbor <CORE-IP> remote-as 65000
 neighbor <CORE-IP> description CORE-PRIMARY-or-SECONDARY
 neighbor <LOCAL-CE-PEER> remote-as <SITE-AS>
 neighbor <LOCAL-CE-PEER> description <SITE>-CE-iBGP
 neighbor <LOCAL-CE-PEER> next-hop-self
```

## Verification commands

På CE-CORE-R1:

```bash
show ip bgp summary
show ip bgp
show ip route bgp
show ip route 22.22.22.22
show ip route 22.22.22.23
show ip route 33.33.33.33
show ip route 33.33.33.34
show ip route 44.44.44.44
show ip route 44.44.44.45
```

På site CE:

```bash
show ip bgp summary
show ip route bgp
show ip route 0.0.0.0
ping 11.11.11.11 source Loopback0
traceroute 8.8.8.8 source Loopback0
```

## Standardisering

BGP timers er ikke helt ens på alle neighbors i de aktuelle configs. Hvis labbet skal være helt standardiseret, bør du vælge én timer-policy, fx:

```text
neighbor <peer> timers 5 15
```

og bruge den ens på alle CE-CORE eBGP peers samt site eBGP peers.
