# 04 - IP-plan

Dette dokument samler IP-planen for H6-Net baseret på `configs/Today-configs`.

## Router loopbacks

| Enhed | Loopback0 | Funktion |
| --- | --- | --- |
| CE-CORE-R1 | 11.11.11.11/32 | CORE router-id og BGP router-id |
| CE-AAH-R1 | 22.22.22.22/32 | AAH CE1 router-id og annonceret BGP network |
| CE-AAH-R2 | 22.22.22.23/32 | AAH CE2 router-id og annonceret BGP network |
| CE-KBH-R1 | 33.33.33.33/32 | KBH CE1 router-id og annonceret BGP network |
| CE-KBH-R2 | 33.33.33.34/32 | KBH CE2 router-id og annonceret BGP network |
| CE-ODE-R1 | 44.44.44.44/32 | ODE CE1 router-id og annonceret BGP network |
| CE-ODE-R2 | 44.44.44.45/32 | ODE CE2 router-id og annonceret BGP network |
| ISP-R1 | 1.1.1.1/32 | PE-side router-id |
| P1 | 2.2.2.2/32 | Provider core router-id |
| P2 | 3.3.3.3/32 | Provider core router-id |
| Remote PE | 4.4.4.4/32 | Xconnect peer set i ISP-R1 config; config mangler i Today-configs |

## Provider core IP-plan

| Link | Enhed/interface | IP | Net |
| --- | --- | --- | --- |
| ISP-R1 - P1 | ISP-R1 Gi0/0 | 10.0.12.1/30 | 10.0.12.0/30 |
| ISP-R1 - P1 | P1 Gi0/0 | 10.0.12.2/30 | 10.0.12.0/30 |
| ISP-R1 - P2 | ISP-R1 Gi0/2 | 10.0.13.1/30 | 10.0.13.0/30 |
| ISP-R1 - P2 | P2 Gi0/0 | 10.0.13.2/30 | 10.0.13.0/30 |
| P1 - Remote PE | P1 Gi0/1 | 10.0.24.1/30 | 10.0.24.0/30 |
| P2 - Remote PE | P2 Gi0/1 | 10.0.34.1/30 | 10.0.34.0/30 |

## CE-CORE til site CE IP-plan

| Site | Link | VLAN | CORE IP | Site CE IP | Net |
| --- | --- | --- | --- | --- | --- |
| AAH | CORE - CE1 primary | 101 | 10.10.0.1/30 | 10.10.0.2/30 | 10.10.0.0/30 |
| AAH | CORE - CE2 secondary | 102 | 10.10.0.5/30 | 10.10.0.6/30 | 10.10.0.4/30 |
| KBH | CORE - CE1 primary | 201 | 10.20.0.1/30 | 10.20.0.2/30 | 10.20.0.0/30 |
| KBH | CORE - CE2 secondary | 202 | 10.20.0.5/30 | 10.20.0.6/30 | 10.20.0.4/30 |
| ODE | CORE - CE1 primary | 301 | 10.30.0.1/30 | 10.30.0.2/30 | 10.30.0.0/30 |
| ODE | CORE - CE2 secondary | 302 | 10.30.0.5/30 | 10.30.0.6/30 | 10.30.0.4/30 |

## CE1 til CE2 interconnect

| Site | CE1 IP | CE2 IP | Net | Formål |
| --- | --- | --- | --- | --- |
| AAH | 10.10.0.9/30 | 10.10.0.10/30 | 10.10.0.8/30 | iBGP og lokal CE-redundans |
| KBH | 10.20.0.9/30 | 10.20.0.10/30 | 10.20.0.8/30 | iBGP og lokal CE-redundans |
| ODE | 10.30.0.9/30 | 10.30.0.10/30 | 10.30.0.8/30 | iBGP og lokal CE-redundans |

## Gamle/midlertidige sekundære WAN-links

Disse findes på CE1-routerne, men bruges ikke som primær del af den nye to-CE BGP-plan i de aktuelle configs.

| Site | Interface | IP | Net | Status |
| --- | --- | --- | --- | --- |
| AAH | CE-AAH-R1 Gi0/1 | 10.11.0.2/30 | 10.11.0.0/30 | Legacy/migration |
| KBH | CE-KBH-R1 Gi0/1 | 10.21.0.2/30 | 10.21.0.0/30 | Legacy/migration |
| ODE | CE-ODE-R1 Gi0/1 | 10.31.0.2/30 | 10.31.0.0/30 | Legacy/migration |

## Upstream/NAT outside

CE-CORE-R1 bruger `GigabitEthernet0/1` som NAT outside og får IP via DHCP. Default route peger mod `10.50.170.1`.
