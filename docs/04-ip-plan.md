# 04 - IP Plan

Dette dokument samler IP-planen for H6-Net.

> Status: Korrigeret efter to-CE design. Endelig verifikation skal altid ske mod aktuelle running-configs.

## CE Loopbacks

| Device | Loopback | Formål |
| --- | --- | --- |
| CE-CORE-R1 | 11.11.11.11/32 | CORE router-id og test source |
| CE-AAH-R1 | 22.22.22.22/32 | AAH CE1 router-id og test source |
| CE-KBH-R1 | 33.33.33.33/32 | KBH CE1 router-id og test source |
| CE-ODE-R1 | 44.44.44.44/32 | ODE CE1 router-id og test source |
| CE-AAH-R2 | TBD | AAH CE2 router-id og test source |
| CE-KBH-R2 | TBD | KBH CE2 router-id og test source |
| CE-ODE-R2 | TBD | ODE CE2 router-id og test source |

## CE to CORE WAN /30 Networks

Hvert site har sin egen site-blok. CE1 bruger første /30, og CE2 bruger næste /30 i samme site-blok.

### AAH - AS 65010

| Link | Network | CORE IP | Site IP |
| --- | --- | --- | --- |
| CE-CORE-R1 to CE-AAH-R1 | 10.10.0.0/30 | 10.10.0.1 | 10.10.0.2 |
| CE-CORE-R1 to CE-AAH-R2 | 10.10.0.4/30 | 10.10.0.5 | 10.10.0.6 |

### KBH - AS 65020

| Link | Network | CORE IP | Site IP |
| --- | --- | --- | --- |
| CE-CORE-R1 to CE-KBH-R1 | 10.20.0.0/30 | 10.20.0.1 | 10.20.0.2 |
| CE-CORE-R1 to CE-KBH-R2 | 10.20.0.4/30 | 10.20.0.5 | 10.20.0.6 |

### ODE - AS 65030

| Link | Network | CORE IP | Site IP |
| --- | --- | --- | --- |
| CE-CORE-R1 to CE-ODE-R1 | 10.30.0.0/30 | 10.30.0.1 | 10.30.0.2 |
| CE-CORE-R1 to CE-ODE-R2 | 10.30.0.4/30 | 10.30.0.5 | 10.30.0.6 |

## /30 Calculation Example

A /30 subnet increments by 4 addresses.

| Subnet | Network | Host 1 | Host 2 | Broadcast |
| --- | --- | --- | --- | --- |
| 10.10.0.0/30 | 10.10.0.0 | 10.10.0.1 | 10.10.0.2 | 10.10.0.3 |
| 10.10.0.4/30 | 10.10.0.4 | 10.10.0.5 | 10.10.0.6 | 10.10.0.7 |

## BGP Neighbor Pattern on CORE

CORE should peer with the site-side IPs.

| Site | CE1 neighbor | CE2 neighbor |
| --- | --- | --- |
| AAH | 10.10.0.2 | 10.10.0.6 |
| KBH | 10.20.0.2 | 10.20.0.6 |
| ODE | 10.30.0.2 | 10.30.0.6 |

## Important Notes

- 10.11.0.0/30, 10.21.0.0/30 and 10.31.0.0/30 must not be documented as CE2 links in the current two-CE design unless configs prove they are still used as legacy links.
- CE2 loopbacks must be added to the CORE NAT ACL if CE2 loopbacks need internet breakout.
- OSPF network statements must match the actual active WAN subnets during migration.
- Running-configs are the source of truth before committing final implementation notes.

## P/PE and MPLS Core IPs

The following still needs to be documented from configs:

- P-router loopbacks
- PE-router loopbacks
- P-to-PE transit links
- PE-to-PE / MPLS transport links
- OSPF process and areas in provider core
- LDP router-ids and active MPLS interfaces
