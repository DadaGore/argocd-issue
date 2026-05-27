# Findings: vCluster Platform GitOps via ArgoCD

**Date:** 2026-05-27
**Platform versions tested:** 4.3.4, 4.9.0
**ArgoCD version:** v3.4.2
**Environment:** kind cluster (`vcluster-repro-control-plane`)

---

## 1. ArgoCD Bootstrapping Issues

### 1.1 Missing ApplicationSet CRD

On a fresh ArgoCD v3.4.2 install, the `applicationsets.argoproj.io` CRD may be absent if
the install manifest was partially applied. The `argocd-applicationset-controller` enters
`CrashLoopBackOff` with:

```
failed to get restmapping: no matches for kind "ApplicationSet" in version "argoproj.io/v1alpha1"
failed to wait for applicationset caches to sync ... timed out waiting for cache to be synced
```

**Fix:** Re-apply the full ArgoCD install manifest at the installed version.

---

## 2. ArgoCD Configuration for Platform Resources

`management.loft.sh/v1` is an **aggregated API** (served by the Platform's `loft-apiservice`),
not a CRD. This requires specific ArgoCD configuration that differs from standard CRD-backed
resources.

### 2.1 Do NOT use ServerSideApply

`ServerSideApply=true` causes ArgoCD to fail with:

```
unable to resolve parseableType for GroupVersionKind: management.loft.sh/v1, Kind=Project
```

ArgoCD cannot resolve the OpenAPI schema for aggregated API types. Remove `ServerSideApply=true`
from the Application's `syncOptions`.

### 2.2 Project resource is cluster-scoped

`Project` (management.loft.sh/v1) is **not namespaced**. Set `destination.namespace: ""`
in the ArgoCD Application. Using a namespace (e.g. `vcluster-platform`) causes silent
tracking-ID mismatches and drift.

### 2.3 Required ArgoCD Application config

```yaml
spec:
  destination:
    server: https://kubernetes.default.svc
    namespace: ""
  ignoreDifferences:
    - group: management.loft.sh
      kind: Project
      jsonPointers:
        - /metadata/resourceVersion
        - /metadata/generation
        - /metadata/managedFields
        - /status
        - /spec/owner
  syncPolicy:
    syncOptions:
      - RespectIgnoreDifferences=true
      - Replace=true
```

`Replace=true` is required because client-side apply (patch) sends `resourceVersion: 0`
through the aggregated API layer, which the `storage.loft.sh` validator rejects:

```
projects.storage.loft.sh "tenants-dev" is invalid:
  metadata.resourceVersion: Invalid value: 0: must be specified for an update
```

---

## 3. Platform-Injected Subresources Cause Sync Drift

The Platform controller mutates `spec.access[*].subresources` after resource creation,
adding entries not present in the user-supplied manifest. These vary by Platform version
and must be included in git to keep the ArgoCD app in sync.

### Subresources injected per version

| Platform version | Subresources added to `loft-access` |
|-----------------|--------------------------------------|
| 4.3.4 | `members`, `clusters`, `templates`, `chartinfo`, `charts`, `runners` |
| 4.9.0 | `members`, `clusters`, `templates`, `nodetypes`, `chartinfo`, `charts`, `runners` |

4.9.0 added `nodetypes`. Any Platform upgrade may introduce new subresources, requiring
a corresponding git update to restore `Synced` status.

### Symptom

After upgrade from 4.3.4 to 4.9.0:

```
NAME                SYNC STATUS   HEALTH STATUS
vcluster-projects   OutOfSync     Healthy
```

Live resource had `nodetypes` in subresources; git manifest did not.

---

## 4. Owner/User Field Revert Bug (Core Finding)

### Setup

- Project `tenants-dev` created with owner `waladtestexamplecom`
- Git manifest updated to swap owner to `avery-buffington-aeratechnology-com`
- ArgoCD forced sync (Replace=true)

### Observed behavior

ArgoCD reports the sync as **succeeded**:

```json
{
  "phase": "Succeeded",
  "message": "successfully synced (all tasks run)",
  "syncResult": {
    "resources": [
      {
        "kind": "Project",
        "name": "tenants-dev",
        "message": "project.management.loft.sh/tenants-dev replaced",
        "status": "Synced"
      }
    ],
    "revision": "13bdf93cd5b06bb26586ece319213a6ee04f5cb4"
  }
}
```

But the live resource immediately reverts to the original owner:

```yaml
# After ArgoCD sync (git: avery-buffington-aeratechnology-com)
spec:
  access:
    - name: loft-admin-access
      users:
        - waladtestexamplecom   # <-- Platform reverted this
  owner:
    user: waladtestexamplecom   # <-- Platform reverted this
```

**resourceVersion at time of observation:** `11350`
**generation at time of observation:** `11` (high generation count confirms repeated reconcile cycles)

### Root cause

The Platform controller (`loft`) reconciles `spec.owner` and `spec.access[*].users`
from its internal `storage.loft.sh` state after every external write. Writes to the
`management.loft.sh` aggregated API for these fields are accepted at the API layer
but immediately overwritten by the controller.

ArgoCD enters a permanent `OutOfSync` loop:

```
NAME                SYNC STATUS   HEALTH STATUS
vcluster-projects   OutOfSync     Healthy
```

The app remains healthy (Platform is working) but ArgoCD and the Platform fight
indefinitely over the owner field.

---

## 5. Workarounds

### Option A: Manage ownership out-of-band

Remove `spec.owner` and `spec.access[*].users` from the git manifest entirely.
Manage them through the Platform UI or CLI. ArgoCD manages the structural config
(members, allowedClusters, templates) while the Platform retains control of ownership.

**Tradeoff:** Ownership is no longer in git. Not fully GitOps.

### Option B: Transfer ownership through Platform UI

Use the Platform UI to transfer project ownership from `waladtestexamplecom` to
`avery-buffington-aeratechnology-com`. The Platform's internal state updates, and
the git manifest (still showing the old owner) will then show as `OutOfSync` --
update git to match after the transfer.

**Tradeoff:** Two-step process; easy to forget to update git.

### Option C: Add spec.owner to ignoreDifferences

```yaml
ignoreDifferences:
  - group: management.loft.sh
    kind: Project
    jsonPointers:
      - /spec/owner
      - /spec/access/0/users
```

Silences the drift detection. Platform remains authoritative for ownership.

**Tradeoff:** Ownership changes in git are silently ignored.

---

## 6. Repo Structure

```
argocd-issue/
├── bootstrap/
│   ├── platform-app.yaml    # ArgoCD Application for Platform Helm chart
│   └── projects-app.yaml    # ArgoCD Application for Platform Projects
├── platform/
│   └── values.yaml          # Platform Helm values
└── projects/
    └── tenants-dev.yaml     # Project manifest (management.loft.sh/v1)
```

Bootstrap apps are applied manually (`kubectl apply -f bootstrap/`).
The `projects/` directory is managed by ArgoCD.
