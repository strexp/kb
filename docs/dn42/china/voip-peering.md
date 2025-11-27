# VOIP Peering

## Server

`pbx.gensokyo.dn42`

## Prefix

`(4240) 0803`

## Numbering Scheme

We follow a unified numbering scheme for routing calls within the network.

| Prefix | Usage | Example |
| :--- | :--- | :--- |
| **424** | Country Code | Pan-DN42 Region |
| **0** / **1** | Region | 0=DN42, 1=NeoNetwork |
| **XXXX** | Area Code | Last 4 digits of target ASN |
| **Variable** | Subscriber | Assigned user number |

Example: To call ASN 424242**0803**, dial: `424` + `0` + `0803` + `Extension`.
