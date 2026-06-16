# 15 - ISP management og config backup

Dette dokument beskriver, hvordan P-, PE- og SW-CE-enheder kan få management-reachability til en backupserver uden at bruge samme internet breakout som kundens CE-routere.

## Designbeslutning

ISP-udstyr og kunde-/site-CE skal holdes adskilt.

Følgende udstyr skal ikke bruge CE-CORE-R1 som internet-/backup-gateway:

- P-routere
- PE-routere
- SW-CE-switches
- ISP management-adgang

CE-CORE-R1 må gerne være internet breakout for kundens sites, men den bør ikke være management breakout for ISP-udstyret.

## Formål

Målet er at kunne sende configs fra provider- og switchlaget til en central backupserver, uden at management-trafik går igennem kundens CE/BGP/NAT-design.

Eksisterende CE-routere bruger allerede denne arkiv-template:

```text
archive
 path tftp://10.50.170.19/$h-running-config.cfg
 write-memory
 time-period 1440
```

Det kan fortsat bruges på CE-routere, men ISP-udstyr bør have en separat management path.

## Korrekt overordnet design

```text
                         Backupserver
                              |
                      ISP-MGMT router/firewall
                              |
                       ISP management switch
              /               |               \
           P-routere        PE-routere        SW-CE


                         Kundeside / sites
                              |
                          CE-CORE-R1
                              |
                       Site CE1 / CE2
```

Der er to adskilte domæner:

| Domæne | Bruges til | Gateway |
| --- | --- | --- |
| ISP-MGMT | Backup, SSH, drift og config-arkiv for P/PE/SW-CE | ISP-MGMT router/firewall |
| CUSTOMER/CE | Kundens site-routing og internet breakout | CE-CORE-R1 |

## Vigtige designregler

- P/PE/SW-CE skal ikke bruge CE-CORE-R1 som default gateway.
- P-routere skal ikke lære kundens CE/BGP-ruter.
- SW-CE skal ikke være transit for management via kundens CE-routing.
- ISP management skal have separat VLAN/subnet eller fysisk out-of-band interface.
- Backupserveren skal nås via ISP-MGMT, ikke via kundens NAT på CE-CORE-R1.
- Hvis backupserveren ligger på samme fysiske host-net som labbet, bør ISP-MGMT-routeren/firewallen have en kontrolleret route til den.

## Status i aktuelle configs

SW-CE1 og SW-CE2 har allerede management SVI'er:

| Enhed | SVI | IP | Default gateway |
| --- | --- | --- | --- |
| SW-CE1 | Vlan10 | 10.10.10.10/24 | 10.10.10.1 |
| SW-CE2 | Vlan10 | 10.20.10.10/24 | 10.20.10.1 |

Men VLAN 10 er ikke med i de trunk allowed-lister, der er fundet i configs. Det betyder, at switchenes default-gateway ikke kan nås, før management-VLAN'et bliver transporteret eller gatewayen placeres lokalt.

Da ISP og kunde skal holdes adskilt, bør disse SVI'er ikke gatewayes via CE-CORE-R1. De bør i stedet flyttes til et ISP-MGMT VLAN/subnet.

## Anbefalet løsning - separat ISP-MGMT net

Brug et separat management-net kun til providerudstyr.

Eksempel:

| Enhed | Management IP | Gateway |
| --- | --- | --- |
| ISP-MGMT-GW | 172.16.99.1/24 | Mod backupserver/host-net |
| ISP-R1 / PE | 172.16.99.10/24 | 172.16.99.1 |
| P1 | 172.16.99.21/24 | 172.16.99.1 |
| P2 | 172.16.99.22/24 | 172.16.99.1 |
| SW-CE1 | 172.16.99.31/24 | 172.16.99.1 |
| SW-CE2 | 172.16.99.32/24 | 172.16.99.1 |
| Backupserver | 10.50.170.19 eller 172.16.99.100 | Via ISP-MGMT |

## P/PE-router config-model

Hvis der findes et ledigt interface, bruges det som management-interface.

Eksempel P1:

```text
interface GigabitEthernet0/3
 description OOB-MGMT-to-ISP-MGMT-SW
 ip address 172.16.99.21 255.255.255.0
 no shutdown
!
ip route 10.50.170.19 255.255.255.255 172.16.99.1
ip tftp source-interface GigabitEthernet0/3
!
archive
 path tftp://10.50.170.19/$h-running-config.cfg
 write-memory
 time-period 1440
```

Eksempel P2:

