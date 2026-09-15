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

# Enabling webhooks in openstack-operator

In general most service operators have webhooks disabled when deployed via openstack-operator. If
and when an API conversion is needed it is necessary to enable the service operator webhooks in the
openstack operator's code base. This will involve edits to controllers/operator/openstack\_controller.go
and hack/sync-bindata.sh. After these edits running a 'make bindata' should show the newly activated
webhooks getting staged in the bindata directory for deployment.


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
