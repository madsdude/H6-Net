# 11 - Failover design

Dette dokument beskriver failover-princippet i H6-Net.

## Mål

Hvert site har to CE-routere:

- CE1 = primær path
- CE2 = sekundær/backup path

CE-CORE-R1 skal foretrække CE1 for site-ruter, men kunne bruge CE2 hvis CE1 eller primær EPL/BGP path fejler.

## Path selection fra CORE mod site

CE-CORE-R1 modtager site-loopbacks fra både CE1 og CE2. Route-maps sætter local preference på indgående routes:

| Neighbor-type | Route-map | Local preference | Resultat |
| --- | --- | --- | --- |
| CE1 primary | SITE-PRIMARY-IN | 200 | Foretrukket |
| CE2 secondary | SITE-SECONDARY-IN | 100 | Backup |

## Path selection fra site mod CORE/internet

CE-CORE-R1 annoncerer default route til både CE1 og CE2.

| Neighbor-type | Route-map | Metric | Resultat |
| --- | --- | --- | --- |
| CE1 primary | DEFAULT-PRIMARY | 0 | Foretrukket default |
| CE2 secondary | DEFAULT-SECONDARY | 200 | Backup default |

## iBGP mellem CE1 og CE2

CE1 og CE2 på samme site har iBGP over et lokalt /30 interconnect. Det giver mulighed for, at den ene CE kan lære routes via den anden.

| Site | iBGP link |
| --- | --- |
| AAH | 10.10.0.9 <-> 10.10.0.10 |
| KBH | 10.20.0.9 <-> 10.20.0.10 |
| ODE | 10.30.0.9 <-> 10.30.0.10 |

## Failover-test

### Test 1 - Luk primær CE-link

På site CE1 eller CORE subinterface til CE1:

```bash
conf t
interface <primary-interface>
 shutdown
end
```

Forventet:

- BGP neighbor til CE1 går down.
- CE-CORE-R1 vælger route via CE2.
- Site beholder default route via CE2.
- Ping fra site loopback mod internet/CORE virker fortsat, hvis NAT ACL og routing er korrekt.

### Test 2 - Luk sekundær CE-link

```bash
conf t
interface <secondary-interface>
 shutdown
end
```

Forventet:

- Trafik fortsætter via CE1.
- CE2 BGP går down.
- Ingen påvirkning på primær path.

### Test 3 - Luk provider path mod P1/P2

På ISP-R1/P1/P2:

```bash
conf t
interface <core-link>
 shutdown
end
```

Forventet:

- BFD trigger OSPF hurtigere.
- MPLS/LDP path ændres, hvis alternativ path findes.
- Xconnect bør blive oppe, hvis remote PE stadig kan nås via anden provider path.

## Verification commands

På CE-CORE-R1:

```bash
show ip bgp summary
show ip bgp
show ip route bgp
show ip route <site-loopback>
show ip cef <site-loopback>
```

På site CE:

```bash
show ip bgp summary
show ip route 0.0.0.0
show ip route bgp
ping 11.11.11.11 source Loopback0
ping 8.8.8.8 source Loopback0
```

På provider:

```bash
show ip ospf neighbor
show bfd neighbors
show mpls ldp neighbor
show xconnect all
```

## Typiske fejl

| Symptom | Mulig årsag |
| --- | --- |
| Site mister default route ved CE1-fail | CE2 BGP neighbor down eller iBGP mangler |
| CORE vælger CE2 selvom CE1 er oppe | Local preference/route-map mangler eller route modtages ikke fra CE1 |
| Internet virker fra CE1 men ikke CE2 | CE2 loopback eller CE2-link mangler i NAT ACL |
| Xconnect falder ned ved core-link fail | MPLS/LDP/OSPF path mangler via anden P-router |
