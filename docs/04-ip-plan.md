# 04 - IP Plan

Dette dokument samler IP-planen for H6-Net.

## Router loopbacks

| Device | Loopback | Formål |
| --- | --- | --- |
| CE-CORE-R1 | 11.11.11.11/32 | CORE router-id og test source |
| CE-AAH-R1 | 22.22.22.22/32 | AAH CE router-id og test source |
| CE-KBH-R1 | 33.33.33.33/32 | KBH CE1 router-id og test source |
| CE-ODE-R1 | 44.44.44.44/32 | ODE CE1 router-id og test source |

## WAN /30 networks

| Site | Primær WAN | Sekundær WAN | Note |
| --- | --- | --- | --- |
| CORE-AAH | 10.10.0.0/30 | 10.11.0.0/30 | CORE til AAH |
| CORE-KBH | 10.20.0.0/30 | 10.21.0.0/30 | CORE til KBH |
| CORE-ODE | 10.30.0.0/30 | 10.31.0.0/30 | CORE til ODE |

## Principper

- /32 loopbacks bruges til router-id, test og BGP network statements
- /30 bruges til point-to-point WAN mellem CORE og sites
- Nye CE2-routere skal have egne loopbacks
- CE2 loopbacks skal huskes i NAT ACL på CORE, hvis de skal kunne bruge internet breakout

## Mangler at udfylde

- P-router loopbacks
- PE-router loopbacks
- MPLS core linknets
- Nye CE2 loopbacks for alle sites
