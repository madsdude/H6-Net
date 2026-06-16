# 15 - ISP management, NAT og config backup

Dette dokument beskriver management- og backupdesignet for ISP-udstyret i H6-Net.

Målet er, at P-, PE- og SW-CE-enheder kan nå en backupserver uden at bruge kundens CE-CORE internet breakout.

## Designbeslutning

ISP-udstyr og kunde-/site-CE skal holdes adskilt.

Følgende udstyr skal bruge ISP-siden til management/backup:

- P-routere
- PE-routere
- SW-CE-switches
- ISP management-adgang

CE-CORE-R1 må gerne være internet breakout for kundens sites, men den skal ikke være management breakout for ISP-udstyret.

## Aktuel implementering

Efter seneste ændring er P1 blevet ISP-sidens internet/NAT breakout.

P1 bruger:

```text
interface GigabitEthernet0/0
 description CORE-TO-PE1
 ip address 10.0.12.2 255.255.255.252
 ip nat inside
 mpls ip
!
interface GigabitEthernet0/1
 description CORE-TO-PE2
 ip address 10.0.24.1 255.255.255.252
 ip nat inside
 mpls ip
!
interface GigabitEthernet0/2
 ip address dhcp
 ip nat outside
!
router ospf 10
 router-id 2.2.2.2
 default-information originate
!
ip nat inside source list 1 interface GigabitEthernet0/2 overload
access-list 1 permit any
```

Det betyder:

| Enhedstype | Status |
| --- | --- |
| P1 | Har direkte internet/NAT via Gi0/2 |
| P2 | Kan bruge default route fra P1 via OSPF |
| PE1 | Kan bruge default route fra P1 via OSPF |
| PE2 | Kan bruge default route fra P1 via OSPF |
| SW-CE1/SW-CE2 | Mangler stadig routed management-gateway på PE-siden |

## Logisk trafikflow

```text
P2 / PE1 / PE2
      |
      | OSPF default route
      v
     P1
      |
      | NAT overload
      v
 Gi0/2 DHCP / internet / host-net
      |
      v
 Backupserver 10.50.170.19
```

Når SW-CE også er færdiggjort:

```text
SW-CE1 Vlan10 -> PE1 mgmt subinterface -> OSPF -> P1 NAT -> backupserver
SW-CE2 Vlan10 -> PE2 mgmt subinterface -> OSPF -> P1 NAT -> backupserver
```

## Hvorfor SW-CE ikke virker endnu

SW-CE1 og SW-CE2 er Layer 2-switches. De bruger `ip default-gateway`, men de deltager ikke i OSPF.

Aktuel management-plan:

| Enhed | SVI | IP | Default gateway |
| --- | --- | --- | --- |
| SW-CE1 | Vlan10 | 10.10.10.10/24 | 10.10.10.1 |
| SW-CE2 | Vlan10 | 10.20.10.10/24 | 10.20.10.1 |

Derfor mangler der to ting:

1. VLAN 10 skal være tilladt på trunk/access-path mellem SW-CE og PE.
2. PE-routerne skal have Layer 3 gateway for SW-CE managementnettene.

## Anbefalet løsning for SW-CE

Brug PE-routerne som management gateway for SW-CE, ikke CE-CORE-R1.

| Management-net | Gateway | Placering |
| --- | --- | --- |
| 10.10.10.0/24 | 10.10.10.1 | PE1 / ISP-R1 |
| 10.20.10.0/24 | 10.20.10.1 | PE2 / ISP-R2 |

## PE1 config til SW-CE1

På PE1/ISP-R1 oprettes et L3 subinterface til SW-CE1 management:

```text
conf t
!
interface GigabitEthernet0/1.10
 description ISP-MGMT-to-SW-CE1
 encapsulation dot1Q 10
 ip address 10.10.10.1 255.255.255.0
 no shutdown
!
router ospf 10
 network 10.10.10.0 0.0.0.255 area 0
!
end
write memory
```

## SW-CE1 config

SW-CE1 skal have VLAN 10 tilladt mod PE1 og backup archive-template:

