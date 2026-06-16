# 14 - Config review og cleanup

Dette dokument er en teknisk gennemgang af configs i `configs/Today-configs` med fokus på standardisering og oprydning.

## Læste configs

| Fil | Hostname i config | Rolle |
| --- | --- | --- |
| configs/Today-configs/CE-CORE-R1 | CE-CORE-R1 | Central CE / NAT / BGP AS65000 |
| configs/Today-configs/CE-AAH-R1 | CE-AAH-R1 | AAH CE1 |
| configs/Today-configs/CE-AAH-R2 | CE-AAH-R2 | AAH CE2 |
| configs/Today-configs/CE-KBH-R1 | CE-KBH-R1 | KBH CE1 |
| configs/Today-configs/CE-KBH-R2 | CE-KBH-R2 | KBH CE2 |
| configs/Today-configs/CE-ODE-R1 | CE-ODE-R1 | ODE CE1 |
| configs/Today-configs/CE-ODE-R2 | CE-ODE-R2 | ODE CE2 |
| configs/Today-configs/ISP-PE1 | ISP-R1 | PE1 / xconnect endpoint |
| configs/Today-configs/ISP-PE2 | ISP-R2 | PE2 / xconnect endpoint |
| configs/Today-configs/P1 | P1 | Provider core + ISP NAT breakout |
| configs/Today-configs/P2 | P2 | Provider core |
| configs/Today-configs/SW-CE1 | SW-CE1 | CE-facing L2 switch / management pending |
| configs/Today-configs/SW-CE2 | SW-CE2 | CE-facing L2 switch / management pending |

## Kritiske fund

### 1. CE-CORE-R1 mangler OSPF network for AAH CE2-link

CE-CORE-R1 har aktivt interface:

```text
interface GigabitEthernet0/0.102
 description AAH-CE2-SECONDARY
 encapsulation dot1Q 102
 ip address 10.10.0.5 255.255.255.252
```

Men OSPF 100 mangler:

```text
network 10.10.0.5 0.0.0.0 area 0
```

Hvis OSPF stadig bruges under migration, bør dette tilføjes.

### 2. AAH VLAN 101/102 skal verificeres i xconnect-laget

CE-CORE-R1 bruger VLAN 101 og 102 til AAH i den nye plan. ISP-R1 config viste tidligere xconnect for 100 og 110, men ikke 101 og 102.

Tjek:

```bash
show xconnect all | include 101|102
show run interface GigabitEthernet0/1.101
show run interface GigabitEthernet0/1.102
```

Hvis de ikke findes, skal de oprettes på relevant PE-side.

### 3. PE2 / remote xconnect endpoint er nu dokumenteret

Remote PE er ikke længere et manglende designpunkt, fordi `ISP-PE2` findes i configs.

| Enhed | Hostname | Loopback | Funktion |
| --- | --- | --- | --- |
| ISP-PE1 | ISP-R1 | 1.1.1.1/32 | PE1 / xconnect endpoint |
| ISP-PE2 | ISP-R2 | 4.4.4.4/32 | PE2 / xconnect endpoint |

Cleanup er nu primært navnestandard: filnavn og hostname bør gøres ens.

### 4. P1 er nu ISP NAT breakout

P1 er ændret til ISP-sidens NAT breakout med DHCP på Gi0/2, NAT outside på Gi0/2, NAT inside på provider links og default route annonceret i OSPF.

Aktuel lab-model:

```text
Gi0/0 -> PE1/provider core -> ip nat inside
Gi0/1 -> PE2/provider core -> ip nat inside
Gi0/2 -> DHCP/internet     -> ip nat outside
OSPF 10 -> default-information originate
```

Dette er en god adskillelse fra CE-CORE-R1, fordi kundens site-internet og ISP-management ikke blandes.

Dog bruger P1 midlertidigt:

```text
access-list 1 permit any
```

Det virker til test, men bør strammes op til en dedikeret `NAT-ISP-MGMT` ACL.

### 5. SW-CE mangler stadig management gateway/reachability

SW-CE1 og SW-CE2 har management SVI'er, men de skal have gateway på PE-siden, ikke CE-CORE-R1.

| Enhed | Management IP | Gateway der bør laves |
| --- | --- | --- |
| SW-CE1 | 10.10.10.10/24 | 10.10.10.1 på PE1 |
| SW-CE2 | 10.20.10.10/24 | 10.20.10.1 på PE2 |

Der skal også sikres, at VLAN 10 er tilladt på relevante trunks/access-links.

### 6. P1/P2 har OSPF network for 10.0.23.0/30 uden fundet interface

P1 og P2 har:

```text
network 10.0.23.0 0.0.0.3 area 0
```

