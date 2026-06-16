# 05 - VLAN-plan

Dette dokument beskriver VLAN-planen for EPL/xconnect og CE-CORE subinterfaces.

## Princip

Hvert EPL-link transporteres som et VLAN ind mod PE/xconnect-laget. På PE-siden mappes VLAN til samme VC-ID, så VLAN 201 eksempelvis transporteres som VC 201.

```text
CE / switch access
  -> VLAN
  -> PE subinterface dot1Q
  -> xconnect VC-ID
  -> remote PE
  -> CE-CORE-R1 subinterface
```

## Aktive nye VLANs

| VLAN | VC-ID | Site | Funktion | CORE subinterface |
| --- | --- | --- | --- | --- |
| 101 | 101 | AAH | CE1 primary | Gi0/0.101 |
| 102 | 102 | AAH | CE2 secondary | Gi0/0.102 |
| 201 | 201 | KBH | CE1 primary | Gi0/0.201 |
| 202 | 202 | KBH | CE2 secondary | Gi0/0.202 |
| 301 | 301 | ODE | CE1 primary | Gi0/0.301 |
| 302 | 302 | ODE | CE2 secondary | Gi0/0.302 |

## Legacy/migrations-VLANs

CE-CORE-R1 har stadig disse subinterfaces, men de er shutdown og uden IP i den aktuelle config:

| VLAN | Funktion/navn i config | Status |
| --- | --- | --- |
| 100 | AAH-PRIMARY | shutdown |
| 110 | AAH-SECONDARY | shutdown |
| 200 | KBH-PRIMARY | shutdown |
| 210 | KBH-SECONDARY | shutdown |
| 300 | ODE-PRIMARY | shutdown |
| 310 | ODE-SECONDARY | shutdown |

## VLANs på ISP-R1

ISP-R1 har subinterfaces for flere VLANs, blandt andet både gamle og nye VLANs:

```text
Gi0/1.100 -> VC 100
Gi0/1.110 -> VC 110
Gi0/1.200 -> VC 200
Gi0/1.201 -> VC 201
Gi0/1.202 -> VC 202
Gi0/1.210 -> VC 210
Gi0/1.300 -> VC 300
Gi0/1.301 -> VC 301
Gi0/1.302 -> VC 302
Gi0/1.310 -> VC 310
```

## Standard for nye sites

For et nyt site bør VLAN-planen følge samme mønster:

| Type | Standard |
| --- | --- |
| CE1 primary | `<site-prefix>01` |
| CE2 secondary | `<site-prefix>02` |
| Interconnect mellem CE1 og CE2 | Ikke transporteret over EPL; lokalt /30-net |

Eksempel for site med prefix `401/402`:

| Funktion | VLAN | CORE IP-net |
| --- | --- | --- |
| Site CE1 primary | 401 | 10.40.0.0/30 |
| Site CE2 secondary | 402 | 10.40.0.4/30 |
| CE1-CE2 interconnect | Lokalt | 10.40.0.8/30 |

## Cleanup-note

Når migrationen er færdig, bør gamle VLANs fjernes både på:

- CE-CORE-R1 subinterfaces
- PE/xconnect subinterfaces
- SW-DIST trunks
- SW-CE trunks/access-konfiguration