```text
interface GigabitEthernet0/3
 description OOB-MGMT-to-ISP-MGMT-SW
 ip address 172.16.99.22 255.255.255.0
 no shutdown
!
ip route 10.50.170.19 255.255.255.255 172.16.99.1
ip tftp source-interface GigabitEthernet0/3
!
archive
 path tftp://10.50.170.19/$h-running-config.cfg
 write-memory
 time-period 1440
```

Eksempel ISP-R1/PE:

```text
interface GigabitEthernet0/3
 description OOB-MGMT-to-ISP-MGMT-SW
 ip address 172.16.99.10 255.255.255.0
 no shutdown
!
ip route 10.50.170.19 255.255.255.255 172.16.99.1
ip tftp source-interface GigabitEthernet0/3
!
archive
 path tftp://10.50.170.19/$h-running-config.cfg
 write-memory
 time-period 1440
```

Brug helst en specifik host route til backupserveren i stedet for en fuld default route. Det minimerer risikoen for, at P/PE begynder at bruge managementnettet som generel internetudgang.

## SW-CE config-model

På L2-switches bruges management-SVI og `ip default-gateway`.

Eksempel SW-CE1:

```text
vlan 999
 name ISP-MGMT
!
interface GigabitEthernet3/3
 description OOB-MGMT-to-ISP-MGMT-SW
 switchport mode access
 switchport access vlan 999
 spanning-tree portfast edge
!
interface Vlan999
 description ISP-MGMT-SW-CE1
 ip address 172.16.99.31 255.255.255.0
 no shutdown
!
interface Vlan10
 shutdown
!
ip default-gateway 172.16.99.1
!
archive
 path tftp://10.50.170.19/$h-running-config.cfg
 write-memory
 time-period 1440
```

Eksempel SW-CE2:

```text
vlan 999
 name ISP-MGMT
!
interface GigabitEthernet3/3
 description OOB-MGMT-to-ISP-MGMT-SW
 switchport mode access
 switchport access vlan 999
 spanning-tree portfast edge
!
interface Vlan999
 description ISP-MGMT-SW-CE2
 ip address 172.16.99.32 255.255.255.0
 no shutdown
!
interface Vlan10
 shutdown
!
ip default-gateway 172.16.99.1
!
archive
 path tftp://10.50.170.19/$h-running-config.cfg
 write-memory
 time-period 1440
```

## Backupmetode

Start med TFTP, fordi det allerede er brugt på CE-routerne. Når reachability virker, kan backupmetoden senere skiftes til en mere sikker metode som SCP.

Manuel test:

```bash
copy running-config tftp:
```

## ISP-MGMT gateway/router

ISP-MGMT gatewayen kan være:

- en lille router i labbet
- en firewall
- en Linux-router
- en dedikeret management-VM

Minimum:

```text
interface MGMT-LAN
 ip address 172.16.99.1/24

route til 10.50.170.19/32 eller direkte interface mod backupserver-net
```

Hvis backupserveren ligger i `10.50.170.0/24`, skal ISP-MGMT gatewayen kunne route dertil. CE-CORE-R1 skal ikke være transit for dette.

## Verifikation

På SW-CE:

```bash
show ip interface brief
show vlan brief
show interfaces status
ping 172.16.99.1
ping 10.50.170.19
copy running-config tftp:
```

På P/PE-routere:

```bash
show ip interface brief
show ip route 10.50.170.19
ping 172.16.99.1 source GigabitEthernet0/3
ping 10.50.170.19 source GigabitEthernet0/3
copy running-config tftp:
```

På ISP-MGMT gateway:

```bash
show ip route
ping 172.16.99.21
ping 172.16.99.22
ping 172.16.99.31
ping 172.16.99.32
ping 10.50.170.19
```

## Anbefalet næste ændring i labbet

1. Lav eller vælg en ISP-MGMT gateway/router/firewall.
2. Lav et dedikeret ISP-MGMT subnet, fx `172.16.99.0/24`.
3. Tilslut P1, P2, ISP-R1/PE og SW-CE til managementnettet via ledige interfaces.
4. Flyt SW-CE management fra Vlan10 til Vlan999 eller et andet dedikeret ISP-MGMT VLAN.
5. Tilføj TFTP archive-template på P1, P2, ISP-R1, SW-CE1 og SW-CE2.
6. Test `ping 10.50.170.19` fra management-source-interface.
7. Test `copy running-config tftp:`.

## Ikke anbefalet

Det anbefales ikke at bruge CE-CORE-R1 som gateway/NAT for ISP-udstyr. Det gør designet uklart, fordi kundens internet breakout og providerens managementplan bliver blandet sammen.
