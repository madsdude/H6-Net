# 02 - Topology

Dette dokument beskriver den overordnede topologi for H6-Net.

## Logisk overblik

```text
                 Internet
                    |
              CE-CORE-R1
                    |
          EPL / xconnect transport
                    |
        +-----------+-----------+
        |           |           |
      CE-AAH      CE-KBH      CE-ODE
```

## Provider transportlag

Provider-laget består af P- og PE-routere.

```text
CE/SW side ---- PE ---- P/Core ---- PE ---- CE/SW side
```

PE-routerne terminerer kundens Layer 2 services. P-routerne transporterer MPLS-labels mellem PE-routerne og bør ikke kende kundens CE-routing.

## Customer routinglag

Customer-laget består af CE-routere, hvor CE-CORE-R1 fungerer som centralt breakout-punkt.

```text
CE-AAH-R1 ---- EPL ---- CE-CORE-R1
CE-KBH-R1 ---- EPL ---- CE-CORE-R1
CE-KBH-R2 ---- EPL ---- CE-CORE-R1
CE-ODE-R1 ---- EPL ---- CE-CORE-R1
CE-ODE-R2 ---- EPL ---- CE-CORE-R1
```

## Designprincip

- P/PE-laget leverer kun transport
- EPL/xconnect leverer Layer 2 mellem CE-sider
- CE-laget bygger Layer 3 routing med BGP ovenpå EPL
- CORE annoncerer default route til sites
- NAT og internet breakout sker centralt på CORE
