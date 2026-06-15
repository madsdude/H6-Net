# H6-Net

H6-Net er et netværkslab, der dokumenterer et ISP-/enterprise-lignende WAN-design med MPLS transport, EPL/xconnect, CE-routing, BGP og central internet breakout.

## Formål

Formålet med H6-Net er at bygge og dokumentere et realistisk netværksdesign, hvor provider-laget leverer transport, og CE-laget bruger denne transport til routing mellem sites og internetadgang.

Labbet indeholder:

- P-routere i provider core
- PE-routere i provider edge
- MPLS/LDP transport
- OSPF i provider core
- EPL/xconnect services
- CE-routere på sites
- eBGP mellem CORE og sites
- Primær og sekundær WAN-forbindelse
- Central NAT og internet breakout
- Failover-test
- Standardiserede configs
- Dokumenteret IP-plan, VLAN-plan og AS-plan

## Designlag

| Lag | Funktion |
| --- | --- |
| P Core | Transport i provider-netværket med OSPF, MPLS og LDP |
| PE Edge | Terminerer EPL/xconnect services mod CE-/switch-laget |
| EPL Layer 2 | Leverer transparente Layer 2-forbindelser mellem CE-sites |
| CE Routing | BGP, default route, site-routing og midlertidig OSPF under migration |
| CORE Breakout | NAT og central internet breakout |

## Hovedprincip

Provider-netværket transporterer Layer 2 services via MPLS/xconnect. CE-routerne bygger Layer 3 routing ovenpå EPL-forbindelserne. CORE-routeren fungerer som central internet breakout for sites.

```text
Site CE
  ↓
EPL / xconnect
  ↓
PE router
  ↓
MPLS / LDP / OSPF provider core
  ↓
PE router
  ↓
CORE CE-router
  ↓
NAT / Internet breakout
```

## Dokumentation

| Fil | Beskrivelse |
| --- | --- |
| docs/01-overview.md | Overblik over hele H6-Net |
| docs/02-topology.md | Fysisk og logisk topologi |
| docs/03-device-roles.md | Roller for P, PE, CE og switches |
| docs/04-ip-plan.md | IP-plan |
| docs/05-vlan-plan.md | VLAN-plan |
| docs/06-isp-core-p-pe-design.md | ISP core design |
| docs/07-mpls-ldp-design.md | MPLS og LDP design |
| docs/08-epl-xconnect-design.md | EPL og xconnect design |
| docs/09-ce-core-bgp-design.md | BGP mellem CORE og sites |
| docs/10-nat-internet-breakout.md | NAT og internet breakout |
| docs/11-failover-design.md | Failover design |
| docs/12-migration-plan.md | Migration fra OSPF til BGP |
| docs/13-troubleshooting.md | Fejlfinding og show-kommandoer |

## Status

Dette repo er under opbygning.

> Note: Repoet bør holdes privat, hvis der senere uploades rigtige running-configs, passwords, SNMP communities, TACACS/RADIUS keys eller anden følsom information.
