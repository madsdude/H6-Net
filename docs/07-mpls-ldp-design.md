# 07 - MPLS og LDP design

Dette dokument beskriver MPLS/LDP-laget i H6-Net.

## Formål

MPLS/LDP bruges i providerlaget til at transportere xconnect pseudowires mellem PE-routerne. CE-routerne kører ikke MPLS. De ser kun en Layer 2 EPL-forbindelse.

## Designprincip

```text
CE/SW access VLAN
  -> PE dot1Q subinterface
  -> xconnect encapsulation mpls
  -> MPLS/LDP core
  -> remote PE
  -> CE-CORE-R1
```

P-routerne skal kun kende provider-loopbacks og provider-linknet. De skal ikke kende kundens BGP-ruter.

## Aktiveret MPLS

MPLS er aktiveret på provider core links med:

```text
mpls label protocol ldp
interface <provider-link>
 mpls ip
```

P1 og P2 har desuden:

```text
mpls ldp router-id Loopback0 force
```

Det bør også standardiseres på ISP-R1.

## LDP reachability

LDP kræver IP reachability mellem PE loopbacks. Derfor skal OSPF 10 annoncere loopbacks og core links.

Minimumskrav:

| Krav | Hvorfor |
| --- | --- |
| Loopbacks i OSPF | LDP router-id reachability |
| `mpls ip` på core interfaces | MPLS label forwarding |
| Samme IGP path mellem PE'er | Stabil transport for pseudowires |
| LDP neighbor up | MPLS labels kan udveksles |

## Verification commands

Kør disse på provider-enhederne:

```bash
show ip ospf neighbor
show ip route ospf
show mpls interfaces
show mpls ldp neighbor
show mpls forwarding-table
show mpls l2transport vc
show xconnect all
```

## Forventet status

- OSPF neighbors mellem ISP-R1, P1 og P2 skal være FULL.
- LDP neighbors skal være up på MPLS-core links.
- `show xconnect all` skal vise VC state UP for aktive EPL services.
- P-routerne skal ikke have CE-BGP-ruter.

## Fejlsøgning

| Symptom | Sandsynlig årsag | Tjek |
| --- | --- | --- |
| Xconnect down | Manglende remote PE, VC-ID mismatch eller LDP issue | `show xconnect all`, `show mpls l2transport vc` |
| LDP neighbor mangler | Ingen `mpls ip`, IGP reachability mangler, router-id issue | `show mpls ldp neighbor`, `show ip route <peer-loopback>` |
| MPLS forwarding mangler | MPLS ikke aktiveret på interface | `show mpls interfaces` |
| Path går forkert | OSPF cost eller manglende link | `show ip route <remote-loopback>` |

## Standardisering

Brug samme MPLS-template på alle provider core-enheder:

```text
mpls label protocol ldp
!
interface Loopback0
 ip address <router-id> 255.255.255.255
!
mpls ldp router-id Loopback0 force
!
router ospf 10
 router-id <router-id>
 bfd all-interfaces
```
