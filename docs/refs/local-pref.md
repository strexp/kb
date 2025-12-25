# Local Pref Calculation

This document outlines the policy for manipulating BGP local_pref attributes. The goal is to optimize traffic engineering by prioritizing routes based on AS Path length and Geographical proximity.

This strategy ensures that traffic prefers:

- Directly connected peers.
- Routes with fewer AS hops.
- Routes that traverse geographically closer regions (lower latency).

## AS Path Length

- **Direct Peering Bonus**: If the AS Path length is 1 (direct peer), we increase the preference significantly (+3) to prioritize direct connections over transit.
- **Hop Penalty**: For all other routes, we strictly penalize the preference based on the number of hops (-length).

```
if bgp_path.len = 1 then bgp_local_pref = bgp_local_pref + 3;
bgp_local_pref = bgp_local_pref - bgp_path.len;
```

## Origin Region Community (DN42)

### Logic & Penalty

This configuration applies a penalty value (ranging from 0 to 5) to the `local_pref`.

- Penalty 0 (Local): No penalty. The route originates or passes through the same region. Highest priority.
- Penalty 1 (Adjacent): Directly neighboring regions (e.g., Europe to North Africa, US East to US Central). Expected Latency: < 80ms.
- Penalty 2 (Inter-Regional): Same continent but distant, or short trans-oceanic links (e.g., Europe to US Central, US East to US West). Expected Latency: < 150ms.
- Penalty 3 (Long Distance): Long trans-oceanic or cross-continental links (e.g., Europe to US West, US East to South America). Expected Latency: < 220ms.
- Penalty 4 (Very Long Distance): Extreme trans-oceanic distances (e.g., Europe to East Asia, US East to Southeast Asia). Expected Latency: < 300ms.
- Penalty 5 (Remote/Antipodal): Physically opposite points on the globe or regions with difficult connectivity (e.g., Europe to Oceania/Antarctica). Expected Latency: > 300ms.

| Region (Local) | EU | NA_E | NA_C | NA_W | CA | SA_E | SA_W | AF_N | AF_S | AS_S | AS_SE | AS_E | PA | AQ | AS_N | AS_W | AS_C |
| :--- | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: |
| **EU (41)** | 0 | 1 | 2 | 3 | 3 | 3 | 4 | 1 | 2 | 3 | 4 | 4 | 5 | 5 | 1 | 1 | 2 |
| **NA_E (42)** | 1 | 0 | 1 | 2 | 2 | 2 | 3 | 2 | 3 | 4 | 4 | 4 | 5 | 5 | 3 | 3 | 4 |
| **NA_C (43)** | 2 | 1 | 0 | 1 | 1 | 3 | 3 | 3 | 4 | 4 | 4 | 3 | 4 | 5 | 3 | 4 | 4 |
| **NA_W (44)** | 3 | 2 | 1 | 0 | 2 | 4 | 4 | 4 | 5 | 4 | 3 | 2 | 2 | 5 | 3 | 4 | 4 |
| **CA (45)** | 3 | 2 | 1 | 2 | 0 | 2 | 2 | 3 | 4 | 5 | 5 | 4 | 4 | 5 | 4 | 4 | 5 |
| **SA_E (46)** | 3 | 2 | 3 | 4 | 2 | 0 | 1 | 3 | 3 | 5 | 5 | 5 | 5 | 4 | 5 | 4 | 5 |
| **SA_W (47)** | 4 | 3 | 3 | 4 | 2 | 1 | 0 | 4 | 4 | 5 | 4 | 5 | 3 | 4 | 5 | 5 | 5 |
| **AF_N (48)** | 1 | 2 | 3 | 4 | 3 | 3 | 4 | 0 | 2 | 3 | 4 | 4 | 5 | 5 | 2 | 1 | 2 |
| **AF_S (49)** | 2 | 3 | 4 | 5 | 4 | 3 | 4 | 2 | 0 | 3 | 3 | 5 | 4 | 3 | 4 | 3 | 4 |
| **AS_S (50)** | 3 | 4 | 4 | 4 | 5 | 5 | 5 | 3 | 3 | 0 | 2 | 3 | 4 | 5 | 3 | 1 | 1 |
| **AS_SE (51)**| 4 | 4 | 4 | 3 | 5 | 5 | 4 | 4 | 3 | 2 | 0 | 1 | 2 | 5 | 3 | 3 | 3 |
| **AS_E (52)** | 4 | 4 | 3 | 2 | 4 | 5 | 5 | 4 | 5 | 3 | 1 | 0 | 2 | 5 | 2 | 3 | 2 |
| **PA (53)** | 5 | 5 | 4 | 2 | 4 | 5 | 3 | 5 | 4 | 4 | 2 | 2 | 0 | 4 | 4 | 5 | 5 |
| **AQ (54)** | 5 | 5 | 5 | 5 | 5 | 4 | 4 | 5 | 3 | 5 | 5 | 5 | 4 | 0 | 5 | 5 | 5 |
| **AS_N (55)** | 1 | 3 | 3 | 3 | 4 | 5 | 5 | 2 | 4 | 3 | 3 | 2 | 4 | 5 | 0 | 2 | 1 |
| **AS_W (56)** | 1 | 3 | 4 | 4 | 4 | 4 | 5 | 1 | 3 | 1 | 3 | 3 | 5 | 5 | 2 | 0 | 1 |
| **AS_C (57)** | 2 | 4 | 4 | 4 | 5 | 5 | 5 | 2 | 4 | 1 | 3 | 2 | 5 | 5 | 1 | 1 | 0 |

