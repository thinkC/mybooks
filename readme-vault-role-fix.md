# README fix: Vault role name and service-account names

Two commits on `main`, both touching only `README.md`'s "Vault prerequisites"
section (steps 3–5). No driver code, manifests, or Helm chart changed.
Written up per the discussion that led to them — see Phase 4 dev-environment
notes (`phase-4-dev-environment-setup.md` §5) for how the underlying drift
was originally found.

---

## Commit 1 — `a2e5ec5`: role name `luks-operator-role` → `luks-csi-role`

**Problem.** `README.md` documented creating a Vault Kubernetes-auth role
named `luks-operator-role`, and explicitly claimed _"The role name
(`luks-operator-role`) matches the default in
`luks-csi-driver/values.yaml`."_ That claim was false: `values.yaml` line 39
sets `vault.role: "luks-csi-role"`, and the Helm chart's `VAULT_ROLE`
container env var is templated straight from `.Values.vault.role`
(`_helpers.tpl`'s `luks-csi-driver.vaultEnv`). Following the README exactly,
then `helm install`-ing with default values, would leave the
controller/node pods authenticating against a Vault role
(`luks-csi-role`) that was never created — Vault auth fails.

The mismatch existed because the raw-manifest deployment path
(`manifests/controller.yaml`, `manifests/node.yaml`) and `vault.py`'s own
Python-level fallback both hardcode `luks-operator-role`, and the README
was written to match that older/raw-manifest value rather than the Helm
chart's actual default.

**Fix.** Changed all four occurrences of `luks-operator-role` to
`luks-csi-role` in `README.md`:

- The `vault write auth/kubernetes/role/...` example command (step 3).
- The sentence claiming the role name matches `values.yaml`'s default (now
  actually true).
- The `--set vault.role="..."` line in the Quick Start Helm install example.
- The `vault.role` row in the values-to-customize table.

**Not changed / left as a known gap.** The raw-manifest path
(`manifests/controller.yaml`/`node.yaml`) still hardcodes
`luks-operator-role`, and `vault.py`'s own default is still
`luks-operator-role`. Deploying via raw manifests with the Vault role now
named `luks-csi-role` (per this fix) would itself mismatch — the README's
Vault prerequisites section, as it now stands, is correct for the **Helm**
path, not the raw-manifest path. This is the same class of problem
addressed in commit 2 below for service-account names, but for the role
name it hasn't been resolved — worth a follow-up if the raw-manifest path
still needs first-class README support.

---

## Commit 2 — `e351fcc`: service-account names differ by deployment path

**Problem, surfaced while reviewing commit 1.** The same Vault-role example
command bound the role to `bound_service_account_names="luks-csi-controller,luks-csi-node"`
— the **raw-manifest** service account names
(`manifests/rbac.yaml`'s `ServiceAccount` objects are literally named
`luks-csi-controller`/`luks-csi-node`). But the Helm chart computes SA names
dynamically via `luks-csi-driver.fullname` in `_helpers.tpl`:

```
fullname =
  .Values.fullnameOverride, if set
  else: Release.Name,                        if Chart.Name is a substring of Release.Name
  else: "{Release.Name}-{Chart.Name}"

controllerServiceAccount = "{fullname}-controller"
nodeServiceAccount        = "{fullname}-node"
```

`Chart.Name` is `luks-csi-driver` (`Chart.yaml`), and `nameOverride`/
`fullnameOverride` are both empty by default. The README's own Quick Start
example installs with `helm install luks-csi-driver ./luks-csi-driver/` —
release name `luks-csi-driver` contains the chart name `luks-csi-driver` as
a substring, so `fullname = Release.Name` unchanged, giving service account
names `luks-csi-driver-controller` / `luks-csi-driver-node` — **not** the
`luks-csi-controller`/`luks-csi-node` the Vault role command bound. A
different release name changes the result again (chart name gets appended
if not already a substring).

Because the Vault prerequisites section is shared by both deployment
paths (raw manifests and Helm) and the two paths produce different SA
names, hardcoding either pair would just move the same bug from one path to
the other — this was flagged and confirmed against `_helpers.tpl` and
`Chart.yaml` before making a choice.

**Fix.** Rewrote the prose to state both pairs explicitly, tied to how each
name is derived, and replaced the hardcoded `bound_service_account_names`
value with a placeholder plus inline comments pointing back to both pairs:

```bash
# Use the pair matching your deployment method — see above:
#   Helm (release name "luks-csi-driver"): luks-csi-driver-controller, luks-csi-driver-node
#   Raw manifests (manifests/rbac.yaml):   luks-csi-controller, luks-csi-node
kubectl exec vault-0 -- vault write auth/kubernetes/role/luks-csi-role \
    bound_service_account_names="<controller-sa-name>,<node-sa-name>" \
    ...
```

The prose above the command also now explains the Helm release-name
substitution rule (`<release-name>-controller`/`-node`, or
`<release-name>-luks-csi-driver-controller`/`-node` if the release name
doesn't already contain `luks-csi-driver`), so a reader using a non-default
release name isn't left guessing.

---

## Net result

`README.md`'s Vault prerequisites section (steps 3–5) is now internally
consistent and matches the Helm chart's actual defaults
(`luks-csi-driver/values.yaml`, `_helpers.tpl`) for a default-named Helm
install. It no longer silently assumes raw-manifest naming while presenting
Helm as the recommended path. The remaining known gap — the role name
itself still doesn't have first-class raw-manifest-path documentation after
commit 1 — is called out above rather than silently left inconsistent.

Both commits touch `README.md` only; no code, RBAC, or chart values changed.
