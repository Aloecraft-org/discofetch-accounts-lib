# discofetch-accounts-lib

Identity and entitlement, extracted from `api/supervisor.lua`: who is asking,
whose face they are wearing, what their plan allows, and how a stranger
becomes a customer.

Nothing `require`s this yet. DRT guests have no `require` and no `dofile`;
the load-time modules slice is designed and unshipped, so this is a real
module written now and consumed later. `accounts.dlua`'s last statement is
`return M`, and the day the slice lands, wiring it up is two lines. Until
then `test/run.sh` concatenates the module and the cases into one program.

## The surface

**Entry points, pure** — no db, no clock, no host, so they are plain fields:

| | |
|---|---|
| `M.tier_of(name)` | the tier registry; **free** on a miss, never an error |
| `M.role_rank(name)` | the rank ladder; **0** on a miss, never admin |
| `M.opaque_token(authorization)` | the token grammar, strictly |
| `M.token_row(r)` | a token as the API shows it — never the hash |
| `M.grant_live_sql(t)` / `M.invite_live_sql(t)` | when a gift / a code counts |
| `M.validate_grant(spec, t)` | what a grant may say, for both doors |
| `M.validate_invite_fields(body, t)` | note / max_uses / expires_at, both doors |
| `M.invite_prefix(code)` | the only spelling of a code that may reach an event |
| `M.require_session(req, why)` | which credential arrived |
| `M.require_own_account(req, why)` | whose face is on |
| `M.resolve_actor(actor)` | the `acted_by` unwrap — see the edges below |

**Entry points, bound** — `M.new(deps)` returns an instance:

`:account_by_token` `:resolve_account` `:act_as_apply` `:quota_of`
`:grants_of` `:grant_insert` `:grant_object` `:grant_by_id` `:apply_confers`
`:invite_json` `:invite_insert` `:me_body` `:require_role` `:require_admin`
`:require_person`, and the handlers, each `(req, p) -> status, payload [, headers]`:
`:me_get` `:me_patch` `:grants_redeem` `:tokens_list` `:tokens_create`
`:tokens_delete` `:invites_list` `:invites_create` `:invites_delete`
`:redeem_post`.

**Configurable values** and **fan-out points** are declared in one block at
the top of `accounts.dlua` and nowhere else: the token and invite grammars,
the list and quota row caps, the five-minute coarseness, the refusal
sentences, and the four registries — `TIERS` (with the `plus` alias applied
at load), `GRANT_KINDS`, `GRANT_RESOURCES`, `ROLES`.

### Three rules that are decisions, not accidents

`quota_of` **layers three ways with three different combining rules**:
`quota_bonus` is **additive**, `tier_grant` is a **last-wins override**, and
the base tier is a **default with fallback**.

It reports provenance as **`tier_source`** (`account` or `grant`), because an
account page reading tier `free` beside a plan of `Homelab` is otherwise a bug
report.

A typo **degrades, never outages**: an unrecognised grant resource adds to
nothing rather than to the wrong thing, and an unknown tier falls back rather
than taking the account out. Write time is the only moment anybody can be
*told* — which is what `validate_grant` is for, and why the validator and
`quota_of` must never drift apart.

### The identity model

`x-df-sub` is the gateway's **assertion**, and it is trustworthy for exactly
one reason: the gateway strips every inbound `x-df-` header and re-adds it
after verifying the token itself. `act-as` sits **deliberately outside** that
namespace — nothing about its name suggests it was checked, because nothing
did until this library. It is honoured only for admins on a session, and for
everybody else it is **refused with an honest 403 rather than ignored**: a
caller who believes they are acting as a customer and is in fact acting as an
admin is the most dangerous of the three possible outcomes.
(`doc/DATA-MODEL.md` says "ignored"; the doc is older than the code.)

## The injected-deps contract

`M.new(deps)` asserts every dep by name, in a sentence. A missing or
wrong-typed dep is a failure at construction, never a nil dereference on
somebody's create.

| dep | what it is |
|---|---|
| `db` | the sql connector handle (`query`, `exec`, `try_exec`) |
| `now` | a function returning unix **seconds**. Never `host.time()` — the composition root is the only place that knows the host answers milliseconds |
| `json` | `encode` / `decode` |
| `ready` | a **predicate** `-> ready, error_sentence`. Never a captured boolean: the flag flips after `migrate()` runs, and a snapshot taken at construction answers 503 forever |
| `crypto` | `random(bytes) -> hex`, `hash(secret)` |
| `drtdb` | drt-db-lib — `rows_by_name` |
| `model` | discofetch-model-lib — `base32_from_hex` (required: the token and code draws) |
| `http` | drt-http-api-lib — `int_param` |
| `events` | a node-event-lib instance — `:log(kind, subject_id, actor, f)`, `:last(kind, subject_id, actor_id)` |
| `rate` | a token-rate-limit-lib instance — `:check(key, policy, what)` and `.POLICIES.redeem`. **One dep, not two**: an accounts-lib holding a raw bucket would silently bypass the policy table |

Two instances over two handles coexist; there is no module-level mutable
state.

## Usage

