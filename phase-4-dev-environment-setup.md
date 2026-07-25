# LUKS CSI Driver — Phase 4: Dev Environment Setup

Covers `.claude/skills/csi-contribution/SKILL.md` Phase 4: uv, grpcio-tools,
k3s, cryptsetup, Vault/OpenBao. Labeling convention as in the Phase 2/3
documents:

- **[VERIFIED]** — read directly from the cited file.
- **[DOC]** — stated in `README.md`/`CLAUDE.md`, not independently re-derived.
- **[DRIFT]** — two sources in the repo disagree.
- **[INFERENCE]** — my reading of intent, not a direct quote.
- **[GAP]** — something `CLAUDE.md` or the skill asks about that has no
  in-repo script/config backing it; flagged rather than invented.

Files verified this pass: `pyproject.toml`, `uv.lock` (header only),
`generate_proto.sh`, `Dockerfile`, `.github/workflows/build.yaml`,
`testing/test-ceph.sh`, `testing/test-longhorn.sh`,
`testing/lima-k3s-ceph.yaml`, `manifests/controller.yaml`,
`manifests/node.yaml`, `luks-csi-driver/values.yaml`,
`luks-csi-driver/templates/_helpers.tpl`, `luks_csi_driver/vault.py`,
`README.md` (Vault prerequisites + Lima sections).

---

## 0. There are three separate "dev environment" flows in this repo — pick one

**[VERIFIED]**, no single canonical setup — the repo documents/scripts three
distinct paths, of increasing realism:

| Flow                        | Where                                                 | Backing storage                     | Automation                                         |
| --------------------------- | ----------------------------------------------------- | ----------------------------------- | -------------------------------------------------- |
| A. Minimal Lima quick-start | `README.md` "Local development with Lima"             | `local-path` + a manual loop device | Manual steps in README, no script                  |
| B. Longhorn dev             | `testing/test-longhorn.sh`                            | Longhorn (CSI)                      | Fully scripted, idempotent, `--teardown` supported |
| C. Ceph (Rook) dev          | `testing/test-ceph.sh` + `testing/lima-k3s-ceph.yaml` | Rook-Ceph RBD (CSI)                 | Fully scripted, idempotent, `--teardown` supported |

**[GAP]**: `CLAUDE.md`'s Environment section describes a fourth thing — "Local
cluster: k3s, 3 nodes (`csi-nfcs-k3s-controller` + 2 workers)" — a
persistent, named, multi-node cluster. I found **no script, Lima template,
or manifest anywhere in this repo** that provisions a cluster matching that
description; all three flows above are single-node Lima VMs. If that 3-node
cluster is what you're actually developing against, its setup is either
undocumented in-repo or lives outside this repository — worth confirming
before assuming any command below targets it.

Flows B and C are the ones that matter most for Phase 4, because they're the
only ones that actually exercise the parts of the driver most worth testing
(the `device.py` CSI-backed staging-pod path from Phase 2 §8, and real key
rotation) — flow A's `local-path` backing skips `ensure_staged` entirely
(Phase 2 §8), so it never touches that code path.

---

## 1. Python environment — `uv`

**[VERIFIED]**, `pyproject.toml`:

- `requires-python = ">=3.12"` (line 5). No `.python-version` file in the
  repo root — confirmed absent.
- Runtime deps: `grpcio>=1.60.0`, `grpcio-tools>=1.60.0`,
  `kubernetes>=29.0.0`, `hvac>=2.3.0` (lines 6–11). No dev-dependency group
  is defined in `pyproject.toml` — no `[dependency-groups]` or
  `[tool.uv]` dev section present, so there's currently no separate test/lint
  tooling declared (pytest, ruff, etc. are absent from the file). Per
  `CLAUDE.md`'s "never invent commands" rule, I'm not assuming a lint/test
  command exists beyond what Phase 5 will need to verify directly.
- `[project.scripts]`: `luks_csi_driver = "luks_csi_driver.main:main"` (line 14) — this is what makes `uv run luks_csi_driver` (or the installed
  console script) equivalent to `python -m luks_csi_driver.main`.

**[VERIFIED]** the Dockerfile's own toolchain, useful as a reference for what
"known good" looks like: `pip install uv==0.11.19`, then
`uv sync --frozen --no-dev --no-install-project --no-editable --no-cache`
before proto generation, and `uv sync --frozen --no-dev --no-editable
--no-cache` again after (Dockerfile lines 14–24). Base image is
`python:3.13.13-slim-trixie` (pinned by digest, line 2).

