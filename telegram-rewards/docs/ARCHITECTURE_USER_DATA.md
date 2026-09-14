# User data architecture — design before implementation

**Status:** design accepted, implementation blocked on restoring the Phase 2 tree
(see [`../RECOVERY.md`](../RECOVERY.md)).
**Scope:** priorities 2–4 of the product plan — user/profile/session architecture,
activity tracking, anti-fraud foundation. Priority 5 (wallet ledger) is designed
here but deliberately not scheduled; see §7.

---

## 1. What the current architecture actually is

This section records only what is **verifiable from surviving evidence** —
migration `0011_plane_separation.sql`, `internal/plane/plane.go`, and the three
plane entrypoints. Everything unverifiable is marked as such rather than
reconstructed from memory.

### 1.1 Process and privilege model

Three binaries, three containers, three database roles. The split is a capability
boundary, not a network one:

| Binary | Role bundle | Holds |
| --- | --- | --- |
| `cmd/user-api` | `tgr_user_runtime` | `users`, `user_sessions`, INSERT on `audit_events` |
| `cmd/admin-api` | `tgr_admin_runtime` | admin identity + WebAuthn tables, SELECT/INSERT on `audit_events` |
| `cmd/webhook` | `tgr_webhook_runtime` | `telegram_updates`, `telegram_events`, INSERT on `audit_events` |

Supporting roles: `tgr_operator` (`cmd/adminctl`), `tgr_bootstrap` (one-shot
first-admin capability, self-revoking), `tgr_app` (login role, member of a
bundle). The retired `tgr_runtime` bundle is stripped to zero privileges.

Two properties of this model constrain everything below:

- **Privileges are column-scoped where they need to be.** PostgreSQL *unions*
  table and column grants, so a later table-wide `GRANT UPDATE` silently undoes a
  narrow one. Every new table must state its grants at column granularity if any
  column is sensitive.
- **The audit trail is append-only and hash-chained**, enforced by a trigger
  calling `audit_chain_field(text)`. Nested PL/pgSQL calls *are* privilege-checked,
  so every role that appends needs `EXECUTE` on that function explicitly.

### 1.2 Known tables

`users`, `user_sessions`, `audit_events`, `telegram_updates`, `telegram_events`,
`schema_migrations`, and the `admin_*` family. Migrations run 0001–0015 under a
forward-only checksummed migrator, so new work starts at **0016**.

### 1.3 Known application packages

`internal/`: `audit`, `auth`, `config`, `httpx`, `plane`, `ratelimit`, `security`,
`storage/redisx`, `telegram/initdata`, `telegram/webhook`, `userapi`, `adminapi`,
`users`.

Session handling is `auth.NewSessionStore(db, AbsoluteTTL, IdleTTL, MaxActivePerUser)`
— absolute expiry, idle expiry, and a per-user active-session cap already exist.
`users.NewStore(db)` upserts on first sight and reads thereafter.

### 1.4 What is NOT known and must be reconciled first

The **column definitions of `users` and `user_sessions`** are not recoverable.
`0001_foundation.sql` was not among the salvaged files. Specifically unknown: the
primary key type of `users`, whether `telegram_user_id` is the PK or a unique
secondary key, and which timestamp columns already exist.

**This is why no migration SQL appears in this document.** Writing
`REFERENCES users(id)` without knowing whether that column is `uuid` or `bigint`
is guesswork, and guesswork in a foreign key is not backward compatible. §8 gives
the exact reconciliation step.

---

## 2. Extended user profile

### 2.1 Decision: widen `users`, do not add a `user_profiles` side table

At one million rows, `users` is a small table. A separate profile table would add
a join to the hot authentication path to save storage that does not need saving.
Profile attributes are read on nearly every request that reads the user at all.

New columns, all nullable or defaulted so the migration is backward compatible and
no existing insert path breaks:

**Telegram identity** (refreshed from `initData` on login, not trusted as stable):
`telegram_username`, `telegram_first_name`, `telegram_last_name`,
`telegram_language_code`, `telegram_is_premium`, `telegram_photo_url`.

**Account metadata:** `first_seen_at`, `last_seen_at`, `registration_source`,
`account_status`.

Three points that are easy to get wrong:

- **Telegram-supplied fields are mutable and attacker-influenced.** A username can
  be changed or released and re-registered by someone else. Nothing may key off
  `telegram_username`; the numeric `telegram_user_id` is the only stable identity.
- **`telegram_is_premium` is a claim in `initData`, not a verified entitlement.**
  It is fine for presentation. It must not gate anything of value without a
  server-side check against the Bot API.
