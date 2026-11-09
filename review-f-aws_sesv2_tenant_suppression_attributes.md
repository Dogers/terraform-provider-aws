# Review: `f-aws_sesv2_tenant_suppression_attributes`

One commit (`40aa5260fca feat(sesv2): new resource aws_sesv2_tenant_suppression_attributes`), 5 files:

- `internal/service/sesv2/tenant_suppression_attributes.go` (new)
- `internal/service/sesv2/tenant_suppression_attributes_test.go` (new)
- `internal/service/sesv2/exports_test.go`
- `internal/service/sesv2/service_package_gen.go` (generated)
- `website/docs/r/sesv2_tenant_suppression_attributes.html.markdown` (new)

Verified: `go build` and `go vet` pass on `internal/service/sesv2`. Acceptance tests not run (need AWS credentials).

Overall: the separate-resource design is the right call here — it mirrors `aws_sesv2_account_suppression_attributes`, there's a dedicated `PutTenantSuppressionAttributes` API, and `WithNoOpDelete` is appropriate since no delete API exists. The code follows recent patterns (smerr throughout, enum custom types, framework model) rather than copying the legacy style of the account-level resource. Generated `service_package_gen.go` matches the annotation.

## Findings

### 1. Missing resource identity (required for new resources)

Every new resource declares one identity strategy via annotation. This resource has none — it hand-rolls `ImportState` with passthrough to `tenant_name`. Should be:

```go
// @FrameworkResource("aws_sesv2_tenant_suppression_attributes", name="Tenant Suppression Attributes")
// @IdentityAttribute("tenant_name")
func newTenantSuppressionAttributesResource(context.Context) (resource.ResourceWithConfigure, error) {
```

with `framework.WithImportByIdentity` embedded (single-attribute parameterized identity, so no `@ImportIDHandler` needed), replacing the custom `ImportState`.

The docs Import section then needs the full identity structure, in this order:

1. Lead-in `In Terraform v1.12.0 and later, the [import block](...)` with an `identity = { tenant_name = "example-tenant" }` example.
2. `### Identity Schema` with `#### Required` (`tenant_name` bullet) and `#### Optional` (the verbatim `account_id` / `region` bullets).
3. The existing v1.5.0 `import` block example.
4. The existing `terraform import` console example.

### 2. `suppressed_reasons` is Optional-only — perpetual-diff risk

`tenant_suppression_attributes.go` schema. The account-level equivalent (`aws_sesv2_account_suppression_attributes`) has it `Required`. As Optional-only: if the user omits it and AWS populates/defaults reasons server-side, `Read` stores values the config never set → diff on every plan → `Update` → repeat. Either:

- make it `Required` (matches the account-level resource), or
- make it Optional+Computed with `setplanmodifier.UseStateForUnknown()`.

Also worth verifying/documenting how `suppressed_reasons` interacts with `suppression_scope = "ACCOUNT"` — if reasons are only meaningful at `TENANT` scope, a note in the docs (or a validator) would help.

### 3. Missing `_disappears` test

The required pair is `_basic` + `_disappears`; this branch has `_basic` and `_update`. Here a disappears test is meaningful: delete the parent `aws_sesv2_tenant` out-of-band, then expect this resource to be removed from state (`Read` already handles NotFound → `RemoveResource`). `acctest.CheckDestroyNoop` is fine given `WithNoOpDelete`.

### 4. Missing CHANGELOG entry

New resources need `.changelog/<pr>.txt`:

````
```release-note:new-resource
aws_sesv2_tenant_suppression_attributes
```
````

### 5. Create/Update read-back bypasses the finder

Create and Update call `findTenantByName` and flatten `out.SuppressionAttributes` directly. If that's nil (eventual consistency right after `PutTenantSuppressionAttributes`), you're flattening a nil pointer. `Read` uses `findTenantSuppressionAttributesByName`, which nil-checks via `tfresource.NewEmptyResultError()`. Use the finder in Create/Update too — consistent and gets the empty-result handling for free.

### Nits

- `Update` always calls Put without gating on changes. Acceptable for a Put-only API (matches the account-level pattern), but `flex.Diff` + `HasChanges()` would avoid no-op API calls.
- `_basic` could add `ConfigPlanChecks` with `plancheck.ExpectResourceAction(resourceName, plancheck.ResourceActionCreate)`, like `TestAccSESV2Tenant_basic`.
- `suppressed_reasons` sets both `CustomType` and `ElementType` — redundant, but matches the account-level resource, so fine.

### Verified as *not* problems

- `WithNoOpDelete` is correct — there is no `DeleteTenantSuppressionAttributes` API.
- `FindTenantSuppressionAttributesByName` export is actually used by the test's Exists check.
- Docs: frontmatter/description opens with "Manages", required-then-optional argument structure, valid values listed, no forbidden description openings.
- Enum types (`fwtypes.SetOfStringEnumType`, `fwtypes.StringEnumType`) match SDK enums; test configs use the correct uppercase values.

## Re-check (commit `d5478aeab46` — "address review feedback")

Verified: `go build` and `go vet` pass on `internal/service/sesv2`.

| Finding | Status |
|---|---|
| 1. Missing resource identity | **Resolved.** `@IdentityAttribute("tenant_name")` + `framework.WithImportByIdentity` replace the hand-rolled `ImportState`; `service_package_gen.go` shows the matching `RegionalSingleParameterIdentity`; generated identity tests (`tenant_suppression_attributes_identity_gen_test.go`) and testdata templates are present and look standard. Docs have the full identity import structure (v1.12.0 identity block → Identity Schema with Required `tenant_name` and the verbatim `account_id`/`region` Optional bullets → v1.5.0 import block → `terraform import`). |
| 2. `suppressed_reasons` Optional-only | **Resolved.** Now Optional+Computed with `setplanmodifier.UseStateForUnknown()`. The scope interaction is documented on the argument ("Only meaningful when `suppression_scope` is `TENANT`"). |
| 3. Missing `_disappears` test | **Resolved.** `TestAccSESV2TenantSuppressionAttributes_disappears_tenant` deletes the parent tenant via `acctest.CheckFrameworkResourceDisappears(ctx, t, tfsesv2.ResourceTenant, ...)` with `ExpectNonEmptyPlan`. |
| 4. Missing CHANGELOG entry | **Still outstanding.** No `.changelog/` file on the branch. |
| 5. Create/Update read-back bypasses finder | **Resolved.** Both now use `findTenantSuppressionAttributesByName` and flatten its result directly. |
| Nit: plan check on `_basic` | **Resolved.** `plancheck.ExpectResourceAction(resourceName, plancheck.ResourceActionCreate)` added. |

No new issues found in the follow-up commit. `testAccCheckTenantSuppressionAttributesDestroy` (added in this commit) is wired up as the `CheckDestroy` for the generated identity tests; the hand-written tests keep `acctest.CheckDestroyNoop`, which is consistent with the NoOp delete.

## Cross-branch note

Both branches (`f-aws_sesv2_tenant-reputation-policy` and this one) touch `internal/service/sesv2/exports_test.go` in the same var block — expect a trivial merge conflict whichever lands second.

Design contrast with the reputation-policy branch: suppression attributes suit a separate resource (dedicated Put API, mirrors the account-level resource, independently manageable), whereas the reputation policy suits an inline attribute (no create/delete API, 1:1 with the tenant, keyed by the tenant's ARN). The two branches made different choices and both are defensible for their respective APIs.