Local setup, mirroring what the Dockerfile does but without `--frozen`'s
strictness assumptions (which requires `uv.lock` to already match
`pyproject.toml` exactly — true right now since both are checked in):

```bash
pip install uv   # or your platform's uv installer
uv sync
```

`uv sync` (no `--no-dev`, no `--frozen`) is the standard "make me a working
`.venv`" command for local iteration; I'm not asserting a specific uv
version is required locally — the Dockerfile pins `0.11.19` for
reproducible CI builds, but any reasonably current `uv` should resolve this
lockfile. **[INFERENCE]** on that last point — not verified against multiple
uv versions.

---

## 2. `grpcio-tools` — generating the CSI stubs

**[VERIFIED]**, `generate_proto.sh` (executable, root of repo):

```bash
bash generate_proto.sh
```

What it actually does (lines 1–33):

1. Runs `python -m grpc_tools.protoc` against `proto/csi.proto`, writing
   `csi_pb2.py` and `csi_pb2_grpc.py` into `luks_csi_driver/generated/`.
2. **Patches** the generated `csi_pb2_grpc.py` in place: `grpc_tools`
   emits a bare `import csi_pb2`, which only works if `generated/` is on
   `sys.path` directly. The script rewrites that to
   `from luks_csi_driver.generated import csi_pb2` (lines 18–30) so it
   works as a proper package-relative import — this is why you can't skip
   straight to running `protoc` yourself without also applying this rewrite.
3. Touches `luks_csi_driver/generated/__init__.py` to make it a package.

**[VERIFIED]**: `luks_csi_driver/generated/` is not checked into git —
confirmed via `.gitignore` categorization is consistent with the README's
own claim ("The `generated/` directory is not included in the repository.
Run this once after cloning"). This means **every fresh clone requires
running `generate_proto.sh` before `main.py` will import successfully** —
`main.py` line 21 does `from luks_csi_driver.generated import csi_pb2_grpc`
unconditionally.

Run it with the project's own `uv`-managed interpreter (so `grpc_tools` is
guaranteed present) rather than a bare system `python`:

```bash
uv run bash generate_proto.sh
```

This exact invocation is what the Dockerfile itself uses (line 23) — not
invented, copied directly from the build.

---

## 3. k3s — three concrete paths, verified per-script

### Path A — README's minimal Lima quick-start

