# Peering Policy

## Service Range

- Any IXP that Strategic Explorations has access to. (For traffic levels > 20Mbps, we require that a BGP session be established directly with us.)
- Any PoP within the Strategic Explorations backbone. (It can be connected through a tunnel. We support GRE, Wireguard.)

## Cost

AS207268 adopts neutral peering policy and pricing strategy.

## Prefix

AS207268 will announce all prefixes in including but not limited to AS-SET:AS-STREXP.

## Filtering

- Incoming routes need to follow the filtering rules of IRR.
- Peers cannot announce private address spaces (RFC1918, RFC4193).
- The peer shall not announce private ASNs.
- Peers must not configure a static/default route to our router.

## SLA

There is no clear SLA guarantee. Strategic Explorations provides best-effort services.

## Maintenance Requirements

- Peers must comply with the relevant laws of the country where the PoP or IXP node is located.
- For asymmetric traffic, you need to negotiate with us and agree on a quota or ratio.
- For peers that frequently update routes or announce too many invalid routes, we will suspend the BGP session.

## Contact

The peer must have a 24x7 NOC to contact.
