# LUKS CSI Driver — Phase 2: Architecture Explanation (Expanded)

This supersedes the Phase 2 section of `architecture-and-module-review.md`.
`SKILL.md` Phase 2 was updated to require a wider topic list and a separate
sequence diagram per RPC — this document covers that expanded spec. Phase 3
(module walkthrough) is unchanged and still lives in
`architecture-and-module-review.md`.

Labeling convention (per `CLAUDE.md`):

- **[VERIFIED]** — read directly from the cited source file.
- **[DOC]** — stated in `README.md` / `SECURITY.md` / `architecture.md` /
  `sequence.md`, not independently re-derived beyond what's noted.
- **[DRIFT]** — doc and code disagree, or two parts of the repo disagree with
  each other.
- **[INFERENCE]** — my reading of intent from code structure, not a direct
  quote.

Files verified for this pass: `main.py`, `driver.py`, `controller.py`,
`k8s.py`, `device.py`, `vault.py`, `luks.py`, `node.py`
(`luks_csi_driver/`), plus `manifests/*.yaml`, `luks-csi-driver/templates/*.yaml`,
`luks-csi-driver/values.yaml`, `SECURITY.md`.

---

## 1. The problem being solved

**[DOC]**, grounded in `SECURITY.md`'s framing (not invented): by default,
whether a Kubernetes PVC's data is encrypted at rest depends entirely on
whatever the backing storage system happens to provide (cloud block storage
encryption, Ceph/Longhorn config, etc.) — Kubernetes itself gives no
guarantee. `SECURITY.md`'s stated audience is "research workloads handling
sensitive data (health records, PII, HIPAA/GDPR-regulated data)" — i.e. the
problem is: give any PVC, on any backing storage, LUKS-grade
block-level encryption, transparently, without every application team having
to solve key management and `cryptsetup` themselves.

**[DOC]**, from `README.md`'s comparison table: the predecessor to this
driver was a kopf-based Kubernetes operator with a custom `EncryptedVolume`
CRD that required every workload pod to run `privileged: true` and install
`cryptsetup` in an init container at pod startup. This CSI driver's stated
goal is to solve the same problem — transparent LUKS encryption — through
the standard CSI interface instead, so only the driver's own Node DaemonSet
is privileged, not user pods.

## 2. Why Kubernetes storage encryption is needed / what LUKS provides

**[INFERENCE]**: encryption-at-rest addresses a specific threat: someone
with access to the _raw storage medium_ (a cloud snapshot, a decommissioned
disk, a storage-admin with backend access, physical theft) but not to a live,
mounted, decrypted volume. It does not protect against an attacker who
already has code execution inside a pod with the volume mounted — the plaintext
filesystem is fully readable there regardless of LUKS.

**[VERIFIED]** what LUKS specifically provides here: `luks.py` formats with
LUKS2 by default (`luksType` StorageClass param, default `luks2`,
`luks.py::luks_format` line 46). LUKS2's default cipher is AES-XTS
(confirmed operationally in the README's Lima walkthrough expected output:
`cipher: aes-xts-plain64`, README lines 473–478). Per `SECURITY.md` finding
**C3**, this is **confidentiality only** — LUKS2 also supports
`--integrity` (dm-integrity/AEAD) for tamper detection, which this driver
does **not** enable. **[VERIFIED]**: no `--integrity` flag anywhere in
`luks.py::luks_format`.

## 3. What a CSI driver is, and this driver's three gRPC services

**[INFERENCE + VERIFIED]** A CSI (Container Storage Interface) driver is a
gRPC server that Kubernetes' storage machinery (kubelet + sidecar
controllers) calls into over a Unix domain socket, instead of Kubernetes
having in-tree knowledge of a specific storage backend. This repo implements
three of the CSI service groups, each as its own Python class:

| CSI service | Class                | File            | Registered when                 |
| ----------- | -------------------- | --------------- | ------------------------------- |
| Identity    | `IdentityServicer`   | `driver.py`     | always (`main.py` line 115)     |
| Controller  | `ControllerServicer` | `controller.py` | `CSI_MODE in (controller, all)` |
| Node        | `NodeServicer`       | `node.py`       | `CSI_MODE in (node, all)`       |

**[VERIFIED]** (`main.py` lines 108–124). A single container image serves
all three roles; `CSI_MODE` (env var, default `all`) picks which get
registered on the same gRPC server bound to `/csi/csi.sock`
(`main.py` lines 35, 142–150).

- **Identity** (`driver.py`): `GetPluginInfo` returns the driver name/version
  (`luks.csi.example.com` / `0.1.0`, lines 7–8); `GetPluginCapabilities`
  advertises only `CONTROLLER_SERVICE` (lines 18–29); `Probe` is an
  unconditional liveness stub with no real Vault/K8s connectivity check
  **[INFERENCE]**.
- **Controller** (`controller.py`): implements `CreateVolume`, `DeleteVolume`,
  `ControllerGetCapabilities`, `ValidateVolumeCapabilities`. **[VERIFIED]**
  no `ControllerPublishVolume`/`ControllerUnpublishVolume` are implemented —
  grepped, absent. This matters for §7 below.
