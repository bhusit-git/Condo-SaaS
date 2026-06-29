# Resident LIFF access goes through Edge Functions

Resident LIFF workflows start in v1.1 and access customer data through Edge
Functions/RPC only, rather than querying Supabase customer-data tables directly
from the LIFF frontend. This keeps resident actor resolution, active LINE Binding
checks, selected condo/unit context, revocation behavior, and audit boundaries
server-side while the product is still using LIFF identity instead of a dedicated
Supabase resident session model.

The required LIFF request verification, context derivation, allowed
selected-context check, error envelope, and v1.1 Edge Function surface are defined
in [v1 Implementation Contract](../v1-implementation-contract.md).

**Considered Options**

- Let LIFF mint a resident Supabase session/JWT and rely on RLS claims for direct table access.
- Keep LIFF as a thin client and resolve resident scope inside Edge Functions/RPC.

**Consequences**

RLS remains the database safety net, but resident-facing reads and writes are
exposed as explicit server-side workflows in v1.1. A future direct Supabase
resident session model would need a separate decision covering custom claims,
token lifetime, context switching, and binding revocation.