### BIRD Configuration Implementation

The following function calc_region_pref() implements the matrix above.

<details>
<summary><strong>Click to expand the full BIRD code</strong></summary>

```bird
function calc_region_pref(){
  if REGION = 41 then {
    if COMM_REGION_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_REGION_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_REGION_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_REGION_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_REGION_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_REGION_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_REGION_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
  }
  if REGION = 42 then {
    if COMM_REGION_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_REGION_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_REGION_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_REGION_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_REGION_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_REGION_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_REGION_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
  }
  if REGION = 43 then {
    if COMM_REGION_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_REGION_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_REGION_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_REGION_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_REGION_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
  }
  if REGION = 44 then {
    if COMM_REGION_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_REGION_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_REGION_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_REGION_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_REGION_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_REGION_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
  }
  if REGION = 45 then {
    if COMM_REGION_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_REGION_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_REGION_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_REGION_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_REGION_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_REGION_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
  }
  if REGION = 46 then {
    if COMM_REGION_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_REGION_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_REGION_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_REGION_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
  }
  if REGION = 47 then {
    if COMM_REGION_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_REGION_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_REGION_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
  }
  if REGION = 48 then {
    if COMM_REGION_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_REGION_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_REGION_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_REGION_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_REGION_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_REGION_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
  }
  if REGION = 49 then {
    if COMM_REGION_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_REGION_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_REGION_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
  }
  if REGION = 50 then {
    if COMM_REGION_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_REGION_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_REGION_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
  }
  if REGION = 51 then {
    if COMM_REGION_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_REGION_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_REGION_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_REGION_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
  }
  if REGION = 52 then {
    if COMM_REGION_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_REGION_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_REGION_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_REGION_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_REGION_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
  }
  if REGION = 53 then {
    if COMM_REGION_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_REGION_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_REGION_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_REGION_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
  }
  if REGION = 54 then {
    if COMM_REGION_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
  }
  if REGION = 55 then {
    if COMM_REGION_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_REGION_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_REGION_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_REGION_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_REGION_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
  }
  if REGION = 56 then {
    if COMM_REGION_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_REGION_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_REGION_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_REGION_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_REGION_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
  }
  if REGION = 57 then {
    if COMM_REGION_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_REGION_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_REGION_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 4;
    if COMM_REGION_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_REGION_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_REGION_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_REGION_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 5;
    if COMM_REGION_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_REGION_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
  }
}
```
</details>

## Import Region Community

While orgion region community represents where the route originated (the physical location of the destination server), import region community represents the location of the Ingress Router (where the traffic enters our backbone network).

This policy penalizes routes based on the distance between the Local Router (where the decision is made) and the Ingress Point. This represents the cost of carrying traffic across our own backbone.

Penalty Range: 0 - 3 (Lower than Origin Region penalty to prefer backbone distance over external path distance in tie-breakers, or simply to add granular control).

- Penalty 0 (Local): Ingress is in the same region. Minimal backbone cost.
- Penalty 1 (Neighbor): Ingress is in a directly adjacent region or connected via low-latency links (Latency < 100ms).
- Penalty 2 (Medium): Ingress is on the same continent but distant, or short trans-oceanic (Latency < 200ms).
- Penalty 3 (Far): Ingress is on a different continent or very distant (Latency > 200ms).