- **`account_status`** is a lookup-table FK, not a boolean and not an enum. It will
  grow values (`active`, `dormant`, `restricted`, `closed`) and Postgres enum
  alteration is a migration each time.

### 2.2 Retention

`telegram_first_name`, `telegram_last_name`, and `telegram_photo_url` are personal
data with no operational use beyond rendering a profile header. They are refreshed
on every login, so they need no history. The design stores the current value only
and never journals changes to them — the cheapest privacy posture is not collecting
the history in the first place.

---

## 3. Localization

Required: English, German, Persian, Russian. The instruction was not to hardcode
language logic, so the shape matters more than the four values.

### 3.1 Decision: split UI strings from content translations

These are different problems and putting them in one place makes both worse.

**UI strings live in the frontend bundle**, one JSON file per locale. They are
needed at first paint, they version with the frontend, and serving them from
Postgres would add a round-trip to every cold start for no benefit.

**Server-owned content** — reward names, ad titles, and later mission and gift
copy — lives in **one generic table**, not a `*_i18n` table per entity:

```
content_translations(entity_type, entity_id, locale, field, value)
PRIMARY KEY (entity_type, entity_id, locale, field)
```

One table means adding a translatable entity is an INSERT, not a migration. The
cost is that referential integrity to the target entity cannot be expressed as a
foreign key; a periodic orphan sweep in the maintenance worker covers it. That
trade is worth taking — the alternative is a new table and new code path per
entity type forever.

### 3.2 Decision: a `locales` table, not a constant

```
locales(code, display_name, is_enabled, is_rtl, fallback_code, sort_order)
```

Adding Arabic or Turkish becomes an INSERT plus a frontend bundle, with no schema
change and no Go constant. `fa` carries `is_rtl = true`, which the frontend reads
to set `dir="rtl"` — Persian is a layout concern, not just a string concern, and
building for it now is far cheaper than retrofitting.

### 3.3 Decision: separate what the user chose from what Telegram said

Two columns, deliberately:

- `telegram_language_code` — raw, as Telegram sent it, overwritten every login.
- `language_code` — the user's effective preference.

If these were one column, every login would overwrite an explicit in-app language
choice with the user's Telegram client language. `language_code` is set from
`telegram_language_code` **only when it is still null**, and thereafter only by the
user. Resolution at read time: `language_code` → `locales.fallback_code` chain →
default locale, stopping at the first enabled locale.

---

## 4. Session and device information

The instruction was to support security, fraud detection and account protection
**without storing excessive fingerprinting data**. Those pull in opposite
directions, so the rule below is stated as a boundary rather than a preference.

### 4.1 Decision: store parsed, coarse attributes — never the raw artifact

On `user_sessions`:

| Column | Source | Why coarse |
| --- | --- | --- |
| `ip_address inet` | connection, via existing `ClientIP` middleware | full value, but see §4.3 |
| `ip_country`, `ip_city` | GeoIP lookup | city nullable; "if available" |
| `asn` | GeoIP lookup | the useful unit for session-hijack detection |
| `client_timezone` | client-reported | advisory only, trivially spoofed |
| `device_type` | parsed from UA | `mobile` / `tablet` / `desktop` / `unknown` |
| `os_family`, `browser_family` | parsed from UA | family only, not version strings |
| `tg_platform`, `tg_version` | Telegram WebApp | already supplied, no extra collection |

**The raw `User-Agent` string is parsed and discarded.** The parsed triple is what
security work actually uses; the raw string is a high-entropy identifier we would
then be obliged to protect.

### 4.2 Explicitly out of bounds

The following must not be added, and this list exists so that a future "we need it
for fraud" change has to argue against a written decision rather than fill a gap:
canvas, WebGL and audio fingerprints; font enumeration; plugin or MIME-type lists;
precise screen dimensions; battery status; any third-party fingerprinting SDK.

### 4.3 Device grouping without device identification

Detecting "repeated device patterns" needs some notion of sameness. The design
uses `device_group_hash` = HMAC(server key, `os_family` ‖ `browser_family` ‖
`device_type` ‖ `tg_platform`).

This is **deliberately low-entropy**. It cannot identify a device — millions of
users share "android ‖ chrome ‖ mobile ‖ android". It is useful only in
conjunction with a network signal (the `/24` for IPv4, `/48` for IPv6, or the ASN),
and that combination is what the fraud rules in §6 consume. A high-entropy device
ID would detect more fraud and would also be a tracking identifier; the coarse hash
is the choice that keeps the fraud signal while making the data not worth stealing.

