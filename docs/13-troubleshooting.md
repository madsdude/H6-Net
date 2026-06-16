# 13 - Troubleshooting

Dette dokument samler fejlfinding for H6-Net.

## Fejlfinding pr. lag

Start altid nedefra og arbejd op:

```text
1. Interface/link
2. VLAN/trunk
3. Xconnect/MPLS/LDP
4. IP reachability
5. OSPF/BGP
6. NAT/internet
```

## Interface og IP

```bash
show ip interface brief
show interfaces description
show interfaces <interface>
show running-config interface <interface>
```

Tjek:

- Er interface up/up?
- Er IP-adressen korrekt?
- Er interface shutdown?
- Er subinterface dot1Q korrekt?

## VLAN og trunk

På switch/PE trunk:

```bash
show interfaces trunk
show vlan brief
show mac address-table vlan <vlan>
```

Tjek:

- Er VLAN tilladt på trunk?
- Er CE-port i korrekt VLAN?
- Er VLAN oprettet på switchen?
- Matcher VLAN med PE subinterface og CORE subinterface?

## Xconnect/MPLS

På PE:

```bash
show xconnect all
show mpls l2transport vc
show mpls forwarding-table
show mpls interfaces
show mpls ldp neighbor
```

Tjek:

- Er VC UP?
- Matcher VC-ID i begge ender?
- Kan PE nå remote PE loopback?
- Er LDP neighbor up?
- Er MPLS aktiveret på core links?

## Provider OSPF/BFD

```bash
show ip ospf neighbor
show ip route ospf
show bfd neighbors
show ip route <remote-pe-loopback>
```

Tjek:

- Er OSPF neighbors FULL?
- Er BFD up?
- Kan ISP-R1 nå `4.4.4.4`?
- Har P1/P2 kun provider routes og ikke kundens BGP-ruter?

## CE OSPF under migration

```bash
show ip ospf neighbor
show ip route ospf
show running-config | section router ospf
```

Vigtigt fund:

- CE-CORE-R1 mangler OSPF network statement for `10.10.0.5`, hvis AAH CE2-linket skal køre OSPF under migration.

## BGP

På CE-CORE-R1:

```bash
show ip bgp summary
show ip bgp
show ip route bgp
show ip bgp neighbors <neighbor> received-routes
show ip bgp neighbors <neighbor> advertised-routes
```

På site CE:

```bash
show ip bgp summary
show ip route 0.0.0.0
show ip route bgp
```

Tjek:

- Er neighbor Established?
- Er remote-as korrekt?
- Er source/destination IP reachable?
- Bliver loopback annonceret med `network`?
- Kommer default route fra CORE?
- Vælger CORE primær route via CE1?

## NAT/internet

På CE-CORE-R1:

```bash
show ip nat translations
show ip nat statistics
show access-lists NAT-SITES
show ip route 0.0.0.0
```

Fra site CE:

```bash
ping 11.11.11.11 source Loopback0
ping 8.8.8.8 source Loopback0
traceroute 8.8.8.8 source Loopback0
```

Tjek:

- Er loopback med i NAT ACL?
- Er CE2 loopback også med?
- Er relevant /30-net med i NAT ACL?
- Har CE-CORE-R1 default route til upstream?
- Er Gi0/1 NAT outside?

## Hurtig kommando-checkliste

### På CE-CORE-R1

```bash
show ip int brief
show ip bgp summary
show ip route bgp
show ip route 0.0.0.0
show access-lists NAT-SITES
show ip nat translations
show run | section router bgp
show run | section router ospf
```

### På site CE

```bash
show ip int brief
show ip bgp summary
show ip route
show ip route 0.0.0.0
show run | section router bgp
ping 11.11.11.11 source Loopback0
```

### På provider PE/P

```bash
show ip ospf neighbor
show bfd neighbors
show mpls ldp neighbor
show mpls forwarding-table
show xconnect all
```

## Typiske fejl og løsning

| Fejl | Sandsynlig årsag | Løsning |
| --- | --- | --- |
| BGP neighbor Idle/Active | IP reachability/VLAN/xconnect fejl | Test ping mellem BGP peer IP'er og tjek xconnect |
| CE2 kan ikke gå på internet | CE2 loopback mangler i NAT ACL | Tilføj CE2 loopback og CE2 /30-net |
| CORE ser kun CE1 route | CE2 BGP/iBGP/xconnect nede | Tjek CE2 eBGP og interconnect |
| Xconnect down | MPLS/LDP eller VC mismatch | Tjek LDP, remote PE loopback og VC-ID |
| OSPF neighbor mangler på AAH CE2 | CE-CORE mangler network `10.10.0.5` | Tilføj network statement eller fjern OSPF afhængigt af mål |
