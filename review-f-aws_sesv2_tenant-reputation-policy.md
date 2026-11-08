# Review: `f-aws_sesv2_tenant-reputation-policy`

One commit (`8d8513cd45b feat(sesv2): add reputation_entity_policy to aws_sesv2_tenant`), 4 files:

- `internal/service/sesv2/tenant.go`
- `internal/service/sesv2/tenant_test.go`
- `internal/service/sesv2/exports_test.go`
- `website/docs/r/sesv2_tenant.html.markdown`

Verified: `go build` and `go vet` pass on `internal/service/sesv2`. Acceptance tests not run (need AWS credentials).

Overall shape is good: schema/model alphabetized, `fwtypes.ARNType` custom type, Optional+Computed+`UseStateForUnknown` is the right combination since AWS applies a default policy, and the test covers create, import, and an update (`standard` → `strict`).

## Findings

### 1. `Read` (and the `Create` read-back) treat a missing reputation entity as a hard error

`tenant.go:149-153`, `tenant.go:113-117`

If `GetReputationEntity` returns `NotFoundException`, `findReputationEntityPolicyByReference` returns a `retry.NotFoundError`, which `Read` turns into an error diagnostic. Two risks:

- Eventual consistency: the read-back in `Create` runs immediately after `CreateTenant`. If the reputation entity hasn't propagated, create fails after the tenant already exists (tainted resource).
- If AWS ever reports no entity for a tenant, every refresh errors instead of drifting to null.

Suggested shape for `Read`:

```go
policy, err := findReputationEntityPolicyByReference(ctx, conn, fwflex.StringValueFromFramework(ctx, state.TenantARN))
if retry.NotFound(err) {
	state.ReputationEntityPolicy = fwtypes.ARNNull()
} else if err != nil {
	smerr.AddError(ctx, &resp.Diagnostics, err, smerr.ID, name)
	return
} else {
	state.ReputationEntityPolicy = fwflex.StringToFrameworkARN(ctx, policy)
}
```

Worth confirming against the API whether a reputation entity always exists for a tenant — if it does, a short retry on the `Create` read-back would still be prudent.

### 2. Missing CHANGELOG entry

The diff has no `.changelog/<pr>.txt`. This is a user-facing enhancement, so it needs one, e.g.:

````
```release-note:enhancement
resource/aws_sesv2_tenant: Add `reputation_entity_policy` argument
```
````

### 3. Docs: `reputation_entity_policy` is listed in both Argument Reference and Attribute Reference

`website/docs/r/sesv2_tenant.html.markdown`

Per the docs convention, Attribute Reference is "in addition to the arguments above" — only computed-only attributes belong there. Since it's already documented as an argument, drop the Attribute Reference entry.

### 4. Unused export

`exports_test.go:32`

`FindReputationEntityPolicyByReference` is exported but the test never calls it (assertions go through `statecheck`). Either drop the export or use it in a check — e.g. verifying the applied policy via the API in a `testAccCheckTenantExists`-style helper would strengthen the test.

### 5. No way to unset the policy (worth a doc note)

With Optional+Computed+`UseStateForUnknown`, removing `reputation_entity_policy` from the config keeps the prior state value — there's no diff, so `Update` never resets it. That's inherent to the pattern and fine, but since `none` is a valid policy, consider adding a sentence to the argument docs: removing the argument does not reset the policy; set the `none` policy ARN explicitly.

### Nits

- `Update` re-fetches the policy via `GetReputationEntity` even when nothing changed (e.g. a tags-only update). Minor extra API call — could skip the read-back when `plan.ReputationEntityPolicy.Equal(state.ReputationEntityPolicy)`.
- The helpers convention wraps finder not-found errors as `smarterr.NewError(&retry.NotFoundError{...})`; the new finder returns it bare. That matches the existing `findTenant` in the same file, so consistency-wise it's defensible.

### Verified as *not* problems

- Tag updates still work: `ResourceWithModel` embeds `withNoOpUpdate`, and tags are handled by the transparent tagging interceptor, not the resource's `Update` — so overriding `Update` here doesn't break tag-only updates.
- `fwflex.StringToFrameworkARN(ctx, nil)` safely produces a null ARN if AWS ever returns a nil policy.

## Design discussion

### Should this be a separate resource?

No — the inline attribute is the right shape. The reputation entity API surface is:

```
GetReputationEntity
ListReputationEntities
UpdateReputationEntityPolicy
UpdateReputationEntityCustomerManagedStatus
```

There is no Create and no Delete. A reputation entity auto-exists for every tenant, can't be removed, always has a policy (AWS applies a default), is strictly 1:1 with the tenant, and is keyed by the tenant's own ARN. That maps naturally to a mutable property, not a standalone object.

A separate resource would be awkward:

- `Delete` would have to fabricate semantics ("reset to the `none` policy ARN") for a thing that can't be deleted.
- No independent identity to import or manage.
- A whole resource (CRUD, identity/import, docs, tests) for one scalar with three possible values.

The case for separate (for the record): the package already follows the granular pattern (`aws_sesv2_tenant_resource_association`, `aws_sesv2_email_identity_policy`) — but both of those have real create/delete lifecycle, which is exactly what the reputation entity lacks. A separate resource would give real removal semantics and would give `customer_managed_status` (the other `Update*` operation) a natural home later. If a separate resource is ever added, the inline attribute becomes a dueling-management problem (cf. `aws_security_group` inline rules vs. `aws_vpc_security_group_rule`), so decide consciously rather than drifting into it. Never do both.

### Should the value be a policy name instead of an ARN?

Keep the ARN. Reasons:

1. **Lossless round-trip.** The API takes an ARN on write and returns an ARN on read. Since the attribute is `Computed`, the read-back goes straight into state. A name-based attribute would have to parse the ARN back to a name on read, and anything not matching the expected pattern (future custom policy, new managed policy) can't round-trip.
2. **Provider convention.** AWS-managed policy ARNs are passed through as ARNs everywhere else (cf. `aws_iam_role_policy_attachment.policy_arn`).
3. **Future-proofing.** The API models this as an ARN, not an enum — AWS left the door open to customer-managed policies. If that ships, the ARN attribute just works; a `stringvalidator.OneOf("none", "standard", "strict")` becomes an awkward schema change.
4. **No translation bugs.** A name-based version must construct the ARN from partition + the *resource's* region (`plan.Region`), not the provider's — an avoidable cross-region failure mode.

The honest cost is config ergonomics (users need partition/region data sources to build the ARN). The `arn_build` provider function is a possible docs addition:

```terraform
reputation_entity_policy = provider::aws::arn_build(
  data.aws_partition.current.partition,
  "ses",
  data.aws_region.current.region,
  "aws",
  "reputation-policy/standard",
)
```