Retention: the full `ip_address` lives on the session row for the session's
lifetime plus a short window. Long-lived `fraud_signals` keep only the network
prefix and ASN, never the full address.

---

## 5. Activity tracking

### 5.1 Decision: one append-only semantic event stream, monthly partitioned

```
user_events(id bigint, user_id, session_id, event_type_id smallint,
            occurred_at timestamptz, attributes jsonb)
PARTITION BY RANGE (occurred_at)
```

At 300k DAU and roughly 20 meaningful events per user-day this is ~6M rows/day and
~180M rows/month. Two consequences drive the shape:

- **Partition monthly.** Retention then becomes `DETACH PARTITION` + `DROP`, which
  is instant, instead of a `DELETE` over hundreds of millions of rows that would
  bloat the table and saturate autovacuum.
- **`event_type_id` is a `smallint` FK to a lookup table**, not a text column and
  not a Postgres enum. Text repeats the label 180M times a month; an enum makes
  adding a value a migration.

Events are **semantic, not per-request**: `app_opened`, `session_started`,
`ad_view_started`, `ad_view_completed`, `reward_redeemed`. There is no event per
HTTP call.

Login history is a **view over this stream**, not a second table. One append-only
stream with many projections beats several tables that can disagree.

### 5.2 Decision: throttle `last_seen_at` through Redis

This is the single most important write-amplification decision in the design.

Updating `users.last_seen_at` on every request at 300k DAU × ~50 requests is 15M
row updates per day on the hottest table in the system. In PostgreSQL every update
writes a new row version, so that is 15M dead tuples per day, continuous autovacuum
pressure, index bloat, and WAL volume — to maintain a timestamp whose precision
nobody needs better than a few minutes.

Instead, gate it: `SET user:seen:<id> 1 NX EX 300`. If the key was set, issue the
UPDATE; otherwise skip. That is at most one update per user per five minutes —
roughly a 50× reduction — and the timestamp is still accurate to within the window.

The same reasoning applies to `user_sessions.last_seen_at`, which the existing idle
expiry depends on. The throttle window must be **strictly shorter than the idle
TTL**, or sessions will expire while in use. That constraint belongs in a test, not
in a comment.

---

## 6. Anti-fraud foundation

The instruction was explicit: **detect first, punish later.** The design honours
that literally — nothing in this section can block, restrict, or deny a user.

### 6.1 Decision: append-only signals, asynchronously derived scores

```
fraud_signals(id, user_id, signal_type_id, severity, observed_at, evidence jsonb, source)
user_risk_scores(user_id, score, band, computed_at, model_version)
```

`fraud_signals` is append-only and never updated — a signal is an observation, and
observations do not change. `user_risk_scores` holds one row per user, **recomputed
by a background worker**, never inline in a request. Scoring in the request path
would put a variable-cost analytical query in front of a user waiting on an API
call, which violates the rule against blocking requests with heavy operations.

`model_version` is on the score row so that when the scoring rules change, scores
computed under the old rules are identifiable and can be recomputed rather than
silently compared against new thresholds.

### 6.2 Initial signal types

Covering the four patterns named in the requirements:

| Signal | Detects |
| --- | --- |
| `account_creation_velocity` | many accounts from one network prefix in a short window |
| `reward_request_rate` | reward actions far above the population distribution |
| `session_context_shift` | country or ASN change within one session's lifetime |
| `device_group_collision` | many accounts sharing `device_group_hash` + network prefix |

### 6.3 Decision: no enforcement in this phase

No `users.account_status` check is wired into any request path as part of this
work. Enforcement is a separate, separately-reviewed change, for two reasons:

1. **Measurement before consequence.** Every one of these rules will fire on
   legitimate users — shared carrier NAT alone will trigger
   `account_creation_velocity` and `device_group_collision` constantly. Running
   detection against real traffic first tells us the false-positive rate. Shipping
   blocking with an unmeasured rule means locking out real users to catch fraud we
   have not yet proven exists at volume.
2. **Enforcement is a privileged capability.** The ability to restrict an account
   belongs behind the existing admin dual-control machinery (migrations 0013/0014),
   not behind a background worker's threshold.

---

## 7. Coin wallet — designed now, deliberately not scheduled

Recording the invariants now prevents them being improvised later under delivery
pressure. **Implementation is gated; see §9.**

### 7.1 Ledger, never a balance column

```
wallets(id, user_id, currency, balance_cached, entry_seq, created_at)
wallet_entries(id bigint, wallet_id, amount bigint, reason_id, reference_type,
               reference_id, idempotency_key, created_at)
```

