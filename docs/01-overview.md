# 01 - Overview

H6-Net dokumenterer et netværkslab med et provider transportlag og et customer routinglag.

## Kort beskrivelse

Labbet bygger et design, hvor P- og PE-routere leverer MPLS-baseret transport, mens CE-routere bruger EPL-forbindelserne til Layer 3 routing med BGP. CORE-routeren fungerer som central internet breakout med NAT.

## Designmål

- Bygge et tydeligt skel mellem provider transport og customer routing
- Dokumentere P-, PE- og CE-roller
- Dokumentere EPL/xconnect over MPLS
- Bruge BGP mellem CORE og sites
- Have primær og sekundær path pr. site
- Kunne teste failover kontrolleret
- Standardisere configs og navngivning
- Gøre labbet nemt at udvide med nye sites

## Lag i designet

| Lag | Protokoller / funktioner | Formål |
| --- | --- | --- |
| Provider core | OSPF, MPLS, LDP | Transport mellem PE-routere |
| Provider edge | Xconnect, subinterfaces, VLANs | Terminerer EPL-services |
| EPL transport | Layer 2 pseudowire | Transparent forbindelse mellem CE-sider |
| CE routing | eBGP, midlertidig OSPF | Routing mellem CORE og sites |
| Internet breakout | NAT, default route | Central internetadgang via CORE |

## Slutmål

Slutdesignet skal ende med, at CE-laget primært bruger BGP, mens OSPF kun bruges i provider core. OSPF på CE-WAN kan bruges midlertidigt under migration, men bør fjernes, når BGP-designet er stabilt.