### Penalty Matrix

| Local \ Import | EU | NA_E | NA_C | NA_W | CA | SA_E | SA_W | AF_N | AF_S | AS_S | AS_SE | AS_E | PA | AQ | AS_N | AS_W | AS_C |
| :--- | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: |
| **EU (41)** | 0 | 1 | 2 | 2 | 2 | 2 | 3 | 1 | 1 | 2 | 3 | 3 | 3 | 3 | 1 | 1 | 1 |
| **NA_E (42)** | 1 | 0 | 1 | 1 | 1 | 2 | 2 | 2 | 2 | 3 | 3 | 3 | 3 | 3 | 2 | 2 | 3 |
| **NA_C (43)** | 2 | 1 | 0 | 1 | 1 | 2 | 2 | 2 | 3 | 3 | 3 | 2 | 3 | 3 | 2 | 3 | 3 |
| **NA_W (44)** | 2 | 1 | 1 | 0 | 1 | 3 | 3 | 3 | 3 | 3 | 2 | 1 | 2 | 3 | 2 | 3 | 3 |
| **CA (45)** | 2 | 1 | 1 | 1 | 0 | 1 | 1 | 2 | 3 | 3 | 3 | 3 | 3 | 3 | 3 | 3 | 3 |
| **SA_E (46)** | 2 | 2 | 2 | 3 | 1 | 0 | 1 | 2 | 2 | 3 | 3 | 3 | 3 | 3 | 3 | 3 | 3 |
| **SA_W (47)** | 3 | 2 | 2 | 3 | 1 | 1 | 0 | 3 | 3 | 3 | 3 | 3 | 2 | 3 | 3 | 3 | 3 |
| **AF_N (48)** | 1 | 2 | 2 | 3 | 2 | 2 | 3 | 0 | 1 | 2 | 3 | 3 | 3 | 3 | 1 | 1 | 1 |
| **AF_S (49)** | 1 | 2 | 3 | 3 | 3 | 2 | 3 | 1 | 0 | 2 | 2 | 3 | 3 | 2 | 3 | 2 | 3 |
| **AS_S (50)** | 2 | 3 | 3 | 3 | 3 | 3 | 3 | 2 | 2 | 0 | 1 | 2 | 3 | 3 | 2 | 1 | 1 |
| **AS_SE (51)**| 3 | 3 | 3 | 2 | 3 | 3 | 3 | 3 | 2 | 1 | 0 | 1 | 1 | 3 | 2 | 2 | 2 |
| **AS_E (52)** | 3 | 3 | 2 | 1 | 3 | 3 | 3 | 3 | 3 | 2 | 1 | 0 | 1 | 3 | 1 | 2 | 1 |
| **PA (53)** | 3 | 3 | 3 | 2 | 3 | 3 | 2 | 3 | 3 | 3 | 1 | 1 | 0 | 3 | 3 | 3 | 3 |
| **AQ (54)** | 3 | 3 | 3 | 3 | 3 | 3 | 3 | 3 | 2 | 3 | 3 | 3 | 3 | 0 | 3 | 3 | 3 |
| **AS_N (55)** | 1 | 2 | 2 | 2 | 3 | 3 | 3 | 1 | 3 | 2 | 2 | 1 | 3 | 3 | 0 | 1 | 1 |
| **AS_W (56)** | 1 | 2 | 3 | 3 | 3 | 3 | 3 | 1 | 2 | 1 | 2 | 2 | 3 | 3 | 1 | 0 | 1 |
| **AS_C (57)** | 1 | 3 | 3 | 3 | 3 | 3 | 3 | 1 | 3 | 1 | 2 | 1 | 3 | 3 | 1 | 1 | 0 |

---

### BIRD Configuration Implementation

<details>
<summary><strong>Click to expand the full BIRD code</strong></summary>

