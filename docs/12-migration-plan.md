# 12 - Migration fra OSPF/legacy til BGP to-CE design

Dette dokument beskriver migrationen fra ældre EPL/OSPF-baserede links til den nye to-CE BGP-plan.

## Målbillede

Målet er, at hvert site har:

- CE1 primary mod CORE
- CE2 secondary mod CORE
- iBGP mellem CE1 og CE2
- eBGP mellem hver CE og CE-CORE-R1
- Central default route fra CE-CORE-R1
- NAT breakout på CE-CORE-R1
- OSPF 100 fjernet eller reduceret, når BGP er stabilt

## Nuværende migrationsstatus

I de aktuelle configs findes både gammel og ny struktur:

| Område | Status |
| --- | --- |
| CE-CORE nye subinterfaces 101/102/201/202/301/302 | Aktivt konfigureret |
| CE-CORE gamle subinterfaces 100/110/200/210/300/310 | Shutdown og uden IP |
| BGP mellem CORE og sites | Konfigureret |
| iBGP mellem CE1 og CE2 | Konfigureret |
| OSPF 100 på CE-laget | Stadig aktiv under migration |
| NAT ACL | Opdateret med nye site loopbacks og nye /30-net |

## Vigtig fund: AAH OSPF mismatch

CE-CORE-R1 har aktivt subinterface `Gi0/0.102` med IP `10.10.0.5/30` til AAH CE2, men OSPF 100 på CE-CORE-R1 har ikke network statement for `10.10.0.5`.

Hvis OSPF stadig skal bruges under migration, bør dette tilføjes:

```text
router ospf 100
 network 10.10.0.5 0.0.0.0 area 0
```

Alternativt skal OSPF fjernes/ikke bruges for dette link, hvis BGP er den eneste ønskede routing.

## Migrationstrin pr. site

### Trin 1 - Verificer Layer 2/EPL

På PE:

```bash
show xconnect all
show mpls l2transport vc
```

På CE:

```bash
show ip interface brief
ping <CORE-link-IP>
```

### Trin 2 - Verificer BGP

På CE-CORE-R1:

```bash
show ip bgp summary
show ip bgp
```

På site CE:

```bash
show ip bgp summary
show ip route 0.0.0.0
```

### Trin 3 - Verificer route selection

På CE-CORE-R1:

```bash
show ip route <CE1-loopback>
show ip route <CE2-loopback>
show ip bgp <CE1-loopback>
show ip bgp <CE2-loopback>
```

Primær route skal have bedre local preference end sekundær route.

### Trin 4 - Verificer NAT/internet

Fra begge CE-routere:

```bash
ping 8.8.8.8 source Loopback0
traceroute 8.8.8.8 source Loopback0
```

På CE-CORE-R1:

```bash
show ip nat translations
show access-lists NAT-SITES
```

### Trin 5 - Fjern legacy

Når BGP og NAT er verificeret:

- Fjern gamle shutdown subinterfaces på CE-CORE-R1.
- Fjern gamle VLAN/xconnect subinterfaces på PE, hvis de ikke længere bruges.
- Fjern gamle VLANs fra trunks.
- Fjern gamle OSPF network statements.
- Fjern legacy WAN IP'er fra CE1, hvis de ikke bruges.

## Standard for nye sites

Brug denne struktur:

| Element | Standard |
| --- | --- |
| CE1 loopback | `<site-loopback>.x` |
| CE2 loopback | `<site-loopback>.x+1` |
| CORE-CE1 /30 | `<site-prefix>.0/30` |
| CORE-CE2 /30 | `<site-prefix>.4/30` |
| CE1-CE2 interconnect | `<site-prefix>.8/30` |
| CE1 BGP | eBGP til CORE + iBGP til CE2 |
| CE2 BGP | eBGP til CORE + iBGP til CE1 |
| NAT ACL | CE1 loopback, CE2 loopback og alle relevante /30-net |

## Rollback

Hvis migration fejler:

1. Reaktiver gammel EPL/subinterface, hvis den stadig er intakt.
2. Gendan OSPF network statements for det gamle link.
3. Fjern eller shutdown ny BGP neighbor midlertidigt.
4. Verificer route tilbage til gammel path.
5. Undersøg BGP/xconnect/NAT før nyt forsøg.
