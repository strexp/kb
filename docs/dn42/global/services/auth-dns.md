---
tags:
  - v6-only
---

# Authorative DNS

!!! info "Not a Recursive Resolver"
    These servers are **Authoritative Only**. Do not configure them as your system's DNS resolver. They will only answer queries for the domains listed below.

## Server Address

| Node | Type | Address |
| :--- | :--- | :--- |
| Anycast | - | `fd00:1926:817:5353::1` |
| sol3.kr | Primary Server | `fd00:1926:817:5353::2` |

## Hosted Domains

| Domain              | DNSSEC |
| ------------------- | ------ |
| nia.dn42            | N      |
| gensokyo.dn42       | N      |
| nu.dn42             | N      |

## Hosted Reverse Domains

| Domain                           | DNSSEC |
| -------------------------------- | ------ |
| 128-25.168.20.172.in-addr.arpa   | N      |
| 128-26.158.20.172.in-addr.arpa   | N      |
| 6.5.f.5.7.d.a.5.0.d.d.f.ip6.arpa | N      |
| 7.1.8.0.6.2.9.1.0.0.d.f.ip6.arpa | N      |
| 7.1.8.0.6.2.9.1.1.0.d.f.ip6.arpa | N      |
