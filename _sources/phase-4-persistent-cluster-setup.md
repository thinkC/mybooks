# Phase 4 (continued): Vault + driver setup against the persistent k3s cluster

This documents the actual procedure used to stand up a Vault dev server and
the LUKS CSI driver against the real, persistent 3-node k3s cluster
described in `CLAUDE.md` (`csi-nfcs-k3s-controller` + `csi-nfcs-k3s-node1` +
`csi-nfcs-k3s-node2`, `KUBECONFIG=~/.kube/nfcs-k3s-csi`) — **no Lima**, per
`CLAUDE.md`'s explicit instruction to target this cluster directly rather
than the Lima-based flows in `README.md`/`testing/*.sh`.

Everything below was actually run and verified against the live cluster in
this session — not copied from docs. Where something surfaced a real
finding (not just "it worked"), that's called out explicitly.

---

## 0. Prerequisites specific to this cluster

- `KUBECONFIG=~/.kube/nfcs-k3s-csi` — already configured, cluster already
  running (per `CLAUDE.md`).
- **No SSH shell access to the nodes.** Access is via a restricted,
  forced-command SSH key (`~/.ssh/csi-control-agent`) that can only invoke
  one fixed script (`/usr/local/sbin/csi-image-import`) on each of
  `csi-control`, `csi-worker1`, `csi-worker2` — reached through a
  `ProxyJump` to an `openstack-bastion` host. This key cannot run arbitrary
  commands, `scp`, or open a real shell; it is scoped to exactly one action.
- This means **there is no way to create a loop device or run `cryptsetup
status` directly on a node**, unlike the README's Lima walkthrough. Use
  `kubectl exec` into the driver's own node pod instead (see §5) — the
  distroless final image has no shell, so invoke binaries directly
  (`kubectl exec ... -- /usr/sbin/cryptsetup ...`), not via `sh -c`.
- An existing OpenStack Cinder CSI plugin (`cinder.csi.openstack.org`) was
  already deployed in `kube-system` before this work started, but no
  Cinder `StorageClass` existed yet — one (`cinder-sc`, `type:
arcus-ceph01-rbd`) was created separately (by the repo owner, not by this
  procedure) before proceeding. `local-path` (the cluster's only
  pre-existing StorageClass) was avoided as backing storage because it's a
  Filesystem-only dynamic provisioner and this driver always requests a
  **Block**-mode backing PVC (`k8s.py::create_pvc`) — untested here, but
  likely incompatible.

---

## 1. Build and import the driver image (no registry, no Lima)

```bash
docker build -t luks-csi:dev .
```

Uses the repo's own `Dockerfile` unmodified — proto generation
(`generate_proto.sh`) happens inside the build, no separate step needed.

Import into each node's containerd via the forced-command key, same
`docker save | ... ctr images import -` pattern the repo's own
`testing/test-longhorn.sh`/`test-ceph.sh` use for Lima, just piped through
SSH to a real host instead of `limactl shell`:

```bash
for h in csi-control csi-worker1 csi-worker2; do
  docker save luks-csi:dev | ssh "$h"
done
```

