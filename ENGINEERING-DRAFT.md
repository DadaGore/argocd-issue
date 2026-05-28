# Engineering Input Needed: Platform GitOps Behavior

Hi team,

During a customer repro of ArgoCD + vCluster Platform GitOps, we hit two distinct
issues we'd like your input on before recommending a fix path.

---

## Issue 1: Platform injects subresources into Project spec on every upgrade

**What happens:**
When a `Project` resource is managed via GitOps, the Platform controller automatically
adds extra entries to `spec.access[*].subresources` after creation. These entries change
between Platform versions -- upgrading from 4.3.4 to 4.9.0 added a new `nodetypes`
subresource. Every upgrade risks breaking the GitOps sync until the git manifest is
manually updated to match what the Platform injects.

**Customer impact:**
After a Platform upgrade, ArgoCD shows `OutOfSync` on all GitOps-managed Projects until
someone manually discovers and adds the new subresource to git. There is no way to
predict which subresources will be added by a new version without testing.

**Suggested approaches -- which do you prefer?**

- A) Document the injected subresources per version so customers can pre-update their manifests before upgrading.
- B) Stop injecting subresources into the spec (make them read-only computed fields in status instead).
- C) Add a `skipDefaultSubresources` flag so customers can opt out of injection.

---

## Issue 2: Platform reverts owner and access user changes made via GitOps

**What happens:**
When `spec.owner` or `spec.access[*].users` is changed in a git manifest and synced via
ArgoCD, the Platform controller immediately overwrites the change back to the original
value. The ArgoCD operation reports `Succeeded` but the live resource is never updated.
The app then stays permanently `OutOfSync`.

**Customer impact:**
Customers cannot transfer project ownership or rotate access users via GitOps. The only
workaround is to either use the Platform UI (breaking GitOps) or remove these fields from
git entirely and accept that ownership is untracked.

**Suggested approaches -- which do you prefer?**

- A) Allow `spec.owner` and `spec.access` writes via the management API (make the controller respect external updates).
- B) Add a dedicated ownership-transfer API/endpoint separate from the resource spec so customers have a supported path.
- C) Document that ownership is Platform-authoritative and provide official `ignoreDifferences` config for ArgoCD users.

---

Please advise on preferred direction for each. Happy to jump on a call to walk through
the repro if helpful. Full repro steps and evidence at:
https://github.com/DadaGore/argocd-issue