Men configs viser ikke et aktivt interface i `10.0.23.0/30`. Enten mangler der et P1-P2 link i config/topologi, eller også skal network statement fjernes.

## Standardisering

### Hostname/filnavn

| Fil | Problem | Anbefaling |
| --- | --- | --- |
| ISP-PE1 | Hostname er ISP-R1 | Omdøb fil til ISP-R1 eller hostname til ISP-PE1 |
| ISP-PE2 | Hostname er ISP-R2 | Omdøb fil til ISP-R2 eller hostname til ISP-PE2 |

### BGP timers

Timers er ikke konsekvent sat på alle eBGP neighbors.

Anbefaling:

```text
neighbor <peer> timers 5 15
```

bruges ens på alle eBGP peers, hvis labbet skal have hurtigere failover.

### LDP router-id

P1 og P2 har:

```text
mpls ldp router-id Loopback0 force
```

PE-routerne bør også have det for standardisering.

### NAT ACL på P1

Anbefaling efter SW-CE-test:

```text
ip access-list standard NAT-ISP-MGMT
 permit 1.1.1.1
 permit 2.2.2.2
 permit 3.3.3.3
 permit 4.4.4.4
 permit 10.0.12.0 0.0.0.3
 permit 10.0.13.0 0.0.0.3
 permit 10.0.24.0 0.0.0.3
 permit 10.0.34.0 0.0.0.3
 permit 10.10.10.0 0.0.0.255
 permit 10.20.10.0 0.0.0.255
```

### Legacy subinterfaces

CE-CORE-R1 har disse shutdown subinterfaces uden IP:

```text
Gi0/0.100
Gi0/0.110
Gi0/0.200
Gi0/0.210
Gi0/0.300
Gi0/0.310
```

Anbefaling: behold dem kun så længe de er relevante for migrationshistorik. Fjern dem, når ny plan er verificeret.

## Oprydningscheckliste

| Punkt | Prioritet | Handling |
| --- | --- | --- |
| Tilføj/verificer OSPF for AAH CE2 `10.10.0.5` | Høj | Ret CE-CORE-R1 eller beslut at OSPF ikke skal bruges |
| Verificer AAH VLAN 101/102 xconnect | Høj | Tjek ISP-R1/ISP-R2/SW-DIST trunks |
| Færdiggør SW-CE management gateway | Høj | Lav PE1/PE2 subinterfaces og OSPF networks |
| Stram P1 NAT ACL | Middel | Erstat `access-list 1 permit any` med `NAT-ISP-MGMT` |
| Standardisér BGP timers | Middel | Brug samme timer-policy på alle peers |
| Standardisér LDP router-id på PE-routere | Middel | Tilføj `mpls ldp router-id Loopback0 force` |
| Afklar `10.0.23.0/30` i OSPF | Middel | Fjern eller dokumentér P1-P2 link |
| Fjern legacy VLAN/subinterfaces | Lav | Efter migration og test |
| Omdøb ISP-PE1/ISP-R1 og ISP-PE2/ISP-R2 | Lav | Gør filnavn og hostname ens |

## Foreslåede rettelser

### CE-CORE-R1 hvis OSPF stadig bruges

```text
conf t
router ospf 100
 network 10.10.0.5 0.0.0.0 area 0
end
write memory
```

### PE-router LDP standardisering

```text
conf t
mpls ldp router-id Loopback0 force
end
write memory
```

### Eksempel AAH xconnect hvis VLAN 101/102 mangler på PE1

```text
conf t
interface GigabitEthernet0/1.101
 description AAH-CE1-PRIMARY-VC101
 encapsulation dot1Q 101
 xconnect 4.4.4.4 101 encapsulation mpls
!
interface GigabitEthernet0/1.102
 description AAH-CE2-SECONDARY-VC102
 encapsulation dot1Q 102
 xconnect 4.4.4.4 102 encapsulation mpls
end
write memory
```

Dette skal kun laves, hvis switch-trunks og PE2 også er klar til VLAN/VC 101 og 102.

### Eksempel SW-CE management gateway på PE1

```text
conf t
interface GigabitEthernet0/1.10
 description ISP-MGMT-to-SW-CE1
 encapsulation dot1Q 10
 ip address 10.10.10.1 255.255.255.0
 no shutdown
!
router ospf 10
 network 10.10.10.0 0.0.0.255 area 0
end
write memory
```

### Eksempel SW-CE management gateway på PE2

```text
conf t
interface GigabitEthernet0/1.10
 description ISP-MGMT-to-SW-CE2
 encapsulation dot1Q 10
 ip address 10.20.10.1 255.255.255.0
 no shutdown
!
router ospf 10
 network 10.20.10.0 0.0.0.255 area 0
end
write memory
```
