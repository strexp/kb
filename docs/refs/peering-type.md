# Peering Types

Strategic Explorations (StrExp) classifies BGP sessions into distinct types to manage routing policies effectively.

## Default Policies

- For IANA, StrExp considers all requests by default to be of the *Peer* type unless otherwise specified.
- For DN42, StrExp considers all requests by default to be of the *Full* type unless otherwise specified, and believes that this will help improve the decentralization level of the entire network.

## Matrix

| FROM(right) TO(down) | full | priv transit | transit | downstream | priv peer  | peer | self          | unknown |
| -------------------- | ---- | ------------ | ------- | ---------- | ---------- | ---- | ------------- | ------- |
| full                 | Y    | N            | Y       | Y          | N          | Y    | Y             | Y       |
| priv transit         | N    | N            | N       | N          | N          | N    | Y             | N       |
| transit              | N    | N            | N       | Y          | N          | N    | Y             | Y       |
| downstream           | Y    | N            | Y       | Y          | N\*        | Y\*  | Y             | Y       |
| priv peer            | N    | N            | N       | N          | N\*        | N    | Y (no-export) | N       |
| peer                 | N    | N            | N       | Y\*        | N\*        | N    | Y             | N       |
| self                 | Y    | Y            | Y       | Y          | Y (path=1) | Y    | -             | Y       |

!!! note "Exceptions"
    Entries marked with `*` in internal docs imply exceptions are allowed based on specific bilateral agreements.