**[VERIFIED against README lines 351–485]**: `limactl create --name=k3s
template://k3s` (Lima's built-in k3s template, not a file in this repo),
then a manual loop-device (`/tmp/test-block.img` → `/dev/loop0`) set up
inside the VM, then `kubectl apply -f manifests/{csidriver,rbac,
storageclass,controller,node}.yaml`. **Note**: per Phase 2 §8's finding,
`manifests/rbac.yaml` is the RBAC file that's out of sync with the code
(grants `volumeattachments`, not `pods`) — but this exact path never
surfaces the gap because `local-path`/loop-device PVs skip the staging-pod
code entirely. If you deviate from this path (e.g. swap in a CSI-backed
`backingStorageClass` while still using raw manifests), you'll want the
Helm chart's RBAC instead, or to patch `manifests/rbac.yaml` first.

### Path B — `testing/test-longhorn.sh`

**[VERIFIED]**, reading the script directly (not the README's description
of it, which doesn't cover Longhorn at all — this script's coverage exists
only in the script's own header comment and body):

```bash
bash testing/test-longhorn.sh                 # full run
bash testing/test-longhorn.sh --skip-build     # reuse existing image
bash testing/test-longhorn.sh --teardown       # tear down everything it created
```

Prerequisites stated in the script's own header (lines 1–12): Lima, a
running Lima VM named exactly `k3s`, a running Lima VM named exactly
`docker`, `helm` on the host. It does **not** stand up Vault externally —
it deploys its own throwaway Vault dev server as part of the run (§5
below).

Steps the script actually performs, in order (verified against the code,
not summarized from a comment): install `open-iscsi` in the VM (Longhorn's
own requirement, unrelated to this driver) → install Longhorn via Helm
_inside_ the Lima VM → deploy a Vault dev pod + configure Kubernetes auth →
build and `docker save | ctr images import` the driver image → `helm
upgrade --install` this driver's chart with
`--set storageClass.backingStorageClass=longhorn` (grep-verified further in
the file past what's quoted above, consistent with the values.yaml default
key name) → apply `testing/longhorn-test-resources.yaml` → wait for the PVC
to bind and the test pod to run → assert file contents.

### Path C — `testing/test-ceph.sh` + `testing/lima-k3s-ceph.yaml`

**[VERIFIED]**. This one needs a **separate, heavier** Lima VM — not the
same `k3s` VM Path B uses:

```bash
limactl start testing/lima-k3s-ceph.yaml --name k3s-ceph   # one-time
bash testing/test-ceph.sh
bash testing/test-ceph.sh --skip-build --skip-ceph --skip-vault  # fast re-run
bash testing/test-ceph.sh --teardown
```

`testing/lima-k3s-ceph.yaml`'s own header (lines 1–23) explains why it's a
separate template: Rook-Ceph needs more CPU/RAM (`cpus: 6`, `memory:
12GiB`, `disk: 60GiB`, lines 20–23) than a base k3s VM, plus a **dedicated
raw block device** for the Ceph OSD — provisioned as a 20GiB sparse file
attached via `losetup` as `/dev/loop10` (lines 65–80), specifically because
Lima's `additionalDisks` mechanism mounts disks as filesystems, which Rook
can't claim as raw block devices (comment, lines 6–8). k3s itself is started
with `--https-listen-port 6445` (not the default 6443) specifically so it
can coexist with a Path B `k3s` VM running simultaneously on the same host
(comment, line 43).

Same downstream shape as Path B: Vault dev pod, image build+import, `helm
upgrade --install ... --set
storageClass.backingStorageClass=rook-ceph-block` (verified in the
StorageClass heredoc preceding the Vault step), test resources, assertion.

**[INFERENCE]**: Paths B and C are the two flows that genuinely exercise the
CSI-backed device-resolution and staging-pod code (`LonghornResolver`,
`CephRBDResolver`, `device.py::ensure_staged`) discussed in Phase 2 — worth
using one of these, not Path A, if the thing being changed touches
`device.py` or the RESOLVERS registry.

---

## 4. `cryptsetup`

Two separate, non-overlapping places `cryptsetup` needs to exist —
conflating them is an easy mistake:

**[VERIFIED]** **(a) Inside the driver's own container image** — this is
what actually runs LUKS operations at request time. `Dockerfile` installs it
in the builder stage (`apt-get install ... cryptsetup e2fsprogs xfsprogs
util-linux`, line 6) and copies just the binary (plus `blkid`, `blockdev`,
`dmsetup`, `mount`, `umount`, `mountpoint`, `mkfs.ext4`, `mkfs.xfs`) into the
final `distroless/cc-debian13` image (lines 38–40) — a minimal-surface
runtime with no shell or package manager, only the specific binaries
`luks.py` shells out to. You never need `cryptsetup` on your own workstation
to run the driver — only inside this image, which the `Dockerfile` already
handles.

**[VERIFIED]** **(b) On the k3s VM/node itself**, independent of (a) — the
README's own verification step at the end of the Lima walkthrough runs
`limactl shell k3s -- sudo cryptsetup status $(ls /dev/mapper/ | grep
^luks- | head -1)` (README lines 467–479) _directly on the VM_, not inside
the driver's container — this is a manual sanity check that the LUKS mapper
the containerized driver created is visible and correctly typed from the
host's perspective, and it requires a `cryptsetup` binary on the VM itself.
This is exactly why `testing/lima-k3s-ceph.yaml`'s provisioning script
separately `apt-get install`s `cryptsetup` on the VM (lines 57–63, comment:
"needed by our LUKS CSI driver on the node") — confirmed this is the same
concern, not a coincidence.

`node.yaml`/`values.yaml` also set `CRYPTSETUP_UDEV_SYNC_DISABLE=1`
(`manifests/node.yaml` lines 33–35, and the Helm equivalent) — **[VERIFIED]**
comment explains this disables cryptsetup's udev synchronisation, "to
prevent hangs in container environments where udev is not running." Worth
knowing if you ever see `cryptsetup` operations hang rather than fail
cleanly in a stripped-down container/VM.

---

## 5. Vault (and the OpenBao gap)

### What's actually scripted and working: a Vault **dev-mode** server

**[VERIFIED]**, identical block in both `test-longhorn.sh` (lines ~96–170)
and `test-ceph.sh` (lines ~260–352) — not paraphrased, this is what the
scripts literally do:

1. Create a dedicated `vault` ServiceAccount + a **long-lived** token
   Secret (`type: kubernetes.io/service-account-token` annotated with
   `kubernetes.io/service-account.name: vault`) — the script's own comment
   explains why: default projected SA tokens expire in ~1h and can't serve
   as a stable `token_reviewer_jwt` for Vault's Kubernetes auth config.
2. `kubectl create clusterrolebinding vault-auth-delegator
--clusterrole=system:auth-delegator --serviceaccount=default:vault` —
   required so Vault's Kubernetes auth method can call the
   TokenReview API to validate JWTs presented by the controller/node pods.
3. Deploy a single **`hashicorp/vault:1.15`** pod in `-dev` mode
   (`-dev-root-token-id=root -dev-listen-address=0.0.0.0:8200`) — dev mode
   auto-unseals and enables KV v2 at `secret/` by default, which is why
   there's no separate "enable KV v2" step here (unlike the README's
   production-oriented Vault prerequisites section, which does have one).
4. `vault auth enable kubernetes`, `vault write auth/kubernetes/config
kubernetes_host=... token_reviewer_jwt=<the long-lived token>`.
5. `vault policy write luks-policy -` — grants `create,read,update,delete,list`
   on `secret/data/tenants/*` and `read,list,delete` on
   `secret/metadata/tenants/*`. **[VERIFIED — narrower than README's
   documented policy]**: the README's Vault prerequisites section (lines
   194–215) additionally grants `list` on `secret/metadata/tenants` and
   `secret/metadata`, plus `read` on `sys/internal/ui/mounts/secret`; the
   test scripts' inline policy omits all three. Not necessarily a bug — the
   driver's own code (`vault.py`) never calls anything under
   `sys/internal/ui/mounts` or does a bare `list` at the `tenants`
   collection level, so the narrower policy appears sufficient for what the
   driver actually does — but it's a real difference between the two
   documented policies, worth knowing if you copy one instead of the other.
