# 08 - EPL og xconnect design

Dette dokument beskriver EPL/xconnect-laget i H6-Net.

## Formål

EPL-laget leverer transparente Layer 2-forbindelser mellem CE-/switch-laget og CE-CORE-R1. Providerlaget transporterer VLANs som MPLS pseudowires.

CE-routerne ser forbindelsen som et almindeligt Ethernet-/WAN-link og bygger Layer 3 routing ovenpå.

## ISP-R1 xconnect mapping

ISP-R1 bruger `GigabitEthernet0/1` som trunk fra SW-DIST-1. Underinterfaces bruger dot1Q og xconnect mod remote peer `4.4.4.4`.

| VLAN | VC-ID | Beskrivelse i ISP-R1 | Status i design |
| --- | --- | --- | --- |
| 100 | 100 | EPL-CUSTOMER-A-VC100 | Legacy |
| 110 | 110 | AAH-SECONDARY-VC110 | Legacy |
| 200 | 200 | EPL-KBH-VC200 | Legacy |
| 201 | 201 | KBH-CE1-PRIMARY-VC201 | Aktiv ny plan |
| 202 | 202 | KBH-CE2-SECONDARY-VC202 | Aktiv ny plan |
| 210 | 210 | KBH-SECONDARY-VC210 | Legacy |
| 300 | 300 | EPL-ODE-VC300 | Legacy |
| 301 | 301 | ODE-CE1-PRIMARY-VC301 | Aktiv ny plan |
| 302 | 302 | ODE-CE2-SECONDARY-VC302 | Aktiv ny plan |
| 310 | 310 | ODE-SECONDARY-VC310 | Legacy |

AAH nye VLANs 101 og 102 findes på CE-CORE-R1, men de er ikke fundet i ISP-R1 xconnect-configen i `Today-configs`. Det skal verificeres på remote/anden PE eller opdateres, hvis AAH skal køre samme nye model som KBH og ODE.

## Standard xconnect config

Eksempel:

```text
interface GigabitEthernet0/1.<VLAN>
 description <SITE>-<CE>-<ROLE>-VC<VLAN>
 encapsulation dot1Q <VLAN>
 xconnect <remote-pe-loopback> <VC-ID> encapsulation mpls
```

For at holde designet enkelt bør VLAN og VC-ID være ens.

## Afhængigheder

For at en EPL/xconnect virker, skal alle disse lag være korrekte:

| Lag | Krav |
| --- | --- |
| Switch/access | VLAN skal være tilladt på trunks og korrekt som access VLAN mod CE |
| PE subinterface | Korrekt dot1Q VLAN |
| Xconnect | Samme VC-ID i begge ender |
| MPLS/LDP | PE loopbacks skal kunne nå hinanden via MPLS/LDP |
| CE Layer 3 | IP/BGP/OSPF på CE-linket skal matche |

## Verification commands

På PE:

```bash
show xconnect all
show mpls l2transport vc
show mpls forwarding-table
show interfaces trunk
show ip interface brief | include GigabitEthernet0/1
```

På CE:

```bash
show ip interface brief
show ip bgp summary
show ip route
ping <remote CE-link-IP>
```

## Cleanup

Når den nye to-CE-plan er færdig, bør legacy VLANs fjernes fra:

- ISP-R1 subinterfaces
- remote PE subinterfaces
- CE-CORE-R1 shutdown subinterfaces
- SW-DIST trunks
- SW-CE trunks/access ports
