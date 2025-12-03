# IPv6 Transition Policy

!!! success "2025/02: All IPv6 Transition steps have been completed and our backbone network is now in full IPv6-only operation."

IANA's IPv4 resources have been exhausted, and although nearly half of DN42's IPv4 resources remain, the overall shift to IPv6 on the Internet is a general trend.

Therefore, under our repeated consideration, we decided to completely terminate IPv4 content services in 2021 and only retain IPv4 Transit services. At the same time, considering compatibility, we will consider cooperating with services such as 464XLAT and NAT64 to continue to provide customers with the IANA network IPv4 access in.

## Current Architecture

- **IANA Network**: Pure IPv6 (Single Stack).
- **DN42 Network**:
    - Global Network: Pure IPv6 (Single Stack).
    - China Network: IPv6 + IPv4 (Dual Stack).

## Service Reachability

For customers and peers requiring access to legacy IPv4 endpoints, we provide the following translation services:

- **NAT46 Prefix**: `2a0d:2580:ff0c:6464::/96`
- **NAT64 Prefix**: `2602:feda:3ca:6464::/96`

## History of Migration

The migration was executed in three phases:

1.  **Phase 1: Service Audit**: All internal services were verified for IPv6 reachability. Legacy-only services were deprecated.
2.  **Phase 2: DNS Shift**:  Authoritative DNS ceased publishing A records for internal infrastructure, moving fully to AAAA.
3.  **Phase 3: Backbone Switch**: Native IPv4 IGP/BGP was turned off. DN42 IPv4 announcements were ceased, and IANA IPv4 Transit was migrated to 464XLAT.

We thank all our peers and customers for their cooperation during this transition.
