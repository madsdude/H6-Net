# 03 - Device Roles

Dette dokument beskriver rollerne for P-, PE-, CE- og switch-enheder i H6-Net.

## P Router

P-routeren er provider core-routeren. Den transporterer MPLS-labels mellem PE-routere og bør ikke have kundespecifik routing.

Typiske funktioner:

- OSPF i provider core
- MPLS på core-links
- LDP neighbor relationships
- Transport mellem PE-routere
- Ingen kunde-VRF i dette EPL-design
- Ingen CE-BGP neighbors
- Ingen NAT
- Ingen customer policy

## PE Router

PE-routeren er provider edge. Den forbinder CE-/switch-laget med MPLS-core og terminerer xconnect/pseudowire-services.

Typiske funktioner:

- OSPF mod P/PE-core
- MPLS/LDP mod provider core
- Loopback som xconnect endpoint
- Subinterfaces med dot1q VLANs
- Xconnect / pseudowire mod remote PE
- Layer 2 service transport for CE-sider

## CE Router

CE-routeren er kundesidens router. Den kører BGP mod CORE eller site edge afhængigt af designet.

Typiske funktioner:

- eBGP mod CE-CORE-R1
- Site routing
- Default route mod CORE
- Loopback til router-id/test
- Midlertidig OSPF under migration
- Ingen MPLS/LDP

## Switches

Switches bruges til VLAN-transport mellem CE-routere og PE-/distribution-laget.

Typiske funktioner:

- Access VLANs til CE
- Trunks mod PE/distribution
- VLAN separation pr. EPL-service
- Allowed VLAN-lister skal holdes ens på relevante trunks
