# CostPreChecker API

`CostPreChecker` evaluates expected transaction fees against an optional ceiling before broadcast, and optionally exposes additive policy-relative context (`policyContext`).

## Constructor

```ts
constructor(options: CostPreCheckConfig)
```

### Options

| Option | Type | Default | Semantics |
| ------ | ---- | ------- | --------- |
| `interceptor` | `Pick<PreFlightInterceptor, "check">` | required | Configured interceptor instance whose simulation prices execution. |
| `maxFeeStroops` | `bigint` | undefined (no ceiling) | Maximum allowable total fee in stroops. ⚠ When omitted, the checker prices the call but never objects on cost — fail-closed cost guarding requires an explicit ceiling. |
| `policySource` | `string | PolicyConfig | null` | `undefined` (disabled) | Opt-in policy source (contract address or `GuardPolicy`/`PolicyConfig`) enabling additive `policyContext`. ⚠ When disabled, `policyContext` is `null` and no policy-relative signals are reported. |
| `policy` | `string | PolicyConfig | null` | `undefined` (disabled) | Alias for `policySource`. |

## Methods

### `check(call: ContractCall, options?: { policySource?: string | PolicyConfig | null; maxFeeStroops?: bigint }): Promise<CostPreCheckResult>`

Returns `CostPreCheckResult` (`CostDecision`), a discriminated union:

- `{ kind: "within_budget", allowed: true, totalFeeStroops, resourceFeeStroops, inclusionFeeStroops, footprintKeys, feeCeilingStroops, policyContext }`
- `{ kind: "over_budget", allowed: false, totalFeeStroops, resourceFeeStroops, inclusionFeeStroops, footprintKeys, feeCeilingStroops, policyContext }`
- `{ kind: "blocked", allowed: false, reason, explanation, detail, totalFeeStroops: 0n, resourceFeeStroops: 0n, inclusionFeeStroops: 0n, policyContext }`
- `{ kind: "undetermined", allowed: false, detail, totalFeeStroops: 0n, resourceFeeStroops: 0n, inclusionFeeStroops: 0n, policyContext }`

### Policy Context (`policyContext`)

When an opt-in policy source is configured, `policyContext` exposes additive policy context derived from the live `check()` call:

```ts
policyContext: {} | {
  perTxCapOk: boolean | null;
  windowRemainingEstimate: number | null;
  reason: string | null;
} | null
```

- `perTxCapOk`: Whether the transaction fits the policy's per-transaction limit (`true` if within cap, `false` if per-tx cap exceeded, or `null` if not determinable).
- `windowRemainingEstimate`: Estimated remaining headroom in the current rolling window. **Always `null` if the contract does not expose sufficient window state.** Note that `null` means "not available / cannot be determined from the current contract state", **not** a zero remaining budget. The SDK never fabricates a zero budget.
- `reason`: The contract's policy block reason if blocked (e.g. `bwindow_cap_exceeded` or `per_tx_cap_exceeded`), or `null` when allowed.
- When no policy source is configured, `policyContext` is `null`.