Verified all three imported the identical image (same manifest digest,
`sha256:9ec6...`, confirmed in each node's import output).

**Getting the forced-command key working required two independent SSH
config fixes**, in case this is needed again on a future node:

1. `ProxyJump ubuntu@<bastion-IP>` (a raw IP) doesn't pick up the bastion's
   own `Host` block/`IdentityFile` in `~/.ssh/config` — `Host` pattern
   matching is against the literal string used to open the connection, not
   the resolved hostname. Fix: `ProxyJump <bastion-alias>` (or
   `ProxyJump user@<bastion-alias>`), not the raw IP.
2. Duplicate `Host csi-control`/`csi-worker1`/`csi-worker2` blocks in the
   config — `ProxyJump` is a single-value directive (first match in the
   file wins), so a corrected second block was silently ignored until the
   first, stale block was removed/commented out.
3. The forced-command script itself was initially copied to `/tmp/` on
   each node, not `/usr/local/sbin/` — `sudo: ... command not found` until
   `sudo install -o root -g root -m 0755 /tmp/csi-image-import
/usr/local/sbin/csi-image-import` was run on each node.

---

## 2. Vault dev server (namespace `default`)

Adapted directly from the Vault block in `testing/test-longhorn.sh` /
`testing/test-ceph.sh`, with the Lima wrapping removed — every command runs
straight against `KUBECONFIG=~/.kube/nfcs-k3s-csi`.

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  name: vault
  namespace: default
---
apiVersion: v1
kind: Secret
metadata:
  name: vault-reviewer-token
  namespace: default
  annotations:
    kubernetes.io/service-account.name: vault
type: kubernetes.io/service-account-token
EOF

kubectl create clusterrolebinding vault-auth-delegator \
  --clusterrole=system:auth-delegator \
  --serviceaccount=default:vault \
  --dry-run=client -o yaml | kubectl apply -f -

until kubectl get secret vault-reviewer-token -n default \
    -o jsonpath='{.data.token}' 2>/dev/null | grep -q .; do sleep 2; done
```

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: vault
  namespace: default
  labels:
    app: vault
spec:
  serviceAccountName: vault
  containers:
    - name: vault
      image: hashicorp/vault:1.15
      args: ["server", "-dev", "-dev-root-token-id=root", "-dev-listen-address=0.0.0.0:8200"]
      ports:
        - containerPort: 8200
      securityContext:
        capabilities:
          add: ["IPC_LOCK"]
---
apiVersion: v1
kind: Service
metadata:
  name: vault
  namespace: default
spec:
  selector:
    app: vault
  ports:
    - port: 8200
      targetPort: 8200
EOF

kubectl wait pod/vault -n default --for=condition=Ready --timeout=60s
until kubectl exec vault -n default -- sh -c \
    'VAULT_ADDR=http://127.0.0.1:8200 vault status' &>/dev/null; do sleep 2; done
```

```bash
REVIEWER_JWT=$(kubectl get secret vault-reviewer-token -n default \
  -o jsonpath='{.data.token}' | base64 -d)

kubectl exec vault -n default -- sh -c "
  export VAULT_TOKEN=root VAULT_ADDR=http://127.0.0.1:8200
  vault auth enable kubernetes 2>/dev/null || true
  vault write auth/kubernetes/config \
    kubernetes_host=https://kubernetes.default.svc:443 \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
    token_reviewer_jwt='${REVIEWER_JWT}'
  vault policy write luks-policy - <<POLICY
path \"secret/data/tenants/*\" { capabilities = [\"create\",\"read\",\"update\",\"delete\",\"list\"] }
path \"secret/metadata/tenants/*\" { capabilities = [\"read\",\"list\",\"delete\"] }
path \"secret/metadata/tenants\" { capabilities = [\"list\"] }
path \"secret/metadata\" { capabilities = [\"list\"] }
path \"sys/internal/ui/mounts/secret\" { capabilities = [\"read\"] }
POLICY
  vault write auth/kubernetes/role/luks-csi-role \
    bound_service_account_names=luks-csi-driver-controller,luks-csi-driver-node \
    bound_service_account_namespaces=kube-system \
    policies=luks-policy \
    ttl=24h
"
```

Role name (`luks-csi-role`) and SA names
(`luks-csi-driver-controller`/`luks-csi-driver-node`) match the now-fixed
README conventions (see `readme-vault-role-fix.md`) for a Helm install with
release name `luks-csi-driver`.

**Caveat**: this is a `-dev` mode Vault with a well-known root token
(`"root"`) — fine for this disposable test instance, not for anything
resembling production, per `CLAUDE.md`'s "test keys ... only" rule.

---

## 3. Deploy the driver via Helm

```bash
helm install luks-csi-driver ./luks-csi-driver/ \
  --namespace kube-system \
  --set vault.address="http://vault.default.svc.cluster.local:8200" \
  --set vault.role="luks-csi-role" \
  --set storageClass.backingStorageClass="cinder-sc" \
  --set apparmor.enabled=false \
  --set apparmor.annotate=false
```

AppArmor disabled — the profile hasn't been loaded on these real nodes
(same pattern `testing/test-ceph.sh`/`test-longhorn.sh` use).

**Blast radius accepted for this run** (discussed and confirmed before
applying): the node `DaemonSet` is privileged, mounts host `/dev`, and —
since none of the three nodes carry scheduling taints, including
`csi-nfcs-k3s-controller` (the control-plane node) — runs on all three,
control-plane included. Decided not to taint the controller node for this
pass, since it would remove a third of cluster capacity cluster-wide for
every workload, not just this driver.

Verified: `kubectl rollout status` clean on both the `Deployment` and
`DaemonSet`; `2/2 Running` on all three nodes; `luks.csi.example.com`
present alongside `cinder.csi.openstack.org` in every node's `CSINode`
object (kubelet registration confirmed).

---

## 4. End-to-end test

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cinder-luks-test-pvc
  namespace: default
spec:
  storageClassName: luks-encrypted
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: luks-cinder-test-pod
  namespace: default
spec:
  containers:
    - name: app
      image: alpine:3.19
      command:
        - sh
        - -c
        - |
          echo 'hello from luks+cinder' > /mnt/data/test.txt
          cat /mnt/data/test.txt
          sleep 3600
      volumeMounts:
        - name: encrypted-vol
          mountPath: /mnt/data
  volumes:
    - name: encrypted-vol
      persistentVolumeClaim:
        claimName: cinder-luks-test-pvc
EOF
```

**Result: fully successful.** PVC bound within seconds; pod reached
`Running`; `kubectl logs luks-cinder-test-pod` printed
`hello from luks+cinder`, confirming a real write+read through the LUKS
mapper.

### What was independently verified along the way

- **Vault key auto-provisioned**: `vault kv get -mount=secret
tenants/default/luks-keys/pvc-4577cb34-...` returned the real generated
  key (`version 1`), confirming `controller.py::CreateVolume` →
  `vault.py::ensure_secret` end to end.
- **Staging-pod mechanism confirmed live** — this is the mechanism
  documented (but never previously observed running) in
  `phase-2-architecture-expanded.md` §8: a pod named
  `luks-stage-<hash>` appeared in `kube-system`, pinned to the node running
  the test pod, mounting the backing PVC as a raw block device
  (`volumeDevices`, `/dev/xvda`), went `Pending → ContainerCreating →
Running` in ~6 seconds, and this is exactly what caused the backing
  Cinder volume to actually attach to the node. Confirms the Helm chart's
  RBAC (`pods: get,list,watch,create,delete` on the node role) is correct
  and sufficient — no permission errors at any point.
- **Real LUKS encryption confirmed directly** via
  `kubectl exec -n kube-system <node-pod> -c luks-csi -- /usr/sbin/cryptsetup
status <mapper-name>` (no shell needed/available in the distroless
  image — invoke the binary directly):
  ```
  type:    LUKS2
  cipher:  aes-xts-plain64
  keysize: 512 bits
  device:  /dev/sdb
  mode:    read/write
  ```
  Device resolved to `/dev/sdb` via `ByIdResolver` (Cinder isn't handled by
  `LocalResolver`/`LonghornResolver`/`CephRBDResolver`, so it correctly fell
  through to the generic fallback, as documented).

### New finding: root cause of the `SECURITY.md` L3 / Phase 2 §15 namespace fallback

Previous passes (`architecture-and-module-review.md`, `SECURITY.md` L3)
documented that the backing PVC lands in `kube-system` when
`csi.storage.k8s.io/pvc-namespace` isn't injected, without knowing _why_ it
wasn't injected. This run found the actual cause:

**The external-provisioner sidecar is missing `--extra-create-metadata`.**
Confirmed directly:

```
$ kubectl get deploy luks-csi-driver-controller -n kube-system \
    -o jsonpath='{.spec.template.spec.containers[?(@.name=="external-provisioner")].args}'
["--csi-address=/csi/csi.sock","--leader-election","--leader-election-namespace=kube-system","--v=5"]
```

`--extra-create-metadata` is what makes `external-provisioner` populate
`csi.storage.k8s.io/pvc-name`, `csi.storage.k8s.io/pvc-namespace`, and
`csi.storage.k8s.io/pvc-uid` in `CreateVolumeRequest.parameters` at all —
without it, none of those three keys are ever present, regardless of
provisioner version. This single missing flag explains, in one place:

- Why the backing PVC always lands in the controller's own namespace
  (`k8s.get_operator_namespace()` fallback, `controller.py` line ~73) rather
  than the requesting PVC's namespace — reproduced live in this test
  (backing PVC `luks-backing-pvc-4577cb34-...` created in `kube-system`
  while the user PVC was in `default`).
- Why no `LuksKeyProvisioned` `Event` was emitted on the test PVC —
  `controller.py::CreateVolume` only emits it
  `if user_pvc_name:` where `user_pvc_name =
params.get("csi.storage.k8s.io/pvc-name")`, which was `None` here.
  Confirmed: `kubectl get events -n default
--field-selector reason=LuksKeyProvisioned` returned nothing.

Neither `manifests/controller.yaml` nor
`luks-csi-driver/templates/controller.yaml` pass this flag to the
`external-provisioner` args. This is a small, well-scoped, verified fix
candidate (`args: [..., --extra-create-metadata]` in both places) — not
applied here, per `CLAUDE.md`'s "show the diff, get confirmation" rule; not
something to guess at implementing without a separate decision to do so.

---

## 5. Cleanup

Test resources (left running for inspection — not yet torn down):

```bash
kubectl delete pod luks-cinder-test-pod -n default
kubectl delete pvc cinder-luks-test-pvc -n default
```

Deleting the PVC triggers `DeleteVolume` → `vault.py::delete_secret`
(destroys the Vault key, `deletionPolicy: Delete` is the StorageClass
default) → backing PVC deletion → Cinder volume deletion.

To remove the whole setup:

```bash
helm uninstall luks-csi-driver --namespace kube-system
kubectl delete pod/vault svc/vault -n default
kubectl delete secret/vault-reviewer-token serviceaccount/vault -n default
kubectl delete clusterrolebinding/vault-auth-delegator
```

---

## Summary of what's now verified against this specific cluster

- Image build/import path via the forced-command SSH key (no registry, no
  Lima) — working, documented above for reuse.
- Vault dev server + Kubernetes auth + policy + role — working.
- Driver deployment via Helm, `cinder-sc` backing storage — working,
  including the CSI-backed staging-pod path (previously
  documented-but-unobserved).
- Full encrypted-volume lifecycle (`CreateVolume` → Vault key →
  `NodeStageVolume` → real LUKS2 format/open → mount → pod read/write) —
  working and independently confirmed via `cryptsetup status` inside the
  driver's own container.
- One new, concrete, fixable finding: missing `--extra-create-metadata` on
  the external-provisioner sidecar, root-causing two previously-known but
  unexplained behaviors.
