# HorizontalPodAutoscaler Support for Package Runtimes

* Owner: Jonasz Łasut-Balcerzak (@jonasz-lasut)
* Reviewers: Crossplane Maintainers
* Status: Draft

## Background

Crossplane's package manager creates a Kubernetes `Deployment` for each
package revision that has a runtime — today that means Providers and
Functions. Users configure these Deployments through a
`DeploymentRuntimeConfig` (DRC), which exposes a templated
`DeploymentSpec` along with templates for the companion `Service` and
`ServiceAccount`.

Two problems keep users from running these Deployments behind a
`HorizontalPodAutoscaler` (HPA).

**The package runtime builder unconditionally pins `.spec.replicas` to
`1`** on every reconcile when the DRC omits the field. The override is
applied at `internal/controller/pkg/runtime/runtime.go:214` via
`DeploymentWithOptionalReplicas(1)`. Any external actor — HPA, KEDA,
`kubectl scale` — that changes the replica count has its value reset on
the next reconcile. The user always loses this silent fight.

**The Deployment's name is dynamically generated from the revision
name.** The revision name is `xpkg.FriendlyID(package-name,
package-digest-hex)`, so every package upgrade produces a new digest, a
new revision name, and a new Deployment name. Because
`HorizontalPodAutoscaler.spec.scaleTargetRef` is name-based (HPA has no
label-selector mode), a user-authored HPA cannot be pointed at
"whatever Deployment the current revision produced" — every upgrade
would orphan it. Pinning the Deployment name via the DRC's
`spec.deploymentTemplate.metadata.name` is a fragile workaround and
introduces other risks at revision-transition time.

Together these mean that an HPA for a package runtime cannot be a
BYO ("bring your own") feature. Crossplane is the only actor that
knows the current revision's Deployment name at reconcile time, so
Crossplane must be the one to author the HPA and fill in its
`scaleTargetRef`.

## Goals

The goals of this design are to:

* Let users configure a `HorizontalPodAutoscaler` for Function and
  Provider package runtimes through the existing
  `DeploymentRuntimeConfig` shape.
* Stop the silent reconcile fight over `.spec.replicas` for users of
  KEDA, `kubectl scale`, and other non-HPA scalers — even when they
  don't opt into the new HPA feature.
* Keep Providers that cannot safely run with more than one replica
  protected by default. A package author must explicitly assert that
  the Provider is safe to replicate before Crossplane will create an
  HPA for it.

## Proposal

### A Crossplane-managed HPA template on DeploymentRuntimeConfig

Add a new optional field to `DeploymentRuntimeConfigSpec` that holds a
template for the HPA Crossplane should create alongside the package
runtime Deployment:

```go
type DeploymentRuntimeConfigSpec struct {
    // ... DeploymentTemplate, ServiceTemplate, ServiceAccountTemplate unchanged

    // HorizontalPodAutoscalerTemplate is the template for a
    // HorizontalPodAutoscaler that Crossplane will create alongside
    // the package runtime Deployment. When set, Crossplane owns and
    // reconciles the HPA; its scaleTargetRef is filled in by Crossplane
    // to point at the package revision's Deployment.
    //
    // This field is alpha and gated on the
    // --enable-package-horizontal-pod-autoscaling Crossplane flag.
    //
    // For Providers, this field is only honored if the Provider package
    // declares the "safe-to-scale-horizontally" capability in its crossplane.yaml.
    // +optional
    HorizontalPodAutoscalerTemplate *HorizontalPodAutoscalerTemplate `json:"horizontalPodAutoscalerTemplate,omitempty"`
}

type HorizontalPodAutoscalerTemplate struct {
    // +optional
    Metadata *ObjectMeta `json:"metadata,omitempty"`

    // Spec contains the configurable spec fields for the HPA. Note
    // that scaleTargetRef is overwritten by Crossplane and must not
    // be set.
    // +optional
    Spec *autoscalingv2.HorizontalPodAutoscalerSpec `json:"spec,omitempty"`
}
```

The full `autoscaling/v2.HorizontalPodAutoscalerSpec` is passed through
verbatim (minus `scaleTargetRef`, which Crossplane owns). We do not
curate which metrics, external metrics, or scaling `behavior` fields
to surface — any future addition to the upstream type becomes
available automatically.

A CEL validation rule rejects manifests that set `scaleTargetRef`:

```go
// +kubebuilder:validation:XValidation:rule="!has(self.spec) || !has(self.spec.scaleTargetRef)",message="scaleTargetRef must not be set; Crossplane overwrites it to point at the package revision's Deployment"
```

This turns "setting scaleTargetRef is a no-op" into an explicit admission
error rather than a silent behavior.

### Crossplane builds and reconciles the HPA

When the feature flag is enabled and a DRC has
`horizontalPodAutoscalerTemplate` set, the package revision reconciler
— through the existing `ManifestBuilder` / hook plumbing — builds an
HPA whose:

* `metadata.name` defaults to the revision name (same default as the
  Deployment).
* `metadata.namespace` is the Crossplane system namespace.
* `metadata.ownerReferences` point at the `PackageRevision`, so the
  HPA is garbage-collected with it.
* `spec` is copied from the template, with `scaleTargetRef` filled in
  to point at the current revision's Deployment by name.

The HPA is applied in the hook's `Post` step (alongside the
Deployment) and deleted in `Deactivate`. Users do not manage the HPA's
lifecycle directly; it follows the revision.

