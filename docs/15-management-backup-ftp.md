# 15 - Management og FTP/TFTP backup

Dette dokument beskriver, hvordan P-routere og SW-CE-switches kan få management-reachability til en backupserver, så running-configs kan kopieres ud.

## Formål

Målet er at kunne sende configs fra provider- og switchlaget til en central FTP/TFTP-server uden at blande management-trafik unødigt ind i kundens CE/BGP-routing.

Eksisterende CE-routere bruger allerede denne arkiv-template:

```text
archive
 path tftp://10.50.170.19/$h-running-config.cfg
 write-memory
 time-period 1440
```

Hvis backupserveren er en FTP-server i stedet for TFTP, skal `path` ændres til FTP-format med username/password.

## Vigtig designregel

Management og transport bør holdes adskilt:

- P-routere skal ikke lære kundens CE/BGP-ruter.
- SW-CE skal ikke bruges som transit for kundetrafik.
- Backup-adgang bør ske via management-IP, management-VLAN eller en dedikeret management-router.
- NAT til backupserver/internet bør ske centralt, ikke på alle enheder.

## Status i aktuelle configs

SW-CE1 og SW-CE2 har allerede management SVI'er:

| Enhed | SVI | IP | Default gateway |
| --- | --- | --- | --- |
| SW-CE1 | Vlan10 | 10.10.10.10/24 | 10.10.10.1 |
| SW-CE2 | Vlan10 | 10.20.10.10/24 | 10.20.10.1 |

Men VLAN 10 er ikke med i de trunk allowed-lister, der er fundet i configs. Det betyder, at switchenes default-gateway ikke kan nås, før management-VLAN'et bliver transporteret eller gatewayen placeres lokalt.

## Anbefalet løsning A - Management via CE-CORE-R1

Dette er den simple løsning i labbet.

### Idé

- Brug en eller flere management VLANs.
- Lad CE-CORE-R1 være default gateway for management-subnets.
- Tillad management VLANs på relevante trunks.
- Tilføj management-subnets til NAT ACL, hvis backupserveren ligger uden for labbet.

### Eksempel: SW-CE1

Hvis SW-CE1 skal bruge `10.10.10.0/24`:

På CE-CORE-R1:

```text
interface GigabitEthernet0/0.10
 description MGMT-SW-CE1
 encapsulation dot1Q 10
 ip address 10.10.10.1 255.255.255.0
 ip nat inside
 ip virtual-reassembly in
```

På SW-CE1 trunks:

```text
interface GigabitEthernet0/0
 switchport trunk allowed vlan add 10
!
interface GigabitEthernet0/1
 switchport trunk allowed vlan add 10
```

På SW-CE1:

```text
interface Vlan10
 description MGMT-SW-CE1
 ip address 10.10.10.10 255.255.255.0
 no shutdown
!
ip default-gateway 10.10.10.1
```

### Problem med SW-CE2

SW-CE2 bruger også `interface Vlan10`, men IP-nettet er `10.20.10.0/24`. Hvis SW-CE1 og SW-CE2 ligger i samme Layer 2 VLAN 10-domain, bør de ikke bruge forskellige subnets på samme VLAN.

Bedre standard:

| Enhed | VLAN | Net | Gateway |
| --- | --- | --- | --- |
| SW-CE1 | 10 | 10.10.10.0/24 | 10.10.10.1 |
| SW-CE2 | 20 | 10.20.10.0/24 | 10.20.10.1 |

Alternativt kan begge bruge samme management-subnet, fx VLAN 10 / `10.10.10.0/24`.

## Anbefalet løsning B - Dedikeret out-of-band management

Dette er den reneste løsning:

```text
Backupserver / mgmt LAN
        |
Management router / firewall
        |
Mgmt switch
   |      |      |
 P1     P2    SW-CE
```

Fordele:

- Management er adskilt fra lab-routing.
- P-routere og switches kan nå backupserver uden at påvirke CE/BGP/MPLS designet.
- Lettere at fejlfinde og mere realistisk.

## P-routere

P1 og P2 har ikke management interface/SVI i de aktuelle configs. Der er to realistiske muligheder:

### Mulighed 1 - Brug Loopback0 som backup-source

Hvis P1/P2 kan route til backupserveren via provider core, kan de bruge Loopback0 som source:

```text
ip tftp source-interface Loopback0
ip ftp source-interface Loopback0
```

Derefter skal der være route til backupserveren, fx via ISP-R1/management-router.

### Mulighed 2 - Tilføj management interface

Hvis der findes et ledigt interface til management:

```text
interface GigabitEthernet0/3
 description MGMT-to-backup-network
 ip address <mgmt-ip> <mask>
 no shutdown
!
ip route 0.0.0.0 0.0.0.0 <mgmt-gateway>
```

Denne løsning er mest tydelig i et lab, men den kræver et ekstra mgmt-net.

## Backup med TFTP

Cisco archive-template med TFTP:

```text
archive
 path tftp://10.50.170.19/$h-running-config.cfg
 write-memory
 time-period 1440
```

Manuel kopi:

```bash
copy running-config tftp:
```

## Backup med FTP

FTP kræver credentials. Undgå at gemme rigtige passwords i offentlige repos.

```text
ip ftp username <username>
ip ftp password <password>
!
archive
 path ftp://<username>:<password>@10.50.170.19/$h-running-config.cfg
 write-memory
 time-period 1440
```

Manuel kopi:

```bash
copy running-config ftp:
```

## NAT ACL hvis backupserver ligger uden for labbet

Hvis management-nettene skal ud via CE-CORE-R1 NAT, skal de tilføjes til `NAT-SITES`:

```text
ip access-list standard NAT-SITES
 permit 10.10.10.0 0.0.0.255
 permit 10.20.10.0 0.0.0.255
```

Hvis backupserveren er direkte routet i management-nettet, er NAT ikke nødvendig.

## Verifikation

På SW-CE:

```bash
show ip interface brief
show vlan brief
show interfaces trunk
ping 10.10.10.1
ping 10.50.170.19
copy running-config tftp:
```

På P-routere:

```bash
show ip route 10.50.170.19
ping 10.50.170.19 source Loopback0
copy running-config tftp:
```

På CE-CORE-R1:

```bash
show ip route 10.10.10.0
show ip route 10.20.10.0
show access-lists NAT-SITES
show ip nat translations
```

## Anbefalet næste ændring i labbet

For denne lab anbefales:

1. Brug TFTP først, fordi CE-routerne allerede bruger `tftp://10.50.170.19`.
2. Lav management VLAN til SW-CE1/SW-CE2.
3. Tilføj trunk allowed VLAN for management VLANs.
4. Tilføj gateway på CE-CORE-R1 eller dedikeret management-router.
5. Tilføj archive-template på P1, P2, SW-CE1 og SW-CE2.
6. Skift først til FTP/SCP senere, når reachability virker.
