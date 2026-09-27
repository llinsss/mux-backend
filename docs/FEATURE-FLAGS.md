# Feature Flags

This document describes the feature-flag and kill-switch model used across
mux-backend, with a focus on the **wallet orchestrator** path. It is the
canonical reference for the e2e coverage in
`test/wallet-orchestrator-feature-flag.e2e-spec.ts`.

Related docs:

- `docs/AUTH-FEATURE-FLAGS.md` — authz roles (owner/delegate/guardian/API-key/JWT).
- `docs/MAINNET-PAYMENT-FEATURE-FLAG.md` — mainnet money-path gating.

## Model

Flags are resolved server-side and are the **source of truth**. Clients cannot
enable a privileged surface by sending a flag value; any client-supplied flag is
ignored and the server resolves the effective value from configuration.

Resolution order (first match wins):

1. **Kill-switch** — if a kill-switch is engaged, the surface is disabled
   regardless of any other flag. Kill-switches are deny-by-default and fail
   closed.
2. **Environment default** — per-environment default (testnet vs mainnet).
3. **Explicit override** — operator-set override, audited and logged.

If flag resolution fails (config store outage, malformed value, unknown flag
name), the surface is treated as **disabled** (fail closed).

## Wallet orchestrator flags

| Flag | Default (testnet) | Default (mainnet) | Notes |
| --- | --- | --- | --- |
| `wallet.orchestrator.enabled` | `true` | `false` | Master switch for the orchestrator path. |
| `wallet.orchestrator.aa.enabled` | `true` | `false` | Account-abstraction operations. |
| `wallet.orchestrator.payments.enabled` | `true` | `false` | Money-path operations; see mainnet doc. |

Mainnet defaults are `false`; enabling any mainnet money-path flag requires the
readiness checklist in `docs/MAINNET-PAYMENT-FEATURE-FLAG.md`.

## Behavior contract (e2e)

The e2e suite asserts the following invariants:

- **Disabled orchestrator** — requests to orchestrator entrypoints return a
  stable error code (`WALLET_ORCHESTRATOR_DISABLED`) and do not mutate state.
- **Kill-switch precedence** — engaging the kill-switch disables the surface even
  when the environment default is `true`.
- **Fail closed** — a flag-resolution failure yields the disabled behavior, not
  an enabled one.
- **Authz is independent of flags** — a disabled flag denies everyone; an enabled
  flag still enforces owner/delegate/guardian/API-key/JWT policy. Flags never
  grant access.
- **Idempotency** — replayed requests with the same idempotency key return the
  same result and do not double-apply.
- **Correlation ids** — every response carries a correlation id; logs include it
  and never include secrets, raw key material, JWTs, or webhook secrets.

## Error codes

| Code | Meaning |
| --- | --- |
| `WALLET_ORCHESTRATOR_DISABLED` | Orchestrator flag resolved to disabled. |
| `WALLET_ORCHESTRATOR_AA_DISABLED` | AA flag resolved to disabled. |
| `WALLET_ORCHESTRATOR_PAYMENTS_DISABLED` | Payments flag resolved to disabled. |
| `FEATURE_FLAG_RESOLUTION_FAILED` | Flag store unavailable or malformed; fail closed. |

## Observability

- Metrics: flag resolution outcome (enabled/disabled/failed) per flag, and
  denied-request counts per error code.
- Logs: include flag name, resolved value, correlation id, and actor role.
  Redact secrets, keys, JWTs, and webhook secrets.

## Rollback

Disabling a flag (or engaging its kill-switch) is the rollback path. Rollback is
safe and immediate; no data migration is required. Document the flag and
kill-switch used in the PR description for any money-path or mainnet-affecting
change.
