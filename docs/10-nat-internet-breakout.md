# 10 - NAT og internet breakout

Dette dokument beskriver central internet breakout på CE-CORE-R1.

## Formål

Alle sites skal kunne bruge CE-CORE-R1 som central internetudgang. Site-routerne modtager default route fra CE-CORE-R1 via BGP, og CE-CORE-R1 laver NAT overload ud af `GigabitEthernet0/1`.

## NAT interfaces

På CE-CORE-R1:

| Interface | Rolle | NAT |
| --- | --- | --- |
| Gi0/0.101 | AAH CE1 primary | inside |
| Gi0/0.102 | AAH CE2 secondary | inside |
| Gi0/0.201 | KBH CE1 primary | inside |
| Gi0/0.202 | KBH CE2 secondary | inside |
| Gi0/0.301 | ODE CE1 primary | inside |
| Gi0/0.302 | ODE CE2 secondary | inside |
| Gi0/1 | Upstream/internet | outside |

## NAT overload

CE-CORE-R1 bruger:

```text
ip nat inside source list NAT-SITES interface GigabitEthernet0/1 overload
```

Det betyder, at trafik fra ACL `NAT-SITES` PAT/NAT'es til adressen på `GigabitEthernet0/1`.

## NAT ACL

ACL'en indeholder:

- Site loopbacks for CE1 og CE2
- Nye WAN-/interconnect-net
- Midlertidige gamle sekundære links under migration

Aktuelle loopbacks:

| Site | CE1 loopback | CE2 loopback |
| --- | --- | --- |
| AAH | 22.22.22.22 | 22.22.22.23 |
| KBH | 33.33.33.33 | 33.33.33.34 |
| ODE | 44.44.44.44 | 44.44.44.45 |

Nye WAN-/interconnect-net:

| Site | CORE-CE1 | CORE-CE2 | CE1-CE2 interconnect |
| --- | --- | --- | --- |
| AAH | 10.10.0.0/30 | 10.10.0.4/30 | 10.10.0.8/30 |
| KBH | 10.20.0.0/30 | 10.20.0.4/30 | 10.20.0.8/30 |
| ODE | 10.30.0.0/30 | 10.30.0.4/30 | 10.30.0.8/30 |

Legacy/migration:

| Site | Net |
| --- | --- |
| KBH | 10.21.0.0/30 |
| ODE | 10.31.0.0/30 |

AAH legacy `10.11.0.0/30` findes på CE-AAH-R1, men er ikke fundet i CE-CORE-R1 NAT ACL. Hvis linket stadig skal testes med internet breakout, skal det tilføjes. Hvis linket er udfaset, bør det dokumenteres som fjernet.

## Default route

CE-CORE-R1 har default route:

```text
ip route 0.0.0.0 0.0.0.0 10.50.170.1
```

CE-CORE-R1 annoncerer default route til CE-sites via BGP `default-originate`.

## Test

Fra en site CE:

```bash
show ip route 0.0.0.0
ping 11.11.11.11 source Loopback0
ping 8.8.8.8 source Loopback0
traceroute 8.8.8.8 source Loopback0
```

På CE-CORE-R1:

```bash
show ip nat translations
show ip nat statistics
show access-lists NAT-SITES
show ip route 0.0.0.0
```

## Standard for nye sites

Når et nyt site tilføjes, skal NAT ACL opdateres med:

```text
permit <CE1-loopback>
permit <CE2-loopback>
permit <CORE-CE1 /30 wildcard>
permit <CORE-CE2 /30 wildcard>
permit <CE1-CE2 interconnect /30 wildcard>
```

Husk at tilføje begge CE-loopbacks. Ellers kan CE2 teste routing, men ikke internet/NAT korrekt.
