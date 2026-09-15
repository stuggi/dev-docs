# CRD version updates

CRDs in our operators are versioned. As a convention almost all of our APIs have an initial version
of 'v1beta1'. This document is meant to describe the general semantics involved in providing
an upgrade path from a 'v1beta1' to a new API version for example called 'v1beta2'.
There are some general upstream kubernetes/kubebuilder [conversion concepts](https://book.kubebuilder.io/multiversion-tutorial/conversion-concepts.html) that can be referenced for best practices. This document is
more about how the use of those recommended best practices can be applied to operators within
openstack-k8s-operators.

# CRD storage and size concerns

Each CRD definition contains all versions in a single file. So for example the CRD definition for
keystoneapi would contain version information for both the v1beta1 and v1beta2 API definitions.
When it comes to size concerns within services operators these multiple versions don't matter much as
we aren't anywhere near the storage limits in this case. For services that roll up into OpenStackControlplane
we have enough space for about 4 versions at this point. This effectively means we could maintain
an upgrade path from 3 previous versions to a current new version in openstack-operator for that CRD.

# Hub and Spoke details

In reading the upstream docs there is some ambiguity about which API version is the hub and which is the
spoke. As a matter of preference it would be preferred to make the *latest* CRD version the Hub
and any older versions still maintained in the operator codebase the spokes. This would involve
implementing ConvertFrom/ConvertTo interfaces for the older APIs directories.

# Webhooks

For the purpose of conversion a new type of webhook will be used called a conversion webhook. You
can use kubebuilder/operator-sdk to help scaffold these but for the most part it involves implementing
the Hub and Spoke interfaces mentioned in the section above. Additionally the CRD definition for
each operator will need to have a new 'webhook' section and certificate manager CA injection
enabled for that API. You can enable these via the config/crd/kustomization.yaml.

NOTE: openstack-operator itself will need special handling to ensure the CRDs for service operators
 are installed correctly with properly named certificates etc. Specifically, there needs to be
 code that ensures the namespace of these new resources corresponds to the namespace the openstack
 operators get installed to ('openstack-operators' by default).

In general it is best to move other types of webhook code for validations or defaulting to the
latest API version. The main.go file for the operator would then setup webhooks only for the latest
API version (you only need to call SetupWebhookWithManager once for the latest API).

Be sure to update any kubebuilder annotations to use the new API version (Example: v1beta2).

# APIs and Controllers

When you generate the new API you only need to create the API itself, not a new controller. Once you
have the conversion webhooks implemented you can set the new version as the 'storage' version by
adding the 'storageversion' annotation to the API struct root object. Example:

```code
//+kubebuilder:storageversion
```

Migrate the old controller and operator code so that it uses the new API version only and thus the
operator code itself only needs to work with the latest version (aside from the conversion code in the api directory itself).

# API references in other Operators

If the CRD that is getting a new version is referenced in any way in other service operators you will
need to update those versions to look for the new API version instead. This is because once the
conversion happens the API version marked as the storage version is what will be stored. Example: if
service operators query KeystoneAPI to ensure the status is complete it would need to be the latest
v1beta2 API version.

## Versioning shared test helpers

The same applies to the reusable functional-test helpers an operator ships (e.g. `api/test/helpers`), which other operators import to create and read a Kind's CRs. Keep a single helper function per Kind pointing at the current (latest/storage) version rather than keeping old-version variants around — consumers must adopt the new version anyway (see above), so the helper should not let them stick to the old one. When the Kind is bumped, switch its helper functions in place to the new version. The helper bump is then just another one of the API-reference updates described above and is coordinated with them across the consuming repos (it is a breaking change to those consumers, so it lands together with their move to the new version).

The one exception is a no-op preparation bump like this reference implementation, where there are no new fields to exercise and the conversion webhook already serves every consumer the old version transparently. There, leave the helpers on the old version so the existing suites keep exercising the conversion path, and switch them when a real field lands.

# Service Accounts concerns

Most service operators in openstack-k8s-operators have code in them to create restricted service accounts (SAs) used to deploy the openstack services themselves. These SAs are created through lib-common (`modules/common/serviceaccount`, via `common_rbac.ReconcileRbac`) which sets a **controller owner reference** on the SA pointing back at the CRD instance, using `controllerutil.SetControllerReference`. That owner reference carries an `apiVersion`, which is whatever version the reconciler currently operates on (e.g. `.../v1beta1`). When the CRD is bumped to a new version this raises the question of what happens to owner references that were written under the old version. The rest of this section answers that.

## Does a stale owner-ref apiVersion break garbage collection?

Short answer: **no, as long as the old version is still served — it is a hygiene concern, not a correctness bug.**

Kubernetes garbage collection keys on the owner's **UID**, not on the owner-ref `apiVersion`. A CRD instance keeps the same UID across a version bump (a storage-version flip does not create a new object). So when the owner CR is deleted, the GC deletes the owned SA based on the UID edge, regardless of the `apiVersion` string recorded on the owner reference. This has been verified on-cluster: with an SA carrying a stale `.../v1beta1` controller owner ref while the CR's storage version was `v1beta2` and **both versions were still served**, deleting the CR garbage-collected the SA within ~1s.

The `apiVersion` only matters when the GC has to independently resolve the owner GVK (via the RESTMapper) to confirm the owner still exists. As long as the referenced version is still **served** by the CRD, that resolution succeeds and GC behaves normally.

### The one scenario where it *does* break GC

It becomes a real GC bug **only** if a version is later *removed* from the CRD's served versions while SAs still reference it. Verified on-cluster: after dropping `v1beta1` from the CRD entirely (leaving only `v1beta2` served/stored) with an SA still carrying a `.../v1beta1` owner ref, deleting the CR **leaked the SA** — it survived indefinitely because the GC could not resolve the now-unserved `v1beta1` owner GVK. Patching that owner ref to the served `v1beta2` made the GC collect it.

**Guardrail:** never remove a served CRD version until all SAs (and other owned objects) that referenced it have had their owner refs rewritten to a still-served version. In practice the rewrite happens automatically (see below), so this is a sequencing rule for the eventual old-version cleanup, not something to enforce during the initial bump.

## The reconcile path rewrites the owner ref automatically

A common assumption is that `controllerutil.SetControllerReference` is a no-op once a controller owner ref already exists, so a plain re-reconcile would leave the old `apiVersion` in place. **That is not how controller-runtime behaves** (verified against controller-runtime `v0.21.0`; check the source if you are on a very different version):

- `SetControllerReference` finds the existing ref with `referSameObject`, which matches on group + kind + name and *ignores the version*.
- It then overwrites that ref in place with a freshly built one carrying the **current** `apiVersion`.
- lib-common's `serviceaccount.CreateOrPatch` diffs the object before/after, sees the `apiVersion` string differ, and issues a `Patch`.

Net effect: once your controller reconciles the CR as the new version (i.e. after the controller code has been migrated to the new API's Go types), the existing `common_rbac.ReconcileRbac` → `serviceaccount.CreateOrPatch` → `SetControllerReference` path **automatically rewrites** each existing SA's owner-ref `apiVersion` from the old version to the new storage version on the next reconcile. No dedicated patching code is required for the owner ref to converge.

## Patch in place, do not recreate

If you do add explicit handling, **patch the owner ref in place** — do **not** delete and recreate the SA. Recreation changes the SA's UID and rotates its bound token secret, and risks a window where long-lived deployments (which mount the SA by name, typically with `AutomountServiceAccountToken: false`) have no SA. Patching `ownerReferences[].apiVersion` is a single, idempotent, churn-free update — and it is exactly what the normal reconcile already does.

## Edge cases

- **Already converged / repeated reconcile** — idempotent. `SetControllerReference` overwrites the matched ref with an identical ref; `CreateOrPatch` sees no diff and does nothing.
- **Mid-reconcile failure** — not a problem for a single ref: `CreateOrPatch` emits one atomic `Patch` of the SA; it either applies or it doesn't, and the next reconcile retries. There is no half-written owner-ref state.
- **SA missing** — `CreateOrPatch` creates it fresh with the current (new-version) owner ref, so it is correct on creation. No recreate branch is needed.
- **Multiple owner refs** — `SetControllerReference` only touches the single controller ref (matched by group + kind + name); owner refs from other owners are left intact.

## What this means for a version bump

- Migrating your controller to the new API's Go types is sufficient for existing SA owner refs to converge to the new storage version on the next reconcile — verify this with a test rather than writing bespoke patch logic. A minimal envtest is: create a CR + SA, force the SA owner ref back to the old version, reconcile, and assert the owner ref becomes the new version (and that a second reconcile is a no-op).
- A separate "runtime validation" that re-checks owner refs every reconcile is redundant — the normal `SetControllerReference` call on every reconcile already is that safety net, and it also covers SAs restored from backup or created by an older operator.
- The only extra thing to remember is the guardrail above: sequence any future removal of a served CRD version *after* owner refs have converged.

# Bumping the OpenStackControlplane CRD

If the CRD version is exposed via the OpenStackControlplane CRD then a new version of OpenStackControlplane will also need to be created. See above section on size concerns. In general the CRD of the OpenStackControlplane works the same as service operators except there is generally no service account concerns.

## Reuse the service operator's conversion code for the bubbled-up spec

OpenStackControlPlane embeds each service's spec as a sub-field (`spec.keystone.template`, `spec.glance.template`, ...). What it embeds is the service's `*SpecCore` type (e.g. `KeystoneAPISpecCore`) — the same "core" spec, minus operator-only fields like `ContainerImage`, that the service operator also shares for defaulting and validation. When OpenStackControlPlane is bumped to a new version, converting it means converting every embedded service spec. If each of those is re-implemented field-by-field inside openstack-operator, the OpenStackControlPlane conversion becomes enormous, duplicates logic that already lives in the service operators, and drifts from them on every service field change.

Handle this the same way defaulting and validation are already shared: the conversion implementation stays in the service operator, and openstack-operator only calls it. Concretely, factor the service operator's Hub/Spoke conversion so that the spec mapping is an **exported** function on the bubble-up unit (the `*SpecCore` type), and the object-level `ConvertTo`/`ConvertFrom` is a thin wrapper that adds ObjectMeta/Status around it. The keystone reference already isolates this mapping in `specCore*` helpers on `KeystoneAPISpecCore`; exporting them is what makes them reusable. OpenStackControlPlane's own conversion then calls each service's exported per-`*SpecCore` converter for the corresponding embedded template — exactly as the OpenStackControlPlane webhook already calls each service's `Default()`/validate functions today. The field-by-field logic then exists once, in the service operator, as the single source of truth for both the service CRD's own conversion and the control plane's.

## A consumer must write the storage version, or it hot-loops

A subtle but important consequence for any operator that *creates* a service CR whose CRD was version-bumped: write it as the **storage (hub) version**. openstack-operator reconciles each service CR with `controllerutil.CreateOrPatch`, which does GET → mutate → diff (before vs after, via `runtime.DefaultUnstructuredConverter.ToUnstructured` + `reflect.DeepEqual`, excluding status) → patch-if-changed. If the consumer still uses the *old* (spoke) Go types after the CRD's storage version moved on, every write goes through the conversion webhook: what it PUTs as v1beta1 is stored as v1beta2 and read back through `ConvertFrom`, so the object it GETs next reconcile is a round-tripped copy. Unless that spoke→hub→spoke round-trip is perfectly idempotent for every field, `before != after` on each reconcile, `CreateOrPatch` returns `Updated`, and the CR is patched forever — a hot loop that also churns the service operator (which reconciles each patch) and bumps `metadata.generation` every second.

This showed up concretely after the KeystoneAPI bump: openstack-operator kept writing `keystonev1beta1.KeystoneAPI` against v1beta2 storage, and both operators reconciled ~once/second with a stable spec but ever-increasing generation. The managedFields told the story — manager `manager` owned `spec` at `v1beta1` and `status` at `v1beta2` on the same object.

The fix is to have the consumer operate on the storage version. Two options:

- **Bump the consumer's own CRD too** (the full pattern — see [Reuse the service operator's conversion code](#reuse-the-service-operators-conversion-code-for-the-bubbled-up-spec) above). OpenStackControlPlane gains a v1beta2, `spec.keystone.template` becomes `keystonev1beta2.KeystoneAPISpecCore`, and `ReconcileKeystoneAPI` builds a `keystonev1beta2.KeystoneAPI` directly. The OSCP conversion delegates the template field to the exported `keystonev1beta1.SpecCoreToV1beta2` / `SpecCoreFromV1beta2`.
- **Convert at the write boundary only** (minimal fix, no OSCP bump): keep `spec.keystone.template` as v1beta1 but build a `keystonev1beta2.KeystoneAPI` in `ReconcileKeystoneAPI`, converting the template with `keystonev1beta1.SpecCoreToV1beta2(*instance.Spec.Keystone.Template)` before the `CreateOrPatch`. This alone stops the loop, because the diff is now computed against the storage version with no webhook round-trip.

Either way the rule is the same: **the type you hand to `CreateOrPatch` must be the CRD's storage version.** Writing a served-but-non-storage version through a conversion webhook on a reconcile loop is the trap.

# Enabling webhooks in openstack-operator

In general most service operators have webhooks disabled when deployed via openstack-operator (`ENABLE_WEBHOOKS=false`), and only infra-operator and openstack-baremetal-operator run with them enabled. When enabling webhooks for a conversion it is important to treat two independent concerns separately, because a conversion webhook is not like a validating/defaulting webhook:

- **Webhook server + serving cert + conversion endpoint (mandatory, everywhere).** A conversion webhook is served by the CRD's `spec.conversion.webhook` and is enforced by the kube-apiserver at the storage layer: the moment a CRD has more than one served version, every read/write/list that needs a non-storage version calls the `/convert` endpoint. Unlike validating/defaulting (an optional admission step that simply does not run when disabled), if no backend serves `/convert` with a trusted CA, *all* operations on that CRD fail. So every operator with a version-bumped CRD must run a webhook server with a serving cert and CA injection, regardless of whether its validating/defaulting webhooks are enabled. This is the concern `ENABLE_WEBHOOKS` currently gates on the deployment side (cert mount, `--webhook-cert-path`) and it becomes mandatory for any converted CRD.
- **Validating/defaulting webhook configs (optional, per-operator).** Whether to *also* install and serve the MutatingWebhookConfiguration/ValidatingWebhookConfiguration is a separate decision that depends on the operator. infra-operator and openstack-baremetal-operator enable them because end users create those CRs directly. For the other service operators the CRs are created by the OpenStackControlPlane reconciler, whose own webhook already validates upstream, so their validating/defaulting webhooks stay off even after conversion is enabled. Enabling conversion does not require turning these on.

The practical consequence is that `ENABLE_WEBHOOKS` today does double duty — it gates both the webhook server/cert *and* validating/defaulting registration. These two must be decided separately: the server/cert half is mandatory for any converted CRD (see "Conversion activation is data-driven off the CRD" below), while the validating/defaulting half stays per-operator. Tying both to "the bundle declares any webhook" would wrongly install validating/defaulting configs for a converted operator like keystone, even though its CRs come via the OpenStackControlPlane. Longer term, once every version-bumped operator serves conversion, the server/cert half of `ENABLE_WEBHOOKS` is on everywhere and the flag can be reduced to (or replaced by) a narrower per-operator switch for the validating/defaulting configs.

## Conversion activation is data-driven off the CRD

Conversion activation must be **derived from the CRD, not gated by a separate deployment switch.** openstack-operator turns on the webhook server, serving cert and CA injection **when the deployed CRD carries a conversion section** (`spec.conversion.strategy: Webhook`, i.e. the CRD has more than one served version), for these reasons:

- **Conversion cannot be toggled off at runtime.** It is enforced by the kube-apiserver: the moment a CRD has a non-storage served version, every read/write of that version *must* go through `/convert`. Turning the endpoint off while the CRD still advertises two versions breaks all operations on that CRD. There is no safe "two versions wired but conversion disabled" state.
- **The only dormant state is single-version, and it is automatic.** With one version there is nothing to convert, controller-runtime registers no `/convert` handler, and the CRD strategy stays `None`. Nothing is needed to keep conversion "off" before it is needed — adding the second version *is* the flip.
- **The CRD is the single source of truth.** Whether an operator serves conversion is fully determined by "does its CRD carry a conversion section". Any separate switch alongside it could only duplicate that fact and drift from it (switch off + CRD says `Webhook` → `/convert` unserved → all requests fail; switch on + single version → a server/cert running for nothing).

This means the conversion **infrastructure** (an always-on webhook server + serving cert in every operator, plus openstack-operator's generic CA injection and cert wiring) can and should be built once and uniformly ahead of need, so that a real version bump on any operator is just "add the version + conversion code", with no new integration and no per-operator divergence — while the conversion endpoint itself stays naturally dormant until a second version exists.

Enabling webhooks involves edits to controllers/operator/openstack\_controller.go and hack/sync-bindata.sh. After these edits running a 'make bindata' should show the newly activated webhooks getting staged in the bindata directory for deployment.

## Defaulting is not applied during conversion

Conversion and admission are separate stages of the kube-apiserver pipeline, and a conversion never triggers the defaulting/mutating admission webhook. The conversion webhook runs on read/list/watch/write and only reshapes the object (your `ConvertTo`/`ConvertFrom`); the defaulting webhook (the Go `Default()` method) runs *only* in the admission chain — create/update/patch — against the version the client submitted. Converting or reading an existing object as the new version does not run `Default()`.

The consequence for a version bump that introduces new defaults:

- **Go webhook defaults (`Default()`)** are NOT applied when an existing stored object is converted/read as the new version. They are applied only the next time that CR is written (create/update/patch), or via an explicit storage migration. So a no-op read of an old object as the new hub version will not carry them.
- **CRD schema defaults (`+kubebuilder:default=`)** are applied by the apiserver itself — on write, and on read for the served version — independent of any webhook. Because we make the latest version the hub/storage version, these get applied when serving the new version.

For a lossless hub object, set any new-version default either as a CRD schema default or explicitly in `ConvertTo`/`ConvertFrom`. Do not rely on the mutating webhook to backfill defaults through conversion — it will not run.

This is also a reason a service operator may want its **own** defaulting webhook enabled, rather than relying solely on the OpenStackControlPlane layer as today. Currently most service operators run with `ENABLE_WEBHOOKS=false` and their CRs are defaulted upstream when the OpenStackControlPlane reconciler templates and creates them; the service operator never defaults the CR itself. That covers the OSCP-created path, but it does not cover defaults that must apply at the service-CR boundary — new-version Go defaults on an already-existing CR (which conversion won't run) or a CR edited outside the exact OSCP path. If a version bump adds defaulting logic that must land on the service CR directly, enable the service operator's defaulting webhook (the validating/defaulting half of the two concerns above) instead of expressing it only at the control-plane level. Expressing the default as a CRD schema default avoids the need for a running webhook entirely and is preferable when the default is a static value.


# Timing of CRD conversion with regards to OpenStack minor updates

CRD conversions will occur when the API reconciles which can be periodic or immediately when a CRD with
a new version gets modified in anyway. This could cause a concern for a service disruption or outage
within OpenStack in general. Service operators may consider an immediate update of an API deployment
to be acceptable. If there is a need (for any reason) to keep pods, deployments, statefulsets
untouched until a 'minor update window' then service operators must provide custom code to accommodate
this. A recommended approach might be to support both the old and new deployment structures
internally and use the [service defaults](version_updates.md#managing-openstack-service-version-incompatibilities) mechanism to support migrating
to the new deployment default during a minor update window when a new OpenStackVersion targetVersion
is applied.