6. `vault write auth/kubernetes/role/luks-csi-role
bound_service_account_names=luks-csi-driver-controller,luks-csi-driver-node
bound_service_account_namespaces=kube-system policies=luks-policy
ttl=24h`.

### [DRIFT] — the Vault role name in step 6 does not match the README

This is a concrete, verified mismatch worth fixing before it costs someone a
debugging session:

- Test scripts (both) and `luks-csi-driver/values.yaml` line 39 use role
  name **`luks-csi-role`** — confirmed the Helm chart's `VAULT_ROLE` env var
  comes directly from `.Values.vault.role`
  (`luks-csi-driver/templates/_helpers.tpl` lines 79–80), so a plain `helm
install` with defaults will have the controller/node pods authenticate
  against Vault role `luks-csi-role`.
- `README.md`'s "Vault prerequisites" section (lines 178–186) documents
  creating role **`luks-operator-role`**, and explicitly claims _"The role
  name (`luks-operator-role`) matches the default in
  `luks-csi-driver/values.yaml`"_ — **this claim is currently false**; the
  actual default in that file is `luks-csi-role`, not `luks-operator-role`.
- `manifests/controller.yaml` / `manifests/node.yaml` (raw manifest path)
  hardcode `VAULT_ROLE value: "luks-operator-role"` in the container env —
  consistent with the README, **inconsistent with the Helm chart's
  default**. `vault.py`'s own Python-level fallback
  (`os.environ.get("VAULT_ROLE", "luks-operator-role")`, line 17) also
  defaults to `luks-operator-role`.

**Net effect**: the raw-manifest deployment path and the README's
documented Vault role name agree with each other (`luks-operator-role`).
The Helm chart's default and the two working test scripts agree with each
other (`luks-csi-role`). The two _groups_ disagree. If you follow the
README's Vault setup steps verbatim and then `helm install` with default
values, the controller/node pods will present `role=luks-csi-role` to Vault,
which was never created — Vault auth will fail. **[VERIFIED down to "these
strings don't match"; I have not reproduced the resulting Vault error
message live, but the mismatch itself is unambiguous from source.]** Fix is
either: pass `--set vault.role=luks-operator-role` when using the README's
Vault setup with Helm, or create the Vault role as `luks-csi-role` to match
Helm's actual default, or update the README. Not fixing this here per
`CLAUDE.md` — flagging for a decision before any diff.

