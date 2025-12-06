# BGP Communities

Strategic Explorations uses BGP communities to control traffic engineering and propagate route information.

## Region Identifers

| Identifers          | Description                                 |
| ------------------- | ------------------------------------------- |
| 41                  | Europe |
| 42                  | North America-E |
| 43                  | North America-C |
| 44                  | North America-W |
| 45                  | Central America |
| 46                  | South America-E |
| 47                  | South America-W |
| 48                  | Africa-N (above Sahara) |
| 49                  | Africa-S (below Sahara) |
| 50                  | Asia-S (IN,PK,BD) |
| 51                  | Asia-SE (TH,SG,PH,ID,MY) |
| 52                  | Asia-E (JP,CN,KR,TW,HK) |
| 53                  | Pacific&Oceania (AU,NZ,FJ) |
| 54                  | Antarctica |
| 55                  | Asia-N (RU) |
| 56                  | Asia-W (IR,TR,UAE) |
| 57                  | Central Asia (AF,UZ,KZ) |

## Well Known Community

### 666: Blackhole (RTBH)

> This community is used to drop traffic to a specific IP under DDoS attack.

| Community           | Description                                 |
| ------------------- | ------------------------------------------- |
| (OURASN, 666, 0)    | Discard traffic to this prefix (Blackhole)  |

## Public Community

> These communities can be set by peers to influence our routing decisions.

### 2: Local Preference Control

| Community           | Description                                 |
| ------------------- | ------------------------------------------- |
| (OURASN, 2, 1)      | Set Local Preference to 100 (default value) |
| (OURASN, 2, 2)      | Set Local Preference to 200                 |
| (OURASN, 2, 3)      | Set Local Preference to 0                   |

### 5: Prepend Control

| Community           | Description                                 |
| ------------------- | ------------------------------------------- |
| (OURASN, 5, 1)      | Path Prepend x1                             | 
| (OURASN, 5, 2)      | Path Prepend x2                             | 
| (OURASN, 5, 3)      | Path Prepend x3                             | 
| (OURASN, 5, 4)      | Path Prepend x4                             | 
| (OURASN, 5, 5)      | Path Prepend x5                             | 

### 4: Peer Type Indicator

| Community           | Description                                 |
| ------------------- | ------------------------------------------- |
| (OURASN, 4, 10)     | Peer Type: Full Table                       |
| (OURASN, 4, 20)     | Peer Type: Transit                          |
| (OURASN, 4, 21)     | Peer Type: Private Transit                  |
| (OURASN, 4, 30)     | Peer Type: Downstream                       |
| (OURASN, 4, 40)     | Peer Type: Peer                             |
| (OURASN, 4, 41)     | Peer Type: Private Peer                     |

### 6: Route Origin Region Indicator

!!! notice "DN42 region communities are auto translated when importing to our network."

| Community             | Description                                 |
| --------------------- | ------------------------------------------- |
| (OURASN, 6, {REGION}) | Origin Region: `REGION`                     |

### 7: Route Import Region Indicator

| Community             | Description                                 |
| --------------------- | ------------------------------------------- |
| (OURASN, 7, {REGION}) | Import Region: `REGION`                     |

### 11/12/13: Regional Prepend Control

| Community              | Description                                 |
| ---------------------- | ------------------------------------------- |
| (OURASN, 11, {REGION}) | Prepend x1 at `REGION`                      |
| (OURASN, 12, {REGION}) | Prepend x2 at `REGION`                      |
| (OURASN, 13, {REGION}) | Prepend x3 at `REGION`                      |
