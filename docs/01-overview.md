# 01 - Overblik over H6-Net

H6-Net dokumenterer et ISP-/enterprise-lignende WAN-design med MPLS-transport, EPL/xconnect, CE-routing, BGP og central internet breakout.

Dokumentationen er baseret på configs i:

```text
configs/Today-configs/
```

## Formål

Labbet viser, hvordan et provider-netværk kan levere transparente Layer 2 EPL-forbindelser mellem kundens CE-routere, mens kundens routing bygges ovenpå med BGP og midlertidig OSPF under migration.

CORE-sitet fungerer som central internet breakout. De øvrige sites modtager default route fra CORE via eBGP og sender internettrafik tilbage til CORE, hvor NAT udføres.

## Designlag

| Lag | Funktion | Teknologi |
| --- | --- | --- |
| ISP Core | Transport mellem PE-routere | OSPF 10, MPLS, LDP, BFD |
| P-routere | Ren provider core forwarding | OSPF, MPLS/LDP |
| PE-routere | EPL/xconnect termination | Dot1Q subinterfaces, MPLS xconnect |
| EPL Layer 2 | Transparent WAN-transport | VLAN til VC-ID mapping |
| CE Routing | Kundens Layer 3 routing | eBGP, iBGP og midlertidig OSPF 100 |
| Internet breakout | Central udgang mod internet | NAT overload på CE-CORE-R1 |

## Aktive hovedkomponenter

| Enhed | Rolle | Primær funktion |
| --- | --- | --- |
| CE-CORE-R1 | Central CE / internet breakout | BGP AS 65000, NAT, default route til sites |
| CE-AAH-R1 / CE-AAH-R2 | AAH site CE-par | AS 65010, primær/sekundær CE-design |
| CE-KBH-R1 / CE-KBH-R2 | KBH site CE-par | AS 65020, primær/sekundær CE-design |
| CE-ODE-R1 / CE-ODE-R2 | ODE site CE-par | AS 65030, primær/sekundær CE-design |
| ISP-R1 | PE-side mod CORE/SW-DIST | MPLS xconnect mod remote PE |
| P1 / P2 | ISP core routers | MPLS/LDP transit med OSPF 10 |

## Routingprincip

```text
Site CE1/CE2
  -> EPL/xconnect via ISP MPLS core
  -> CE-CORE-R1
  -> NAT overload
  -> Internet / upstream gateway
```

CE-CORE-R1 annoncerer default route til site-CE'erne via BGP. Site-routerne annoncerer deres loopbacks tilbage til CORE. CE1 og CE2 på samme site bruger iBGP imellem sig, så begge lokale CE-routere kender hinandens site-ruter.

## Nuværende status

- Ny to-CE plan er aktiv for AAH, KBH og ODE.
- CORE bruger subinterfaces pr. site og CE-router.
- BGP er aktivt mellem CORE og sites.
- OSPF 100 findes stadig på CE-laget som migrationshjælp.
- NAT ACL er opdateret med CE1/CE2 loopbacks og nye WAN-/interconnect-net.
- Providerlaget bruger OSPF 10, MPLS/LDP og BFD.

## Vigtige noter

- `configs/Today-configs/ISP-PE1` har hostname `ISP-R1`. Dokumentationen bruger hostname `ISP-R1`, men nævner filnavnet hvor det er relevant.
- Remote PE med xconnect peer `4.4.4.4` er ikke fundet som config i `Today-configs`. Den er derfor dokumenteret som en manglende/må verificeres enhed.
- P1 og P2 har OSPF network statement for `10.0.23.0/30`, men der er ikke fundet et tilsvarende aktivt interface i de uploadede configs.