### OpenBao — genuinely in-progress, not yet a distinct dev-setup path

**[VERIFIED]**: I grepped the entire repository (source, manifests, Helm
chart, values, testing scripts, workflows) for `openbao`/`bao` — the only
hits are in `CLAUDE.md` itself and this skill's own phase list. There is
**no OpenBao-specific test script, values flag, or documentation section**
anywhere in the repo yet.

What _does_ exist, per `git log`, is groundwork that makes `vault.py`
backend-agnostic rather than OpenBao-specific support per se:

- `8dceb62` / `ba937a1` "fix: allow configuration of the kubernetes auth
  mount path" → `VAULT_AUTH_MOUNT` env var (`vault.py` line 19, default
  `"kubernetes"`), used in `get_client()`'s
  `client.auth.kubernetes.login(..., mount_point=VAULT_AUTH_MOUNT)` (line
  35).
- `f3203ce` / `8c4514e` "fix: allow namespace support for openbao / vault" →
  `VAULT_NAMESPACE` env var (line 20), passed as `hvac.Client(url=...,
namespace=VAULT_NAMESPACE or None)` (line 34).

**[INFERENCE]**: since OpenBao maintains an API-compatible surface with
Vault's KV v2 and Kubernetes auth endpoints, and `vault.py` talks to Vault
purely through the generic `hvac` client rather than any Vault-specific
SDK feature, pointing `VAULT_ADDR` at a running OpenBao instance and setting
`VAULT_AUTH_MOUNT`/`VAULT_NAMESPACE` as needed _should_ work — but this is
an inference from how the client is used, not something I've run against a
live OpenBao instance, and there is no OpenBao dev-server script analogous
to the Vault ones in `testing/` to verify it against. If Phase 4 work
requires an actual OpenBao dev loop, that script doesn't exist yet and
would need to be written (likely a near-copy of the Vault dev-pod block in
`test-longhorn.sh`/`test-ceph.sh`, swapping the container image and init
commands) — worth a design proposal (Phase 8 territory) rather than
guessing at OpenBao's dev-mode equivalent here.

---

## Summary — recommended setup sequence

For work that doesn't touch `device.py`/resolvers (e.g. `vault.py`,
`controller.py` provisioning logic):

```bash
uv sync
uv run bash generate_proto.sh
# README "Local development with Lima" path (A) is enough
```

For work touching device resolution, staging, or anything CSI-backed
(`device.py`, RBAC, rotation under real Vault version bumps):

```bash
uv sync
uv run bash generate_proto.sh
# then Path B (testing/test-longhorn.sh) or Path C (testing/test-ceph.sh),
# whichever backing driver is relevant to the change
```

Before relying on the README's Vault prerequisites section together with a
Helm install, resolve the `luks-operator-role` vs `luks-csi-role` mismatch
above — either override `--set vault.role=luks-operator-role` or align the
Vault role name to `luks-csi-role`.

---

## Open items surfaced in this pass

1. **[DRIFT, new]** — Vault role name: README (`luks-operator-role`) vs.
   Helm chart default + working test scripts (`luks-csi-role`). See §5.
2. **[DRIFT, minor]** — Vault policy in the test scripts is narrower than
   the README's documented policy (missing two `list` grants and one
   `sys/internal/ui/mounts` read) — appears harmless given what `vault.py`
   actually calls, but the two documented policies aren't identical.
3. **[GAP]** — `CLAUDE.md` references a persistent 3-node k3s cluster
   (`csi-nfcs-k3s-controller` + 2 workers) with no corresponding
   provisioning script/config found in-repo; all three documented dev flows
   are single-node Lima VMs.
4. **[GAP]** — No OpenBao dev-server script exists yet, unlike Vault's two
   working dev-pod blocks in `testing/*.sh`. Backend-agnostic env vars
   (`VAULT_AUTH_MOUNT`, `VAULT_NAMESPACE`) are in place, but nothing in the
   repo has exercised them against a real OpenBao instance.

No files were modified — Phase 4 output only, per `CLAUDE.md`.
