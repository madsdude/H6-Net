# Device Roles

## P Router

P-routeren er provider core-routeren. Den kender kun providerens interne routing og MPLS-labels.

Typiske funktioner:

- OSPF i provider core
- MPLS LDP
- Transport mellem PE-routere
- Ingen kunde-VRF
- Ingen CE-BGP direkte
- Ingen NAT
- Ingen customer policy

## PE Router

PE-routeren er kanten af provider-netværket. Den forbinder CE-/kundesiden med MPLS-core.

Typiske funktioner:

- OSPF mod P/PE-core
- MPLS LDP mod P-routere
- Xconnect / pseudowire / EPL
- VLAN/subinterface mod CE-transport
- Terminerer Layer 2 services
- Har loopback som transport-endpoint

## CE Router

CE-routeren er kundens router.

Typiske funktioner:

- BGP mod CORE eller anden CE/edge
- Default route mod CORE
- Site routing
- Eventuelt OSPF midlertidigt under migration
- LAN/WAN interface mod kundeside
