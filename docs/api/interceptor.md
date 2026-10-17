# PreFlightInterceptor API

PreFlightInterceptor evaluates whether a planned contract call will be accepted by the guard contract.

## Constructor

```ts
constructor(options: PreFlightInterceptorOptions)
```

### Options

`PreFlightInterceptorOptions` constructor options:

| Option | Type | Default | Semantics |
| --- | --- | --- | --- |
| `server`? | `rpc.Server` | required | Soroban RPC server instance used to simulate the guard check. |
| `networkPassphrase? | `string` | required | Stellar network passphrase for transaction building and signing. |
| `guard?` | `string` | required | Custom account contract address (`C...`) that implements the guard check. |
| `agent?` | `Keypair` | required | Keypair registered as the agent in the guard. |
| `source? | `Keypair` | required | Keypair paying for transaction fees. |
| `cache?` | `PreFlightCacheOptions` | disabled | Optional short-lived verdict cache. ⚠ Enabling the cache means a verdict may be stale relative to the latest chain state; see below. |

All options are required except `cache`. The constructor does not perform any network I/O and does not validate that `guard` exists on-chain; the first call to `check` or `assertAllowed` will surface such errors as an `undetermined` verdict.

### Optional cache

`PreFlightCacheOptions`:

| Option | Type | Default | Semantics |
| --- | --- | --- | --- |
| `ttlMs`? | `number` | unset | Cache entry lifetime in milliseconds. Setting this or `ttlLedgers` opts into caching. |
| `ttlLedgers?` | `number` | unset | Cache entry lifetime in ledgers. Setting this or `ttlMs` opts into caching. |
| `policyRevision? | `string | (() => string | Promise<string>)` | unset | Opaque revision label or async getter; changing it invalidates matching entries. |

Set either `cache.ttlMs` or `cache.ttlLedgers` to opt in. The effective TTL is
capped at one approximate five-second ledger-close interval, and a cached entry
is discarded when the observed ledger advances. `policyRevision` may be a value
or an async getter; changing it invalidates the matching entry. Call
`invalidate(call?)` after policy or account-state changes. ⚠ A cached verdict can
be staler than one admitted transfer, so callers must accept that tradeoff
explicitly. Disabling the cache (the default) is the strictest safety posture:
every check reflects the latest simulated state.

## Methods

### `check(call: ContractCall): Promise<PreFlightDecision>

Evaluates a contract call against the guard without throwing or broadcasting.

### `invalidate(call?: ContractCall): void`

Clears all cached decisions, or only the entries for the supplied call.

### `assertAllowed(call: ContractCall): Promise<AdmissibleDecision>`

Evaluates a call and throws `GuardBlockedError` if blocked or undetermined.

## CheckResult decoding and fallback behavior

The interceptor reads the guard contract's `CheckResult` value from the
simulation response. The contract returns an enum with the following shapes:

- `Allowed` — a unit variant with no payload.
- `Blocked(reason)` — a variant carrying a symbol reason (e.g. `recipient_not_allowed`,
  `per_tx_cap_exceeded`, `paused`).

`decodeCheckResult` accepts either the raw `xdr.ScVal` the contract returns
(for example `simulation.returnValue`) or the native value `scValToNative`
produces from it, and validates a `Blocked` reason against the contract's own
reason vocabulary.

Decoding is fail-closed. If the returned ScVal is not a recognized
`CheckResult` shape — an unknown enum tag, a `Blocked` reason the contract does
not define, an empty or over-long `ScVec`, a blocked variant whose reason is not
a symbol, or a non-`ScVec` input — the decoder does not throw. Instead it
returns the typed fallback for the call-site contract:

| Call site | Fallback verdict | Notes |
| --- | --- | --- |
| `PreFlightInterceptor.check` | `undetermined` with `null` reason | The caller sees a typed decision, not an exception. |
| `assertAllowed` | `GuardBlockedError` with an `undetermined` verdict | The error is thrown by the caller-facing assertion, not by the decoder. |

In other words, a malformed or future `CheckResult` never escapes to the agent
loop as a raw throw. The interceptor treats any decode failure as
undetermined`, which is the fail-closed result.