```text
conf t
!
vlan 10
 name ISP-MGMT-SW-CE1
!
interface GigabitEthernet0/0
 description TRUNK-TO-PE
 switchport trunk allowed vlan add 10
!
interface Vlan10
 description MGMT-SW-CE1
 ip address 10.10.10.10 255.255.255.0
 no shutdown
!
ip default-gateway 10.10.10.1
!
archive
 path tftp://10.50.170.19/$h-running-config.cfg
 write-memory
 time-period 1440
!
end
write memory
```

## PE2 config til SW-CE2

På PE2/ISP-R2 oprettes et L3 subinterface til SW-CE2 management:

```text
conf t
!
interface GigabitEthernet0/1.10
 description ISP-MGMT-to-SW-CE2
 encapsulation dot1Q 10
 ip address 10.20.10.1 255.255.255.0
 no shutdown
!
router ospf 10
 network 10.20.10.0 0.0.0.255 area 0
!
end
write memory
```

## SW-CE2 config

```text
conf t
!
vlan 10
 name ISP-MGMT-SW-CE2
!
interface GigabitEthernet0/0
 description TRUNK-TO-PE-or-UPLINK
 switchport trunk allowed vlan add 10
!
interface Vlan10
 description MGMT-SW-CE2
 ip address 10.20.10.10 255.255.255.0
 no shutdown
!
ip default-gateway 10.20.10.1
!
archive
 path tftp://10.50.170.19/$h-running-config.cfg
 write-memory
 time-period 1440
!
end
write memory
```

## NAT ACL cleanup på P1

Den nuværende P1-løsning bruger:

```text
access-list 1 permit any
```

Det er fint til hurtig lab-test, men det bør strammes op, når reachability er verificeret.

Anbefalet erstatning:

```text
conf t
!
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
!
no ip nat inside source list 1 interface GigabitEthernet0/2 overload
ip nat inside source list NAT-ISP-MGMT interface GigabitEthernet0/2 overload
!
no access-list 1
!
end
write memory
```

## Backupmetode

Start med TFTP, fordi det allerede bruges i labbet.

Archive-template:

```text
archive
 path tftp://10.50.170.19/$h-running-config.cfg
 write-memory
 time-period 1440
```

Manuel test:

```bash
copy running-config tftp:
```

## Verifikation

### P1

```bash
show ip interface brief
show ip route 0.0.0.0
show ip ospf neighbor
show ip nat translations
show access-lists
ping 10.50.170.19
```

### PE1

```bash
show ip route 0.0.0.0
show ip route 10.10.10.0
show ip ospf neighbor
ping 10.50.170.19
```

### PE2

```bash
show ip route 0.0.0.0
show ip route 10.20.10.0
show ip ospf neighbor
ping 10.50.170.19
```

### SW-CE1

```bash
show ip interface brief
show vlan brief
show interfaces trunk
ping 10.10.10.1
ping 10.50.170.19
copy running-config tftp:
```

### SW-CE2

```bash
show ip interface brief
show vlan brief
show interfaces trunk
ping 10.20.10.1
ping 10.50.170.19
copy running-config tftp:
```

### P1 routecheck efter SW-CE gateway er lavet

P1 skal lære SW-CE managementnettene via OSPF:

```bash
show ip route 10.10.10.0
show ip route 10.20.10.0
```

Forventet:

```text
10.10.10.0/24 via PE1
10.20.10.0/24 via PE2
```

## Ikke anbefalet

Det anbefales ikke at bruge CE-CORE-R1 som gateway/NAT for ISP-udstyr. Det gør designet uklart, fordi kundens internet breakout og providerens managementplan bliver blandet sammen.

## Næste konkrete skridt

1. Tilføj `Gi0/1.10` på PE1 med IP `10.10.10.1/24`.
2. Tilføj `Gi0/1.10` på PE2 med IP `10.20.10.1/24`.
3. Advertisér begge managementnet i OSPF 10.
4. Tilføj VLAN 10 på SW-CE trunks mod PE.
5. Test ping fra SW-CE til gateway og backupserver.
6. Test `copy running-config tftp:`.
7. Stram P1 NAT ACL fra `permit any` til `NAT-ISP-MGMT`.
