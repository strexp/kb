# BGP Communities

Strategic Explorations uses BGP communities to control traffic engineering and propagate route information.

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

| Community           | Description                                 |
| ------------------- | ------------------------------------------- |
| (OURASN, 6, 41)     | Origin Region: Europe
| (OURASN, 6, 42)     | Origin Region: North America-E
| (OURASN, 6, 43)     | Origin Region: North America-C
| (OURASN, 6, 44)     | Origin Region: North America-W
| (OURASN, 6, 45)     | Origin Region: Central America
| (OURASN, 6, 46)     | Origin Region: South America-E
| (OURASN, 6, 47)     | Origin Region: South America-W
| (OURASN, 6, 48)     | Origin Region: Africa-N (above Sahara)
| (OURASN, 6, 49)     | Origin Region: Africa-S (below Sahara)
| (OURASN, 6, 50)     | Origin Region: Asia-S (IN,PK,BD)
| (OURASN, 6, 51)     | Origin Region: Asia-SE (TH,SG,PH,ID,MY)
| (OURASN, 6, 52)     | Origin Region: Asia-E (JP,CN,KR,TW,HK)
| (OURASN, 6, 53)     | Origin Region: Pacific&Oceania (AU,NZ,FJ)
| (OURASN, 6, 54)     | Origin Region: Antarctica
| (OURASN, 6, 55)     | Origin Region: Asia-N (RU)
| (OURASN, 6, 56)     | Origin Region: Asia-W (IR,TR,UAE)
| (OURASN, 6, 57)     | Origin Region: Central Asia (AF,UZ,KZ)

### 7: Route Import Region Indicator

| Community           | Description                                 |
| ------------------- | ------------------------------------------- |
| (OURASN, 7, 41)     | Import Region: Europe
| (OURASN, 7, 42)     | Import Region: North America-E
| (OURASN, 7, 43)     | Import Region: North America-C
| (OURASN, 7, 44)     | Import Region: North America-W
| (OURASN, 7, 45)     | Import Region: Central America
| (OURASN, 7, 46)     | Import Region: South America-E
| (OURASN, 7, 47)     | Import Region: South America-W
| (OURASN, 7, 48)     | Import Region: Africa-N (above Sahara)
| (OURASN, 7, 49)     | Import Region: Africa-S (below Sahara)
| (OURASN, 7, 50)     | Import Region: Asia-S (IN,PK,BD)
| (OURASN, 7, 51)     | Import Region: Asia-SE (TH,SG,PH,ID,MY)
| (OURASN, 7, 52)     | Import Region: Asia-E (JP,CN,KR,TW,HK)
| (OURASN, 7, 53)     | Import Region: Pacific&Oceania (AU,NZ,FJ)
| (OURASN, 7, 54)     | Import Region: Antarctica
| (OURASN, 7, 55)     | Import Region: Asia-N (RU)
| (OURASN, 7, 56)     | Import Region: Asia-W (IR,TR,UAE)
| (OURASN, 7, 57)     | Import Region: Central Asia (AF,UZ,KZ)