- **Node** (`node.py`): implements `NodeStageVolume`, `NodeUnstageVolume`,
  `NodePublishVolume`, `NodeUnpublishVolume`, `NodeGetCapabilities`
  (advertises only `STAGE_UNSTAGE_VOLUME`, lines 255–264), `NodeGetInfo`
  (returns the node name, line 267).

## 4. The role of the CSI external-provisioner

**[VERIFIED]**, from `manifests/controller.yaml` lines 37–46 and the Helm
equivalent `luks-csi-driver/templates/controller.yaml` lines 41–50: the
Controller Deployment runs the driver container plus a sidecar,
`registry.k8s.io/sig-storage/csi-provisioner:v5.1.0`, started with
`--csi-address=/csi/csi.sock --leader-election
--leader-election-namespace=<ns>`. This sidecar is what actually watches for
unbound PVCs referencing a StorageClass whose `provisioner:` field matches
this driver's name (`luks.csi.example.com`, set in
`manifests/storageclass.yaml` line 5 / `values.yaml` line 10), translates
each into a `CreateVolume` gRPC call against `controller.py`, and on success
creates the real `PersistentVolume` object that gets bound to the user's
PVC. The driver's own `ControllerServicer` never talks to the User PVC
directly — only to the external-provisioner sidecar over the local socket,
and to the _backing_ PVC via `k8s.py`.

Leader election (`coordination.k8s.io` `Leases`) is why the controller
ClusterRole grants `leases: get,list,watch,create,update,patch,delete`
**[VERIFIED]** (`manifests/rbac.yaml` lines 41–44) — needed because
`replicaCount` could be scaled beyond 1 for HA, though `values.yaml` line 32
defaults it to `1`.

## 5. The role of the node-driver-registrar

**[VERIFIED]**, `manifests/node.yaml` lines 52–69 /
`luks-csi-driver/templates/node.yaml` lines 62–76: the Node DaemonSet also
runs the driver container plus
`registry.k8s.io/sig-storage/csi-node-driver-registrar:v2.12.0`, with
`--kubelet-registration-path=/var/lib/kubelet/plugins/luks.csi.example.com/csi.sock`.
This sidecar is what tells kubelet, on this specific node, "there's a CSI
plugin named `luks.csi.example.com` listening at this socket" by writing a
registration file kubelet watches (`registration-dir`, hostPath
`/var/lib/kubelet/plugins_registry`). Without it, kubelet would never call
`NodeStageVolume`/`NodePublishVolume` on this driver at all — CSI plugin
discovery is registrar-driven, not automatic. A `preStop` hook removes the
registration socket on pod shutdown so a terminating node pod doesn't leave
kubelet pointing at a dead socket. **[VERIFIED]**

## 6. The backing StorageClass vs. the user-facing encrypted StorageClass

**[VERIFIED]**, these are two distinct `StorageClass` objects:

