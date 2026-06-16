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
| configs/Today-configs/ISP-PE1 | ISP-R1 | PE-side/xconnect |
| configs/Today-configs/P1 | P1 | Provider core |
| configs/Today-configs/P2 | P2 | Provider core |

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

### 2. AAH VLAN 101/102 mangler i fundet ISP-R1 xconnect config

CE-CORE-R1 bruger VLAN 101 og 102 til AAH i den nye plan. ISP-R1 config viser xconnect for 100 og 110, men ikke 101 og 102.

Tjek:

```bash
show xconnect all | include 101|102
show run interface GigabitEthernet0/1.101
show run interface GigabitEthernet0/1.102
```

Hvis de ikke findes, skal de oprettes på relevant PE-side.

### 3. Remote PE config mangler

ISP-R1 bruger xconnect peer:

```text
xconnect 4.4.4.4 <VC-ID> encapsulation mpls
```

Der er ikke fundet en config for remote PE `4.4.4.4` i `Today-configs`. Upload/dokumentér den, så xconnect-laget kan valideres fuldt.

### 4. P1/P2 har OSPF network for 10.0.23.0/30 uden fundet interface

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

ISP-R1 bør også have det for standardisering.

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
| Verificer AAH VLAN 101/102 xconnect | Høj | Tjek ISP-R1/remote PE/SW-DIST trunks |
| Upload remote PE config `4.4.4.4` | Høj | Nødvendig for fuld MPLS/xconnect dokumentation |
| Standardisér BGP timers | Middel | Brug samme timer-policy på alle peers |
| Standardisér LDP router-id på ISP-R1 | Middel | Tilføj `mpls ldp router-id Loopback0 force` |
| Afklar `10.0.23.0/30` i OSPF | Middel | Fjern eller dokumentér P1-P2 link |
| Fjern legacy VLAN/subinterfaces | Lav | Efter migration og test |
| Omdøb ISP-PE1/ISP-R1 | Lav | Gør filnavn og hostname ens |

## Foreslåede rettelser

### CE-CORE-R1 hvis OSPF stadig bruges

```text
conf t
router ospf 100
 network 10.10.0.5 0.0.0.0 area 0
end
write memory
```

### ISP-R1 LDP standardisering

```text
conf t
mpls ldp router-id Loopback0 force
end
write memory
```

### Eksempel AAH xconnect hvis VLAN 101/102 mangler på ISP-R1

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

Dette skal kun laves, hvis switch-trunks og remote PE også er klar til VLAN/VC 101 og 102.