```bird
function calc_import_region_pref_dn42(){
  if REGION = 41 then {
    if COMM_IMPORT_REGION_DN42_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
  }
  if REGION = 42 then {
    if COMM_IMPORT_REGION_DN42_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
  }
  if REGION = 43 then {
    if COMM_IMPORT_REGION_DN42_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
  }
  if REGION = 44 then {
    if COMM_IMPORT_REGION_DN42_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
  }
  if REGION = 45 then {
    if COMM_IMPORT_REGION_DN42_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
  }
  if REGION = 46 then {
    if COMM_IMPORT_REGION_DN42_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
  }
  if REGION = 47 then {
    if COMM_IMPORT_REGION_DN42_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
  }
  if REGION = 48 then {
    if COMM_IMPORT_REGION_DN42_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
  }
  if REGION = 49 then {
    if COMM_IMPORT_REGION_DN42_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
  }
  if REGION = 50 then {
    if COMM_IMPORT_REGION_DN42_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
  }
  if REGION = 51 then {
    if COMM_IMPORT_REGION_DN42_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
  }
  if REGION = 52 then {
    if COMM_IMPORT_REGION_DN42_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
  }
  if REGION = 53 then {
    if COMM_IMPORT_REGION_DN42_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
  }
  if REGION = 54 then {
    if COMM_IMPORT_REGION_DN42_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
  }
  if REGION = 55 then {
    if COMM_IMPORT_REGION_DN42_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
  }
  if REGION = 56 then {
    if COMM_IMPORT_REGION_DN42_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
  }
  if REGION = 57 then {
    if COMM_IMPORT_REGION_DN42_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_DN42_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_DN42_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_DN42_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
  }
}

function calc_import_region_pref_iana(){
  if REGION = 41 then {
    if COMM_IMPORT_REGION_IANA_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
  }
  if REGION = 42 then {
    if COMM_IMPORT_REGION_IANA_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
  }
  if REGION = 43 then {
    if COMM_IMPORT_REGION_IANA_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
  }
  if REGION = 44 then {
    if COMM_IMPORT_REGION_IANA_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
  }
  if REGION = 45 then {
    if COMM_IMPORT_REGION_IANA_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
  }
  if REGION = 46 then {
    if COMM_IMPORT_REGION_IANA_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
  }
  if REGION = 47 then {
    if COMM_IMPORT_REGION_IANA_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
  }
  if REGION = 48 then {
    if COMM_IMPORT_REGION_IANA_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
  }
  if REGION = 49 then {
    if COMM_IMPORT_REGION_IANA_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
  }
  if REGION = 50 then {
    if COMM_IMPORT_REGION_IANA_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
  }
  if REGION = 51 then {
    if COMM_IMPORT_REGION_IANA_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
  }
  if REGION = 52 then {
    if COMM_IMPORT_REGION_IANA_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
  }
  if REGION = 53 then {
    if COMM_IMPORT_REGION_IANA_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
  }
  if REGION = 54 then {
    if COMM_IMPORT_REGION_IANA_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
  }
  if REGION = 55 then {
    if COMM_IMPORT_REGION_IANA_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
  }
  if REGION = 56 then {
    if COMM_IMPORT_REGION_IANA_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_AS_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
  }
  if REGION = 57 then {
    if COMM_IMPORT_REGION_IANA_EU ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_NA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_NA_C ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_NA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_CA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_SA_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_SA_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AF_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_AF_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_S ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_AS_SE ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
    if COMM_IMPORT_REGION_IANA_AS_E ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_PA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AQ ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 3;
    if COMM_IMPORT_REGION_IANA_AS_N ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
    if COMM_IMPORT_REGION_IANA_AS_W ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
  }
}
```
</details>

## Special: Strategic Explorations Managed Networks

```
if arg_network = "dn42" then {
  if (OWNAS42 = 4242420803) && (bgp_path.first = 4242421331) then bgp_local_pref = bgp_local_pref + 2;
}
```

## Special: Avoid transit across China GFW

```
if REGION_COUNTRY_CN != COUNTRY && COMM_REGION_COUNTRY_CN ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
if REGION_COUNTRY_CN = COUNTRY && COMM_REGION_COUNTRY_CN ~ bgp_large_community then bgp_local_pref = bgp_local_pref + 1;
```

## Special: Avoid unknown source origin

```
if COMM_REGION_UNKNOWN ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 2;
```

## Special: Prefer eBGP

```
if arg_network = "iana" then {
  if COMM_SOURCE_IBGP_IANA ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
}
if arg_network = "dn42" then {
  if COMM_SOURCE_IBGP_DN42 ~ bgp_large_community then bgp_local_pref = bgp_local_pref - 1;
}
```