- **User-facing**: `manifests/storageclass.yaml` — `name: luks-encrypted`,
  `provisioner: luks.csi.example.com` (this driver), `volumeBindingMode:
Immediate` (comment at line 7–8 explains why: `CreateVolume` provisions
  the backing PVC synchronously, so `WaitForFirstConsumer` — which defers
  provisioning until a pod is scheduled — isn't supported yet). Its
  `parameters` block is what `controller.py::CreateVolume` reads via
  `request.parameters` (`PARAM_BACKING_SC` etc., lines 31–37).
- **Backing**: whatever StorageClass name is given as the `backingStorageClass`
  parameter (default `local-path` in both the raw manifest and
  `values.yaml` line 55) — this is a **pre-existing** StorageClass in the
  cluster (Longhorn, Ceph RBD, Cinder, EBS, or the sample's `local-path`),
  provisioned by a _different_ driver entirely. This driver never
  provisions raw storage itself; it always delegates to another
  StorageClass and layers LUKS on top of whatever block device that
  StorageClass produces.

## 7. The backing PVC and PV

**[VERIFIED]** `controller.py::CreateVolume`:

1. Derives a backing PVC name: `_backing_pvc_name(name)` = `"luks-backing-" +
name.lower().replace("_","-")`, truncated to 63 chars, trailing `-`
   stripped (lines 45–47).
2. `k8s.create_pvc(pvc_name, namespace, backing_sc, size_str)` — creates a
   **Block-mode** PVC (`volume_mode="Block"`, `k8s.py` line 69) against the
   backing StorageClass. Block mode (not Filesystem mode) is what exposes
   the volume to the node as a raw block device rather than kubelet
   pre-mounting a filesystem — required for `cryptsetup` to operate on it
   directly.
3. `k8s.wait_for_pvc_bound(pvc_name, namespace)` polls every 3s, up to
   120s, until `pvc.status.phase == "Bound"`, then returns
   `pvc.spec.volume_name` (the backing PV's name). **[VERIFIED]** (`k8s.py`
   lines 80–91)
4. The returned `volume_id` for the _user-facing_ CSI volume is literally
   `f"{namespace}/{pvc_name}"` (line 80) — this string is the only handle
   downstream RPCs (`NodeStageVolume`, `DeleteVolume`) get; there is no
   separate CSI "volume database," Kubernetes objects themselves are the
   state store. **[VERIFIED]**

## 8. VolumeAttachment handling — corrected

This is the one place earlier documentation (`architecture.md`,
`sequence.md`, and even a comment in `node.py` itself) is **[DRIFT]**-flagged
against the actual code, and it's worth stating precisely now with full
evidence, including two pieces of evidence not in the earlier
`architecture-and-module-review.md` pass:

**What's actually true, verified from multiple independent sources:**

1. **`CSIDriver.spec.attachRequired: false`** — both
   `manifests/csidriver.yaml` line 6 and
   `luks-csi-driver/templates/csidriver.yaml` line 8. This tells Kubernetes
   itself that _this driver_ (`luks.csi.example.com`) has no attach phase —
   kubelet will never call `ControllerPublishVolume`/`ControllerUnpublishVolume`
   on it, and no `VolumeAttachment` object is ever created for _this_
   driver's own volumes. **[VERIFIED]**
2. **No `ControllerPublishVolume`/`ControllerUnpublishVolume` in
   `controller.py`** — confirmed by grep; consistent with (1). **[VERIFIED]**
3. **No `VolumeAttachment` API calls anywhere in `luks_csi_driver/*.py`** —
   confirmed by grep (`create_volume_attachment`, `delete_volume_attachment`,
   `VolumeAttachment(` all absent). **[VERIFIED]**
4. What actually happens instead: `device.py::ensure_staged()` (lines
   244–335) creates a short-lived **pause pod**
   (`registry.k8s.io/pause:3.9`) pinned to the target node
   (`node_name=...`), which mounts the _backing_ PVC as a raw block device
   (`V1VolumeDevice`). This indirectly triggers **kubelet's own normal CSI
   machinery for the backing driver** (e.g. `ControllerPublishVolume` +
   `NodeStageVolume` on the Ceph/Longhorn/Cinder driver, whatever `backingStorageClass`
   points at) — because now a real pod on a real node references that PVC.
   `release_staged()` deletes the pod during `NodeUnstageVolume`, letting
   kubelet release the device the normal way. **[VERIFIED]**
5. **RBAC evidence, and a real doc-vs-doc drift between the two deployment
   paths in this repo**: the **raw manifest** RBAC
   (`manifests/rbac.yaml` lines 61–77) still grants the node ServiceAccount
   `volumeattachments: get,list,watch,create,delete` and its comment still
   says _"VolumeAttachments are created/deleted to trigger attach/detach on
   the backing CSI driver"_ — describing the old design. It grants **no
   `pods` permissions at all**. The **Helm chart** RBAC
   (`luks-csi-driver/templates/rbac.yaml` lines 60–68), by contrast, grants
   `pods: get,list,watch,create,delete` and its comment correctly describes
   the pause-pod mechanism — this one matches the code. **[VERIFIED, both
   files]**

**Practical consequence** — stated carefully since I have not run this
against a live cluster: `device.py::ensure_staged` calls
`core.create_namespaced_pod` / `read_namespaced_pod` /
(`release_staged`) `delete_namespaced_pod`. If the **raw manifests**
(`manifests/rbac.yaml`) are what's deployed, the node ServiceAccount has no
RBAC grant for the `pods` resource, so those calls would fail with a 403
Forbidden — but only for backing storage classes where `_csi_spec(pv)` is
truthy (i.e. any CSI-backed backing driver: Longhorn, Ceph RBD, Cinder,
EBS). `local`/`hostPath` backing PVs skip `ensure_staged` entirely (`device.py`
line 263: `if not _csi_spec(pv): return`), which is exactly the path the
README's own Lima quick-start exercises (`local-path` backing SC, static
loop-device PV) — so this gap would not surface in the documented local dev
workflow, only against a real CSI-backed backing StorageClass. **[VERIFIED
code path; INFERENCE on "would fail" since I haven't reproduced it live —
recommend confirming with `kubectl auth can-i create pods
--as=system:serviceaccount:kube-system:luks-csi-node` against a
manifests-deployed cluster before relying on this]**.

## 9. Device discovery and resolver selection

**[VERIFIED]**, `device.py::RESOLVERS` (lines 148–153), tried in order via
`resolve_device_path()` (lines 221–227), first non-`None` wins:

1. `LocalResolver` — `pv.spec.local.path` or `pv.spec.host_path.path`,
   direct from the PV spec.
2. `LonghornResolver` — `csi.driver` containing `"longhorn"` →
   `/dev/longhorn/<volumeHandle>`.
3. `CephRBDResolver` — `csi.driver` containing `"rbd"` → scans
   `/sys/bus/rbd/devices/*/{pool,name}` directly rather than trusting
   `/dev/rbd/<pool>/<image>` udev symlinks, because (class docstring, lines
   99–107) those symlinks need `ceph-common` udev rules not present on a
   bare k3s node with only the kernel RBD module loaded.
4. `ByIdResolver` (fallback) — scans `/dev/disk/by-id/` for an entry whose
   name contains the CSI volume handle, hyphens stripped on both sides to
   handle drivers (Cinder) that truncate the handle in the symlink name.

Runs _after_ `ensure_staged` in `attach_and_resolve()` (line 386–390) so the
device is guaranteed present by the time resolution is attempted; then
`wait_for_device()` polls `os.path.exists()` every `POLL_INTERVAL=3`s up to
`DEVICE_TIMEOUT=120`s. **[VERIFIED]**

## 10. Vault key creation and retrieval

**[VERIFIED]**, `vault.py`. Keys live in Vault KV v2 at
`{vault_mount}/{vault_path}/{volume_name}` (`_split_path`, lines 25–27) —
default `secret/tenants/default/luks-keys/{volume_name}`, matching the
README's documented path.

- **Creation** (`ensure_secret`, lines 39–57, called from
  `controller.py::CreateVolume` line 94): tries a read first; on
  `hvac.exceptions.InvalidPath` generates `secrets.token_hex(32)` (256 bits
  from Python's CSPRNG) and writes it. Idempotent — a retried `CreateVolume`
  finds the existing key and no-ops.
- **Retrieval** (`read_secret`, lines 60–74, called from
  `node.py::NodeStageVolume` line 155 and `_open_with_rotation`): optional
  `version` param for fetching a specific historical version (used only for
  rotation, see §14).
- **Auth**: every one of `ensure_secret`/`read_secret`/`current_version`/
  `delete_secret` independently calls `get_client()` (lines 30–36), which
  reads the pod's service account JWT off disk and does a fresh
  `client.auth.kubernetes.login()` — **[VERIFIED]** there is no client/token
  caching or reuse across calls, so every Vault operation costs one extra
  Kubernetes-auth login round trip.

## 11. LUKS formatting and opening

**[VERIFIED]**, `luks.py`, invoked from `node.py::NodeStageVolume` (lines
157–164): `is_luks(device)` (bypasses the shared `_run()` error-raising
wrapper on purpose — nonzero exit means "not LUKS," not a fatal error,
per the function's own docstring) branches into:

- **First use**: `luks_format(device, key, luks_type)` →
  `cryptsetup luksFormat --batch-mode --type <luks1|luks2> --key-file -`
  (key piped via stdin, never argv) → `luks_open(device, mapper, key)` →
  `cryptsetup luksOpen --key-file -`.
- **Already formatted**: `_open_with_rotation(...)` (see §14).

Both `luks_open` and `luks_close` check `mapper_exists()`
(`/dev/mapper/<name>` presence) before acting, making repeated
`NodeStageVolume`/`NodeUnstageVolume` calls (kubelet retries) safe.
**[VERIFIED]**

## 12. Filesystem creation

**[VERIFIED]**, `luks.py::make_filesystem` (lines 86–95), only on first
format, immediately after `luks_open`: `mkfs.ext4 -F` or `mkfs.xfs -f`
against `/dev/mapper/<mapper>`, selected by the `filesystem` StorageClass
parameter (default `ext4`). Any other value raises `ValueError` — no silent
fallback.

## 13. Staging and publishing

**[VERIFIED]**, `node.py`:

- `NodeStageVolume` (lines 104–179) does the _volume-wide_ setup once per
  node: attach/resolve device → fetch key → format-or-open → mount at the
  **global staging path** (`request.staging_target_path`, one per volume
  per node, shared across however many pods on that node use it — though
  `ValidateVolumeCapabilities` restricts this driver to `ReadWriteOnce`
  today, so in practice at most one pod).
- `NodePublishVolume` (lines 208–237) does the _pod-specific_ step: a
  bind-mount from the staging path to `request.target_path`
  (the pod's actual mount point). Read-only is implemented as **two**
  mount calls — `mount --bind` then `mount -o remount,ro,bind` on the same
  target — because (per `SECURITY.md` **H2**, now fixed, and the inline
  logic) Linux silently ignores the `ro` flag on the initial bind mount; a
  separate remount is required to actually enforce it. **[VERIFIED, and
  DOC-corroborated]**

## 14. Unpublishing and unstaging

**[VERIFIED]**, `node.py`:

- `NodeUnpublishVolume` (lines 239–253): unmounts `target_path` if mounted.
  Simple, no LUKS interaction — the device stays open for other
  potential publishes against the same staged mount.
- `NodeUnstageVolume` (lines 181–206): unmounts `staging_path` (falls back
  to `umount -fl`, lazy force-unmount, if the plain `umount` fails — e.g.
  path still busy), then `luks.luks_close_robust(mapper)`
  (`luksClose`, falling back to `dmsetup remove --force` then
  `--deferred` on failure — mirrors, per its own docstring, "the janitor
  job logic from the upstream kopf operator"), then
  `device.release_staged(volume_id)` — deletes the staging pod (no-op if
  none exists, e.g. local/hostPath volumes never had one), which lets
  kubelet drive the backing driver's own detach.

## 15. Deletion and key-retention policies

**[VERIFIED]**, `controller.py::DeleteVolume` (lines 139–172):

1. Parse `namespace`/`pvc_name` back out of `volume_id`
   (`"namespace/pvc-name"`).
2. `k8s.get_pv_volume_attributes_by_pvc(pvc_name, namespace)` — reads the
   **still-live** backing PV's `volume_context` attributes
   (`deletionPolicy`, `vaultMount`, `vaultPath`, `volumeName`) _before_
   anything is deleted, since once the PVC/PV are gone there's nowhere
   else to recover them from. Returns `{}` (not an exception) on any 404
   along the way.
3. If `deletionPolicy == "Delete"` (the default, `manifests/storageclass.yaml`
   line 26 / `values.yaml` line 63) **and** both `vault_path_prefix` and
   `volume_name` are non-empty: `vault_mod.delete_secret(...)` —
   `delete_metadata_and_all_versions`, a **permanent** delete including
   version history, not a soft-delete. `deletionPolicy == "Retain"` skips
   this, leaving the key recoverable.
4. `k8s.delete_pvc(pvc_name, namespace)` unconditionally — safe to call if
   already gone (`404` swallowed).

Per `SECURITY.md` **M3**: deleting the backing PVC returns the block device
to the storage pool with LUKS ciphertext still physically present; only the
key is destroyed (rendering it cryptographically inaccessible), there's no
`cryptsetup erase` or explicit wipe. **[DOC]**, not independently
re-verified beyond confirming `luks.py` has no `erase`/wipe function at all
— true, grepped.

## 16. Key rotation

There are **two distinct rotation-related mechanisms** in this codebase, and
they interact only loosely:

**(a) The actual rotation mechanism — `node.py::_open_with_rotation`**
(lines 68–99), invoked from `NodeStageVolume` whenever a device is already
LUKS-formatted (line 164): fetches the _current_ Vault version and key; if
no mapper is open yet, tries `luks_open` with the current key; if that fails
with `"No key available"` **and** `ver >= 2` (refuses to "rotate" when
there's no prior version to roll back from), fetches the _previous_ Vault
version's key (`version=ver-1`), does `luks_add_key(current, using
old_key)` then `luks_remove_key(old_key)`, then finally `luks_open` with
the current key. This is real, exercised on every mount of an
already-formatted device, and mirrors the README's claimed parity with the
upstream operator's rekey Job (`luksAddKey` + `luksRemoveKey`).
**[VERIFIED]**

**(b) The visibility mechanism — `main.py::_vault_sync_loop`**
(lines 97–105), started only in `controller`/`all` mode
(`main.py` line 120), runs `_sync_vault_versions()` every
`VAULT_SYNC_INTERVAL` (default 30s) seconds. Per its own docstring (lines
56–62), its purpose is purely observational: annotate each managed PV with
`luks.csi.example.com/vault-version` so an operator can `kubectl describe
pv` and see the current key version, mirroring the upstream kopf operator's
timer.

> **[DRIFT — newly found in this pass, not previously flagged]**: this
> function lists PVs with
> `label_selector=f"csi-driver={_DRIVER_NAME}"` (line 66, i.e.
> `csi-driver=luks.csi.example.com`). I grepped the **entire** Python
> source and every manifest/Helm template for any place that sets a label
> with key `csi-driver` on a PV — there is none. The only `labels=` in the
> whole codebase (`device.py` line 282) is
> `app.kubernetes.io/managed-by: luks-csi-driver` on the **staging pod**,
> not any PV. Nothing else in the controller path
> (`k8s.py::create_pvc`, `wait_for_pvc_bound`) sets labels either, and
> `external-provisioner` does not add arbitrary custom labels to the PVs it
> creates by default. **Conclusion: as currently coded, this
> `list_persistent_volume` call matches zero PVs, every single time, so the
> annotation side of key rotation (mechanism (b) above) never actually
> annotates anything** — the background thread runs forever, harmlessly,
> logging nothing found. This does **not** affect actual rotation
> correctness (mechanism (a), the one that matters for data access, is
> independent and works), only the operator-visibility annotation feature.
> \*\*[VERIFIED down to "no label-setting code exists"; INFERENCE that
>
> > external-provisioner doesn't add it either — worth a 30-second live check
> > (`kubectl get pv -o yaml | grep csi-driver`) before treating this as fully
> > confirmed, since I have not run this against the live k3s cluster.]\*\*

## 17. Failure and retry behaviour

**[VERIFIED]**, patterns repeated across modules:

- **Idempotent creates**: `k8s.create_pvc` checks-then-creates (read first,
  404 → create); `vault.ensure_secret` read-then-create-on-InvalidPath;
  `device.ensure_staged` read-pod-then-create-on-404.
- **Idempotent teardown**: `k8s.delete_pvc`, `device.release_staged`,
  `vault.delete_secret` all swallow "already gone" (404 /
  `InvalidPath`) rather than erroring.
- **Bounded polling with explicit timeouts, not infinite retry**:
  `wait_for_pvc_bound` (120s), `wait_for_device` (120s), `ensure_staged`'s
  pod-Running wait (300s) — all raise `TimeoutError`/`RuntimeError` on
  expiry rather than hanging forever; gRPC surfaces this as
  `INTERNAL` back to the CSI caller (kubelet/external-provisioner), which
  is expected to retry the whole RPC per the CSI spec's own retry
  semantics.
- **Exceptions bubble to `grpc.StatusCode.INTERNAL` with `str(e)` as the
  detail** in both `controller.py` and `node.py` top-level handlers — per
  `SECURITY.md` **H1**, this returns raw internal detail (cryptsetup
  stderr, paths) to the RPC caller. **[DOC]**, consistent with what I read
  in both files (`context.set_details(str(e))` appears in every handler's
  `except` block).
- **Background threads never crash the process**: both
  `main.py::_vault_sync_loop` and the per-PV loop inside
  `_sync_vault_versions` catch `Exception` broadly and continue/return
  rather than propagating. **[VERIFIED]**

## 18. Idempotency — summary table

| Operation                         | Idempotency mechanism                 | Source                       |
| --------------------------------- | ------------------------------------- | ---------------------------- |
| `CreateVolume` (backing PVC)      | read-then-create                      | `k8s.py::create_pvc`         |
| `CreateVolume` (Vault key)        | read-then-create-on-InvalidPath       | `vault.py::ensure_secret`    |
| `CreateVolume` (Event)            | deterministic event name + ignore 409 | `k8s.py::emit_event`         |
| `NodeStageVolume` (staging pod)   | read-then-create-on-404               | `device.py::ensure_staged`   |
| `NodeStageVolume` (luksOpen)      | `mapper_exists()` check first         | `luks.py::luks_open`         |
| `NodeStageVolume` (mount)         | `_is_mounted()` check first           | `node.py::NodeStageVolume`   |
| `NodePublishVolume` (bind mount)  | `_is_mounted()` check first           | `node.py::NodePublishVolume` |
| `NodeUnstageVolume` (luksClose)   | `mapper_exists()` check first         | `luks.py::luks_close_robust` |
| `DeleteVolume` (PVC)              | delete + ignore 404                   | `k8s.py::delete_pvc`         |
| `DeleteVolume` (Vault key)        | delete + ignore InvalidPath           | `vault.py::delete_secret`    |
| `NodeUnstageVolume` (staging pod) | delete + ignore 404                   | `device.py::release_staged`  |

All **[VERIFIED]** from the line references given earlier in this document.

---

## Corrected architecture diagram

```{mermaid}
flowchart TB
    subgraph cluster["Kubernetes Cluster"]
        subgraph userNS["User Namespace"]
            pod(["User Pod — unprivileged"])
            upvc["User PVC\nstorageClassName: luks-encrypted"]
        end

        subgraph ctrlPod["LUKS Controller Deployment · kube-system"]
            ep["external-provisioner sidecar\nwatches provisioner=luks.csi.example.com"]
            ctrl["controller.py\nCreateVolume / DeleteVolume"]
            ctrlK8s["k8s.py"]
            ctrlVault["vault.py"]
            vsync["main.py _vault_sync_loop\n(controller/all mode only)"]
            ep --> ctrl --> ctrlK8s & ctrlVault
        end

        subgraph nodePod["LUKS Node DaemonSet · per node · privileged, attachRequired=false"]
            registrar["node-driver-registrar sidecar\nregisters csi.sock with kubelet"]
            nodeSvc["node.py\nNodeStage/Publish/Unpublish/Unstage"]

            subgraph devPy["device.py"]
                direction TB
                stage["ensure_staged() / release_staged()\npause pod pinned to this node,\nmounts backing PVC as block device"]
                subgraph resolvers["RESOLVERS, tried in order"]
                    direction TB
                    r1["1 LocalResolver"]
                    r2["2 LonghornResolver"]
                    r3["3 CephRBDResolver"]
                    r4["4 ByIdResolver (fallback)"]
                end
                stage --> resolvers
            end

            luksPy["luks.py — cryptsetup wrappers"]
            nodeVault["vault.py"]
            nodeSvc --> devPy & luksPy & nodeVault
        end

        kapi[("Kubernetes API")]
        kubelet(["kubelet"])

        subgraph backDrv["Backing CSI Driver (Longhorn / Ceph / Cinder / EBS / ...)"]
            backCtrl["Controller + external-attacher"]
            backNode["Node Plugin"]
            backCtrl --> backNode
        end

        storage[("Backing raw block storage")]
    end

    vault[("HashiCorp Vault · KV v2\nsecret/tenants/{institution}/luks-keys/{volume}")]

    pod --> upvc
    upvc -- "unbound PVC" --> ep
    ctrlK8s <-- "backing PVC create/wait-bound\nEvents" --> kapi
    ctrlVault <-- "ensure_secret (idempotent)" --> vault
    vsync -. "list PVs label=csi-driver=...\n(matches none — see Phase 2 sec 16)" .-> kapi

    registrar -. "register CSI socket" .-> kubelet
    kubelet -- "NodeStageVolume etc." --> nodeSvc
    stage -- "create/delete pause pod\n(namespace of backing PVC)" --> kapi
    kapi -- "kubelet schedules pause pod →\nnormal CSI attach for backing driver" --> backCtrl
    backNode --> storage
    nodeVault <-- "read key / rotate\n(re-authenticates every call)" --> vault

    classDef luks fill:#fff3e0,stroke:#e65100,color:#212121
    classDef vaultStyle fill:#f3e5f5,stroke:#6a1b9a,color:#212121
    classDef k8sStyle fill:#e3f2fd,stroke:#0d47a1,color:#212121
    classDef backStyle fill:#e8f5e9,stroke:#1b5e20,color:#212121
    classDef resolverStyle fill:#fafafa,stroke:#bdbdbd,color:#212121
    classDef warn fill:#ffebee,stroke:#c62828,color:#212121

    class ep,ctrl,ctrlK8s,ctrlVault,registrar,nodeSvc,luksPy,nodeVault,stage luks
    class r1,r2,r3,r4 resolverStyle
    class vault vaultStyle
    class kapi,kubelet k8sStyle
    class backCtrl,backNode,storage backStyle
    class vsync warn
```

---

## Sequence diagrams — one per RPC

### CreateVolume

```{mermaid}
sequenceDiagram
    autonumber
    actor User
    participant K8s as Kubernetes API
    participant EP as external-provisioner
    participant Ctrl as controller.py
    participant K8sPy as k8s.py
    participant Vault as vault.py / Vault

    User->>K8s: create PVC (storageClassName=luks-encrypted)
    K8s->>EP: unbound PVC event (provisioner=luks.csi.example.com)
    EP->>Ctrl: CreateVolume(name, parameters, capacity)
    Ctrl->>Ctrl: validate backingStorageClass param present
    Ctrl->>K8sPy: create_pvc(backing name, ns, backingSC, size)
    Note right of K8sPy: read-then-create — no-op if exists
    K8sPy->>K8s: create backing PVC (Block mode)
    K8sPy->>K8s: poll every 3s, up to 120s
    K8s-->>K8sPy: PVC Bound, pv_name
    Ctrl->>Vault: ensure_secret(mount, path, volumeName)
    Note right of Vault: read first; create token_hex(32)\nonly on InvalidPath
    Vault-->>Ctrl: key version
    opt csi.storage.k8s.io/pvc-name present
        Ctrl->>K8sPy: emit_event(LuksKeyProvisioned)
        Note right of K8sPy: deterministic event name,\nignores 409 on retry
    end
    Ctrl-->>EP: CreateVolumeResponse(volume_id="ns/pvc",\nvolume_context={backingPvName, vaultMount,\nvaultPath, volumeName, luksType,\nfilesystem, deletionPolicy})
    EP->>K8s: create PV, bind User PVC
    K8s-->>User: PVC Bound
```

### NodeStageVolume

```{mermaid}
sequenceDiagram
    autonumber
    participant kbl as kubelet
    participant Node as node.py
    participant Dev as device.py
    participant K8s as Kubernetes API
    participant BDrv as Backing CSI Driver
    participant Vault as vault.py / Vault
    participant LUKS as luks.py

    kbl->>Node: NodeStageVolume(volume_id, staging_path, volume_context)
    Node->>Dev: attach_and_resolve(volume_id, pv_name,\nbackingPvcName, backingPvcNamespace, nodeName)
    Dev->>K8s: read_persistent_volume(pv_name)
    alt PV is CSI-backed (not local/hostPath)
        Dev->>K8s: get/create pause pod pinned to nodeName,\nmounting backing PVC as block device
        K8s->>BDrv: kubelet schedules pod → triggers backing\ndriver's normal ControllerPublish + NodeStage
        BDrv-->>K8s: block device now present on node
        Dev->>Dev: poll pod phase every 3s, up to 300s until Running
    else local / hostPath PV
        Note right of Dev: ensure_staged is a no-op
    end
    Dev->>Dev: resolve_device_path(pv) via RESOLVERS
    Dev->>Dev: wait_for_device — poll every 3s, up to 120s
    Dev-->>Node: block device path
    Node->>Vault: read_secret(vaultMount, vaultPath, volumeName)
    Vault-->>Node: current LUKS key
    alt device not yet LUKS-formatted
        Node->>LUKS: luks_format(device, key, luksType)
        Node->>LUKS: luks_open(device, mapper, key)
        Node->>LUKS: make_filesystem(mapper, filesystem)
    else already LUKS-formatted
        Node->>Node: _open_with_rotation(...) — see Key Rotation diagram
    end
    Node->>Node: mkdir staging_path; mount /dev/mapper/<mapper>\nif not already mounted
    Node-->>kbl: NodeStageVolumeResponse
```

### NodePublishVolume

```{mermaid}
sequenceDiagram
    autonumber
    participant kbl as kubelet
    participant Node as node.py

    kbl->>Node: NodePublishVolume(staging_path, target_path, readonly)
    Node->>Node: mkdir target_path
    alt target_path not already mounted
        Node->>Node: mount --bind staging_path target_path
        opt readonly requested
            Node->>Node: mount -o remount,ro,bind target_path target_path
            Note right of Node: separate remount required —\nLinux ignores 'ro' on the initial bind
        end
    end
    Node-->>kbl: NodePublishVolumeResponse
```

### NodeUnpublishVolume

```{mermaid}
sequenceDiagram
    autonumber
    participant kbl as kubelet
    participant Node as node.py

    kbl->>Node: NodeUnpublishVolume(target_path)
    alt target_path is mounted
        Node->>Node: umount target_path
    end
    Node-->>kbl: NodeUnpublishVolumeResponse
```

### NodeUnstageVolume

```{mermaid}
sequenceDiagram
    autonumber
    participant kbl as kubelet
    participant Node as node.py
    participant LUKS as luks.py
    participant Dev as device.py
    participant K8s as Kubernetes API
    participant BDrv as Backing CSI Driver

    kbl->>Node: NodeUnstageVolume(volume_id, staging_path)
    alt staging_path is mounted
        Node->>Node: umount staging_path
        opt umount fails (busy)
            Node->>Node: umount -fl staging_path (lazy fallback)
        end
    end
    Node->>LUKS: luks_close_robust(mapper)
    alt luksClose succeeds
        LUKS-->>Node: mapper closed
    else luksClose fails
        LUKS->>LUKS: dmsetup remove --force mapper
        LUKS->>LUKS: dmsetup remove --deferred mapper
    end
    Node->>Dev: release_staged(volume_id)
    alt staging pod exists
        Dev->>K8s: delete pause pod (grace_period=0)
        K8s->>BDrv: kubelet releases pod → triggers backing\ndriver's normal NodeUnstage/ControllerUnpublish
    else no staging pod (local/hostPath)
        Note right of Dev: no-op
    end
    Node-->>kbl: NodeUnstageVolumeResponse
```

### DeleteVolume

```{mermaid}
sequenceDiagram
    autonumber
    participant EP as external-provisioner
    participant Ctrl as controller.py
    participant K8sPy as k8s.py
    participant K8s as Kubernetes API
    participant Vault as vault.py / Vault

    EP->>Ctrl: DeleteVolume(volume_id = "namespace/pvc-name")
    Ctrl->>Ctrl: parse namespace, pvc_name from volume_id
    Ctrl->>K8sPy: get_pv_volume_attributes_by_pvc(pvc_name, namespace)
    K8sPy->>K8s: read PVC → read bound PV
    K8s-->>K8sPy: PV volume_context (or {} on 404)
    K8sPy-->>Ctrl: deletionPolicy, vaultMount, vaultPath, volumeName
    alt deletionPolicy == "Delete" and vault fields present
        Ctrl->>Vault: delete_secret(mount, path, volumeName)
        Note right of Vault: delete_metadata_and_all_versions —\npermanent, all version history gone
    else deletionPolicy == "Retain"
        Note right of Ctrl: Vault key left in place
    end
    Ctrl->>K8sPy: delete_pvc(pvc_name, namespace)
    K8sPy->>K8s: delete backing PVC (ignore 404)
    Ctrl-->>EP: DeleteVolumeResponse
```

### Key rotation

```{mermaid}
sequenceDiagram
    autonumber
    participant Node as node.py
    participant Vault as vault.py / Vault
    participant LUKS as luks.py

    Note over Node,LUKS: Triggered inside NodeStageVolume, only on the\n"already LUKS-formatted" branch — see _open_with_rotation

    Node->>Vault: current_version(mount, path, volumeName)
    Vault-->>Node: ver (latest Vault key version)
    Node->>Vault: read_secret(mount, path, volumeName)
    Vault-->>Node: current_key (version=ver)

    alt mapper not already open
        Node->>LUKS: luks_open(device, mapper, current_key)
        alt open succeeds — device already uses current key
            LUKS-->>Node: mapper open, done
        else fails with "No key available"
            alt ver >= 2 (a previous version exists)
                Node->>Vault: read_secret(mount, path, volumeName, version=ver-1)
                Vault-->>Node: prev_key
                Node->>LUKS: luks_add_key(device, current_key, auth=prev_key)
                Node->>LUKS: luks_remove_key(device, prev_key)
                Node->>LUKS: luks_open(device, mapper, current_key)
                LUKS-->>Node: mapper open with rotated key
            else ver < 2
                Node->>Node: re-raise — no prior version to roll back from
            end
        end
    else mapper already open
        Note right of Node: skip — nothing to do
    end

    Note over Node,Vault: Separately, main.py::_vault_sync_loop polls every\n30s and tries to annotate PVs with the current\nversion for operator visibility — but its\nlabel_selector matches no PVs in this codebase\n(see section 16). Rotation itself is unaffected;\nonly the "kubectl describe pv" visibility is.
```

---

## Findings summary (new in this pass)

1. **[DRIFT, confirmed with more evidence]** — VolumeAttachment handling:
   diagrams/comments describe direct `VolumeAttachment` management; actual
   mechanism is a pause-pod trick, confirmed independently via
   `CSIDriver.attachRequired: false`, absence of Controller(Un)Publish RPCs,
   and RBAC content. See §8.
2. **[DRIFT, new]** — `manifests/rbac.yaml` (raw manifest deployment path)
   grants the node ServiceAccount `volumeattachments` permissions it never
   uses, and withholds `pods` permissions that `device.py::ensure_staged`
   actually needs for any CSI-backed backing storage. The Helm chart's RBAC
   is correct. See §8.
3. **[BUG, new]** — `main.py::_vault_sync_loop`'s PV `label_selector`
   (`csi-driver=luks.csi.example.com`) matches no PV in this codebase,
   because nothing sets that label anywhere. The rotation-visibility
   annotation feature is a no-op as currently wired; actual key rotation
   (`node.py::_open_with_rotation`) is unaffected. See §16.

No files were modified — this is Phase 2 output only. Per `CLAUDE.md`, any
fix (RBAC alignment, the label selector, the stale `.md`/comment drift)
should be proposed as a diff and confirmed before implementation.