### Provider safety rail via the package capability mechanism

Not all Providers can safely run with more than one replica. Providers
that lack leader election or horizontal sharding will race each other,
double-reconcile resources, and potentially corrupt state.

Crossplane already has a package-metadata capability mechanism for
declaring package-author-facing assertions, declared in a package's
`crossplane.yaml` and surfaced on the revision's
`status.capabilities`. Providers use it today to advertise
`safe-start`, opting in to running before their dependent CRDs are
fully established. We reuse that mechanism: a new
`safe-to-scale-horizontally` capability is declared in the Provider's
`crossplane.yaml`:

```yaml
apiVersion: meta.pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-example
spec:
  capabilities:
  - safe-to-scale-horizontally
```

At revision reconcile time, if a DRC attached to a Provider sets
`horizontalPodAutoscalerTemplate`, the reconciler checks whether the
Provider has declared `safe-to-scale-horizontally`. If it has not, the hook
returns an error (surfaced on the `RevisionHealthy` condition) and the
HPA is not applied. The error message tells the user exactly which
capability is missing and where to declare it.

Functions have no such gate: they are stateless gRPC servers, every
invocation is independent, and running them behind a load-balancing
Service is the standard case.

### Unconditional replicas default removal

Delete line `DeploymentWithOptionalReplicas(1),` from
`internal/controller/pkg/runtime/runtime.go` and the now-unused
`DeploymentWithOptionalReplicas` helper in
`internal/controller/pkg/runtime/runtime_override_options.go`.

After this change, a DRC that omits
`spec.deploymentTemplate.spec.replicas` produces a Deployment with
`.spec.replicas: nil`. Because the field is `*int32` with `omitempty`,
this marshals as an omitted field. Kubernetes defaults it to `1` on
initial creation and never touches it on subsequent applies, leaving
the managed HPA — or any other external scaler — free to own the
field.

This change is intentionally **not** gated on the new feature flag.
Three user states:

| DRC `replicas` | Externally scaled? | Today        | After change        |
|----------------|--------------------|--------------|---------------------|
| Explicit `N`   | n/a                | `N`          | `N` (unchanged)     |
| Unset          | no                 | `1`          | `1` (unchanged)     |
| Unset          | yes                | reset to `1` | external value wins |

Only the third row changes behavior. Users in that state today are
already in a broken state and will simply stop being fought after the
upgrade. This is a bug fix, not a regression, and deserves to ship
independent of the alpha HPA feature.

### Feature flag

Gated behind a new alpha feature flag
`EnableAlphaPackageHorizontalPodAutoscaling`, exposed as the CLI flag
`--enable-package-horizontal-pod-autoscaling` on the core Crossplane binary
and as `args.enablePackageHorizontalPodAutoscaling` in the Helm chart. The
flag is off by default. When off, Crossplane ignores the new DRC
field entirely.

The Provider `safe-to-scale-horizontally` capability gate is a safety rail,
not a feature gate. It stays in place forever, including after this
feature graduates to beta and GA.

## Alternatives Considered

### BYO HPA

Let users author their own HPA and simply remove the replicas default
so Crossplane stops fighting them. The dynamic
Deployment name problem surfaced: every package upgrade would orphan
a user-authored HPA, because HPA's `scaleTargetRef` has no
label-selector mode. A user could pin the Deployment name in the DRC
to work around this, but that is fragile at revision-transition time
and does not compose well with other Crossplane features.

### Opinionated subset of HPA fields

Surface only `minReplicas`, `maxReplicas`, and a simplified CPU
target — wrap the HPA behind Crossplane's own schema. This gives us
more API stability but loses access to every advanced feature of
autoscaling/v2 (custom metrics, external metrics, behavior tuning,
new fields added upstream). Given that the target audience is
cluster operators who already understand Kubernetes autoscaling, the
cost of curating the subset outweighs the benefit.

### Support KEDA / other autoscalers via the same field

Make the DRC field generic over autoscaler types. Rejected as
premature: we do not yet have a user asking for KEDA-managed package
runtimes, and the replicas-default removal alone unblocks KEDA BYO
for users who want it today. If a future request materializes, a
new sibling field can be added without breaking this one.

## Backwards Compatibility

The HPA feature itself is off by default behind an alpha flag; users
on existing clusters observe zero change until they opt in.

The replicas default removal is a behavior change, but only for users
currently experiencing the silent reconcile fight — see the table in
the Proposal section. Users with explicit replicas in their DRC, and
users who have not tried to scale externally, see no change.

Provider authors whose controllers are safe to run multi-replica
today must add `safe-to-scale-horizontally` to their `crossplane.yaml` before
users can attach an HPA template to a DRC bound to their Provider.
This is an additive change on the Provider side; existing Providers
without the capability continue to function exactly as before. If a
user attaches an HPA template to a DRC bound to a Provider that has
not declared the capability, Crossplane does not create the HPA and
marks the `ProviderRevision` unhealthy with an error message naming
the missing capability.

## Future Work

* Graduate the feature flag to beta after adoption signal.
* Consider a symmetric safety declaration for Functions if a future
  Function type is ever made stateful.
* Consider additional scaler integrations (KEDA, CronHPA) if demand
  materializes, either via a sibling field on DRC or via an extension
  mechanism.