```lua
-- the event log's resolve_actor is supplied BY this library, which is what
-- keeps node-event-lib from ever learning the word `acted_by` -- so the log
-- is built first, and accounts is handed the instance:
local events = NodeEvent.new({ ..., resolve_actor = Accounts.resolve_actor })

local accounts = Accounts.new({
  db = db, json = json, now = now,
  ready  = function() return db_ready, db_error end,
  crypto = { random = host.crypto.random, hash = host.crypto.hash },
  drtdb = drtdb, model = model, http = httpapi,
  events = events, rate = limiter,
})

-- a route:
ROUTES['GET /v1/me'] = function(req, p) return accounts:me_get(req, p) end

-- a sibling reading the registry, never a second copy of it:
local per_hour = Accounts.tier_of(owner.tier).dns_per_hour
```

`rate:check` answers `true` when allowed, and `false, refusal` when not —
**one table**, carrying `status` / `code` / `message` / `body` / `headers` /
`wait`. It is assembled rather than split because the limiter serves two doors
that take it differently: `fail(conn, status, code, message, headers)` on the
fetchpoint side, a returned `status, body, headers` triple here. Unpacked as a
triple instead, the status would be the table and the body nil.
This library derives the key (`redeem:<x-real-ip>`, or `redeem:unknown`
— never a header the caller can seed), spends the bucket on every attempt
including the successful ones, and returns the refusal on through. It never
converts units and never names the numbers.

## Depends on

- **drt-db-lib** — `rows_by_name`, on every named read.
- **discofetch-model-lib** — `base32_from_hex`, for the token mint and the
  invite-code draw.
- **node-event-lib** — `:log` for eleven call sites, and `:last` for the act-as
  recency read (the only reader of the event table outside the log itself).
  This library **supplies** that one's `resolve_actor`; the two were extracted
  together, or the audit trail silently names the customer as the actor.
- **token-rate-limit-lib** — `:check` and `POLICIES.redeem` for the stranger's
  door.
- **drt-http-api-lib** — `int_param`.

Everything else depends on *this* library and never the reverse:
discofetch-fetchpoint-lib imports `tier_of`/`TIERS` so the dns and join verbs
meter at the **owner's** numbers, and the dispatcher imports `opaque_token`
for its credential gate.

## What changed in extraction, and why it is not drift

The arithmetic, the branch order and every refusal code are
`supervisor.lua`'s. What moved:

- `db_ready` / `db_error` became the `ready()` **predicate** (they flip after
  `migrate()`; a captured boolean freezes false).
- `rate_allow` + `rate_headers` + `REDEEM_PER_HOUR`/`REDEEM_BURST` became one
  `rate:check` call against token-rate-limit-lib's policy table.
- the act-as recency `SELECT` became `events:last(...)`, keeping the table
  name private to one module.
- `log_event`'s `acted_by` unwrap became exported `M.resolve_actor`.
- `host.crypto` became `deps.crypto`.
- the forward declaration of `act_as_apply` (a Lua local must precede its use)
  disappears.
- literals that were spelled at first use — the list caps, the 120-character
  names, the 64-character code bound, the two five-minute windows, and the
  refusal sentences that were inline strings in `tokens_create`,
  `tokens_delete` and `me_patch` — are declared in the surface block. No
  behaviour changes; `?2 - 300` is now concatenated from `M.SEEN_COARSE`, on
  `grant_live_sql`'s argument that an integer this program computed is safe to
  interpolate.

## Known, carried over

Believed to be defects. **Not fixed here** — this pass preserves behaviour.

1. **A code can be redeemed twice by the same account.** `redeem_post` checks
   `used_count < max_uses` and nothing records that *this* account already
   used *this* code, so an account holder can spend a friend's three-use code
   three times and collect its `confers` three times. The account INSERT is
   the boundary the comments defend; the confers are not behind it.
2. **`tokens_delete` is not idempotent the way `grants_redeem` is.** Both
   guard on `revoked_at IS NULL`, but the token route answers 404 when
   `changes == 0` — so the loser of two concurrent revokes is told the token
   does not exist, moments after it was revoked.
3. **`validate_grant` mutates its argument.** It writes the defaulted
   `detail.resource` back into the table it was handed. Both current callers
   want that; a third caller reusing a spec would not.
4. **Two clock reads per `/v1/me`.** `me_body` calls `now()` and `quota_of`
   calls it again, so a grant that expires between them is counted in the cap
   and missing from the list (or the reverse).
5. **`GRANT_COLS` selects `account_id`, which `grant_object` never serves.**
   One column per row read for nothing; the comment above it in supervisor
   reads as though the SELECT omitted it.
6. **A token-only account looks dormant.** The token path stamps
   `api_token.last_used_at` and never `account.last_seen_at`, so the panel's
   "last seen" for an account used exclusively through the API stops moving.

Behaviour that looks odd and is *not* a bug — it is argued for in the
comments, and the tests assert it: the four invite refusals are deliberately
indistinguishable; every token failure answers one sentence; `name_credit`'s
`spent` has no consumer on purpose; and a lapsed `tier_grant` in a code's
`confers` lands born-expired rather than silently conferring nothing.

## Tests

```
sh test/run.sh          # DRT=/path/to/drt to point it elsewhere
```

Green **only** when the last line is exactly `PASS`. The harness wraps
`accounts.dlua` in an IIFE and concatenates `test/cases.dlua`, which stands in
for the five sibling libraries and the connector with doubles and then asserts
this module's own arithmetic: the three quota layering rules and their
degradation, the six act-as refusals in order, the absence-vs-nil discipline
on every nullable column, the empty-array splices, the five-minute
coarseness, the idempotent redeem, and the key derivation the limiter is
handed.