- `wallet_entries` is **append-only**. No UPDATE, no DELETE, enforced by trigger
  and by withholding the privilege — the same belt-and-braces the audit trail uses.
- `amount` is a **signed integer in minor units**. Never a float. Credits are
  positive, debits negative, and the balance is `SUM(amount)`.
- `balance_cached` is an **optimisation, not the truth**. It is updated in the same
  transaction as the entry insert, and a reconciliation job asserts
  `balance_cached = SUM(amount)` per wallet on a schedule. When they disagree, the
  ledger wins and the cache is rebuilt.
- `currency` exists from day one even though the only value will be `COIN`, so that
  Stars or any other unit does not require a schema change.

### 7.2 Idempotency is mandatory, not advisory

`UNIQUE (wallet_id, idempotency_key)`.

Ad-network completion callbacks retry. Without this constraint a retried callback
mints coins, and that is the single most likely way this system loses money. The
constraint — not application logic — is what makes the credit exactly-once.

### 7.3 Balance floor

A `CHECK` cannot span rows, so non-negativity cannot be a column constraint. The
debit path takes `SELECT ... FOR UPDATE` on the wallet row, re-reads the balance
inside that lock, and refuses if the debit would go below zero. The lock is on the
wallet, so it serialises only that user's writes.

### 7.4 `ADMIN_ADJUSTMENT` is a privileged operation

Manual balance adjustment is the one reason code that mints value from nothing. It
must route through the existing admin proposal/approval tables (0013/0014), require
dual control, and write to `audit_events`. It must **not** be reachable from
`tgr_user_runtime`.

Note for Phase 3: the round-4 review adopted the invariant that `tgr_admin_runtime`
must not become a general financial-administration role. Wallet write privileges
therefore belong to a dedicated bundle, not to the existing admin plane.

---

## 8. Migration plan

Forward-only, each independently applicable, each backward compatible — every new
column is nullable or defaulted and no existing insert path is modified.

| # | Migration | Contents |
| --- | --- | --- |
| 0016 | `locales` | `locales` table, seed `en`/`de`/`fa`/`ru`, `content_translations` |
| 0017 | `user_profile` | widen `users`; `account_statuses` lookup; grants |
| 0018 | `session_device` | widen `user_sessions` with §4.1 columns; grants |
| 0019 | `user_events` | `event_types` lookup; partitioned `user_events`; initial partitions |
| 0020 | `fraud_signals` | `fraud_signal_types`, `fraud_signals`, `user_risk_scores` |
| — | *(gated)* | `wallets`, `wallet_entries`, `wallet_reasons` |

Each migration must also `GRANT` at column granularity to the correct bundle, and
`GRANT EXECUTE ON FUNCTION audit_chain_field(text)` to any new role that appends to
the audit trail — see §1.1.

### 8.1 Required reconciliation before 0016 is written

Against the restored tree:

1. Read `0001_foundation.sql` for the real definitions of `users` and
   `user_sessions` — PK type, existing timestamp columns, existing indexes.
2. Read `internal/users/store.go` and `internal/auth/session_store.go` for the
   insert and update paths that must keep working unchanged.
3. Confirm the highest applied migration number (expected 0015).
4. Confirm the naming and constraint conventions in use, and follow them rather
   than the placeholders above.

### 8.2 New process

Risk scoring, partition maintenance, orphan sweeps and balance reconciliation all
need to run outside the request path. They belong in one new `cmd/worker` binary
following the existing `internal/plane` pattern, with its own `tgr_worker_runtime`
bundle. No Kubernetes, no queue broker — a single process with a ticker, consistent
with the instruction to keep the architecture simple but scalable.

---

## 9. What is blocked, and why

**Implementation of §2–§6 is blocked** on restoring the Phase 2 tree. The reason is
narrow and specific: §8.1. Nothing else is missing.

**Implementation of §7 is separately gated.** The Phase 2 security gate stood at
**BLOCKED** at the end of round 5 — not on any open implementation defect, but on
absent evidence: CI had never executed the test suite, no hardware WebAuthn drill
had been performed, and `govulncheck` could not reach its vulnerability database.
All five independent reviews concluded that coin, ad-provider, gift and premium
functionality should not begin until that evidence exists.

That gate does not block §2–§6. Profile, localization, session metadata, activity
events and fraud *detection* are not financial logic and hold no value an attacker
can extract. They are the correct thing to build first, which is also the order the
product plan specifies.
