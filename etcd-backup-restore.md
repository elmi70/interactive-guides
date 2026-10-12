---
title: etcd Backup and Restore Guide
---

# etcd Backup and Restore Guide

## What is etcd and why back it up?

etcd is the **key-value store that holds all cluster state**: every pod, deployment, configmap, secret, and node registration. If you lose etcd, you lose the entire cluster. A snapshot is your only way to get it back.

## The whole process at a glance

```
BACKUP (no downtime)                       RESTORE (downtime)
────────────────────                       ──────────────────
etcdctl snapshot save  ──► snapshot.db ──► 1. Stop kube-apiserver
   (live, needs certs)      chmod 600         2. etcdutl snapshot restore → NEW data dir
                            copy off-node     3. Point etcd at the new dir
                                              4. Restart etcd
                                              5. Start kube-apiserver
                                              6. Verify with kubectl
```

## Two tools: etcdctl vs etcdutl

This is a common source of confusion. They are two different binaries with different jobs:

| | etcdctl | etcdutl |
|---|---|---|
| Purpose | Talks to a **live, running** etcd server | Works **offline**, directly on etcd data files |
| Needs etcd running? | ✅ Yes | ❌ No |
| Backup | `etcdctl snapshot save`: live snapshot over the API (**standard method**) | `etcdutl backup`: raw file copy (legacy, rarely used) |
| Restore | ⚠️ `etcdctl snapshot restore` is deprecated | ✅ `etcdutl snapshot restore` |
| Status check | `etcdctl snapshot status` (deprecated) | `etcdutl snapshot status` |
| Requires certs? | ✅ Yes, it connects over TLS | ❌ No, it reads files directly |

**Key rule:** use etcdctl to back up (live), and etcdutl to restore (offline).

Older Kubernetes docs use `etcdctl snapshot restore`. It still works on etcd 3.5, but `etcdutl snapshot restore` is the recommended command.

### Check the etcdctl API version

```bash
etcdctl version
```

```
etcdctl version: 3.5.16
API version: 3.5        ← must be v3
```

API v3 has been the default since etcd 3.4. You only need the `ETCDCTL_API=3` prefix if the output shows API version 2 (older etcd). etcdutl only supports v3, so it never needs the prefix.

## Two ways etcd can run

How etcd runs decides which restore process you follow:

| Setup | How etcd runs | How to stop/start |
|---|---|---|
| kubeadm cluster | Static pod (`/etc/kubernetes/manifests/etcd.yaml`) | Move the manifest out and back in |
| Non-kubeadm / bare metal | systemd service (`etcd.service`) | `systemctl stop/start etcd` |

Check which one you have:

```bash
kubectl get pods -n kube-system | grep etcd   # static pod: shows a pod
systemctl status etcd                          # service: shows active/inactive
```

## Certificate locations

etcdctl needs certificates for every command against a live server. The paths are different on every setup, so **always read them from the manifest or the service file** instead of guessing. The table shows typical values:

| Flag | kubeadm (typical) | systemd service (example, varies) |
|---|---|---|
| `--cacert` | `/etc/kubernetes/pki/etcd/ca.crt` | `/etc/etcd/ca.pem` |
| `--cert` | `/etc/kubernetes/pki/etcd/server.crt` | `/etc/etcd/kubernetes.pem` |
| `--key` | `/etc/kubernetes/pki/etcd/server.key` | `/etc/etcd/kubernetes-key.pem` |

Find the exact paths:

```bash
# kubeadm
grep -E 'cert|key|ca' /etc/kubernetes/manifests/etcd.yaml

# systemd service
grep -E 'cert|key|ca' /etc/systemd/system/etcd.service
```

## What each certificate does

Every etcdctl command against a live etcd server needs all three flags:

```bash
etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /opt/snapshot-pre-boot.db
```

### `--cacert`: the certificate authority

This is the **root of trust** for all etcd TLS traffic: the CA that signed every other etcd certificate.

When etcdctl connects over HTTPS, it must check that it's talking to the real etcd and not an impostor. It does this by checking that etcd's certificate was signed by this CA.

Think of a passport office: the CA is the government that issues passports, and anyone with a passport from that government is trusted.

```
without --cacert → etcdctl: "I don't know who signed your certificate. I refuse to connect."
with --cacert    → etcdctl: "I trust this CA and it signed your cert. Connection allowed."
```

On kubeadm, etcd has **its own CA** (`/etc/kubernetes/pki/etcd/ca.crt`), separate from the cluster CA (`/etc/kubernetes/pki/ca.crt`). Using the cluster CA here fails with `x509: certificate signed by unknown authority`.

### `--cert`: the client certificate

This is **who etcdctl says it is** when it connects. It's named `server.crt`, but here it's used as a **client certificate** to authenticate etcdctl to etcd.

etcd uses mutual TLS (mTLS), so both sides authenticate each other:
- etcd proves its identity to the client with its own certificate
- etcdctl proves its identity to etcd with this certificate

Without it, etcd rejects the connection because it requires clients to identify themselves.

**Which client cert should you use?** `server.crt` works because the kubeadm etcd server cert is allowed to act as both server and client. A cleaner choice is a cert made only for clients:

| Cert | Path (kubeadm) | Intended use |
|---|---|---|
| `server.crt` / `server.key` | `/etc/kubernetes/pki/etcd/` | etcd serving cert; also works as a client |
| `healthcheck-client.crt` / `.key` | `/etc/kubernetes/pki/etcd/` | Client-only, used by liveness probes |
| `apiserver-etcd-client.crt` / `.key` | `/etc/kubernetes/pki/` | Client-only, used by the API server |

All three are signed by the etcd CA, so any of them works with `--cacert=/etc/kubernetes/pki/etcd/ca.crt`. In the CKA exam, use the paths the question gives you.

### `--key`: the private key

This is the **private key that matches the certificate**. It proves etcdctl actually owns that certificate and isn't presenting a stolen copy.

A certificate is like a photo ID, and anyone can photocopy it. The private key proves the ID is really yours, because only the owner has it.

### How they work together (mTLS handshake)

```
etcdctl → connects to etcd at 127.0.0.1:2379

  Step 1: server verification (--cacert)
  etcdctl checks: "Was etcd's cert signed by ca.crt?"
  Yes → I trust this server

  Step 2: client authentication (--cert + --key)
  etcd checks: "Who is this client?"
  etcdctl presents its cert and proves ownership with the key
  etcd → identity verified, connection allowed

  Step 3: encrypted communication
  All data flows encrypted over TLS
```

Because both sides verify each other (mTLS), all three flags are needed every time.

### What happens if you leave one out

| Missing flag | What breaks |
|---|---|
| `--cacert` | etcdctl can't verify the server, so the connection is refused |
| `--cert` | etcd can't identify the client, so the connection is rejected |
| `--key` | etcdctl can't prove it owns the cert, so the TLS handshake fails |

### The simple mental model

```
--cacert  →  "I trust certificates signed by this authority"   (verify the server)
--cert    →  "Here is my identity"                            (prove who you are)
--key     →  "Here is proof that identity is mine"            (prove you own it)
```

## Backup: take a snapshot

**No downtime is needed.** etcd can be snapshotted while it's running, so don't stop anything to back up.

### Option A: etcdctl snapshot (standard method, use this in the CKA)

#### kubeadm cluster

```bash
etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /opt/snapshot-pre-boot.db
```

Expected output (log lines trimmed):

```
{"level":"info","ts":"...","caller":"snapshot/v3_snapshot.go",...}
...
Snapshot saved at /opt/snapshot-pre-boot.db
```

The last line, `Snapshot saved at /opt/snapshot-pre-boot.db`, confirms the backup finished.

#### etcd as a service

Use the cert paths from your `etcd.service` file:

```bash
etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem \
  snapshot save /opt/snapshot-pre-boot.db
```

### Option B: etcdutl file backup (legacy, rarely used)

This copies the etcd data directory directly. **Stop etcd first**, because copying the files while etcd is writing can give you an inconsistent backup. Prefer Option A unless you have a specific reason not to.

```bash
etcdutl backup \
  --data-dir /var/lib/etcd \
  --backup-dir /backup/etcd-backup
```

This copies the backend database and the WAL (write-ahead log) files. It needs no certs because it reads from disk, not from the etcd API.

### Verify the snapshot

```bash
etcdutl snapshot status /opt/snapshot-pre-boot.db --write-out=table
```

```
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| abc12345 |   475629 |       1234 |     3.2 MB |
+----------+----------+------------+------------+
```

```bash
ls -lh /opt/snapshot-pre-boot.db
```

### Protect the snapshot

⚠️ **The snapshot contains every Secret in the cluster.** Unless encryption at rest (`EncryptionConfiguration`) is turned on, those Secrets are only base64-encoded and anyone holding the file can read them.

```bash
chmod 600 /opt/snapshot-pre-boot.db
```

- Copy it **off the node**. A backup on the same disk is lost along with the node.
- Store it somewhere encrypted with restricted access.

## Restore from a snapshot

**This needs downtime.** Stop the API server before restoring, so nothing writes to etcd during the restore and the data stays consistent.

The examples below assume a **single etcd member**. For multi-member clusters, see [Multi-member clusters](#multi-member-clusters).

### Restore: etcd as a service

#### Step 1: stop the kube-apiserver

```bash
systemctl stop kube-apiserver
```

#### Step 2: restore the snapshot to a new data directory

```bash
etcdutl snapshot restore /opt/snapshot-pre-boot.db \
  --data-dir /var/lib/etcd-from-backup
```

```
... restore compact to 475629
```

- This creates a **new** data directory and never overwrites the existing one.
- etcdutl works offline, so it needs no certs and no running etcd.
- The revision number in the output shows which snapshot point was restored.

#### Step 3: give the etcd user ownership

If etcd runs as a dedicated `etcd` user, the restored directory belongs to root and etcd won't start. Check the `User=` line in `etcd.service`. If it's set, fix the owner:

```bash
chown -R etcd:etcd /var/lib/etcd-from-backup
```

This is a common CKA gotcha. If etcd fails to start after the restore, check ownership first.

#### Step 4: point etcd.service at the new data directory

```bash
vim /etc/systemd/system/etcd.service
```

Find the `--data-dir` flag and change only that line:

```
ExecStart=/usr/local/bin/etcd \
  --name ${ETCD_NAME} \
  --cert-file=/etc/etcd/kubernetes.pem \
  --key-file=/etc/etcd/kubernetes-key.pem \
  --trusted-ca-file=/etc/etcd/ca.pem \
  --client-cert-auth \
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://${INTERNAL_IP}:2379 \
  ...
  --data-dir=/var/lib/etcd-from-backup    # ← change this line
```

Save and exit.

#### Step 5: reload systemd and restart etcd

```bash
systemctl daemon-reload
systemctl restart etcd
systemctl status etcd     # should show active (running)
```

#### Step 6: start the kube-apiserver

```bash
systemctl start kube-apiserver
```

#### Step 7: verify

```bash
kubectl get nodes
kubectl get pods --all-namespaces
```

### Restore: kubeadm cluster (static pod)

`systemctl stop kube-apiserver` **doesn't work** here, because the API server is a static pod, not a service. Instead, move its manifest out of the folder and kubelet stops it automatically.

#### Step 1: stop the API server by moving its manifest

```bash
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
sleep 30
crictl ps | grep kube-apiserver    # should return nothing
```

#### Step 2: restore the snapshot to a new data directory

```bash
etcdutl snapshot restore /opt/snapshot-pre-boot.db \
  --data-dir /var/lib/etcd-from-backup
```

Expected output (trimmed):

```
... restoring snapshot
... Trimming membership information from the backend...
... added member
... restored snapshot
```

No certs are needed, because etcdutl reads the files directly.

On kubeadm, etcd runs as root inside the pod, so you don't need to change ownership.

#### Step 3: point the etcd pod at the restored directory

```bash
vi /etc/kubernetes/manifests/etcd.yaml
```

**Change only the `etcd-data` hostPath in the `volumes:` section.** Leave the `--data-dir` flag alone. The container keeps using `/var/lib/etcd` internally; you're only changing which folder on the node gets mounted there.

**From:**

```yaml
volumes:
- hostPath:
    path: /etc/kubernetes/pki/etcd
    type: DirectoryOrCreate
  name: etcd-certs
- hostPath:
    path: /var/lib/etcd              # OLD directory
    type: DirectoryOrCreate
  name: etcd-data
```

**To:**

```yaml
volumes:
- hostPath:
    path: /etc/kubernetes/pki/etcd
    type: DirectoryOrCreate
  name: etcd-certs
- hostPath:
    path: /var/lib/etcd-from-backup  # NEW restored directory
    type: DirectoryOrCreate
  name: etcd-data
```

```
Node disk                         etcd container
/var/lib/etcd-from-backup  ──►    /var/lib/etcd   (--data-dir stays the same)
```

Save and exit. Kubelet notices the manifest change and recreates the etcd pod on its own.

#### Step 4: bring the API server back

```bash
mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/
```

Wait about **60 seconds** for the API server to start fully.

#### Step 5: restart the other control-plane components

Restart the remaining components so they reconnect cleanly to the restored etcd:

```bash
# kube-controller-manager
mv /etc/kubernetes/manifests/kube-controller-manager.yaml /tmp/
sleep 20
mv /tmp/kube-controller-manager.yaml /etc/kubernetes/manifests/

# kube-scheduler
mv /etc/kubernetes/manifests/kube-scheduler.yaml /tmp/
sleep 20
mv /tmp/kube-scheduler.yaml /etc/kubernetes/manifests/

# kubelet
systemctl restart kubelet
```

#### Step 6: watch the restart

```bash
watch crictl ps
```

- Every component should show `STATUS = Running`.
- The whole process usually takes **2–3 minutes**.

#### Step 7: verify the restore

```bash
kubectl get nodes
kubectl get pods --all-namespaces
kubectl get deployments,services --all-namespaces
```

You should see the resources that existed when the snapshot was taken.

## Multi-member clusters

The steps above restore a **single** etcd member. With three members, every member must be restored from the **same snapshot**, and each one needs its own identity and the full member list.

On **each** etcd node, stop etcd and run:

```bash
etcdutl snapshot restore /opt/snapshot-pre-boot.db \
  --name <this-member-name> \
  --initial-cluster <m1>=https://<ip1>:2380,<m2>=https://<ip2>:2380,<m3>=https://<ip3>:2380 \
  --initial-cluster-token <new-token> \
  --initial-advertise-peer-urls https://<this-member-ip>:2380 \
  --data-dir /var/lib/etcd-from-backup
```

Then point each member at the new data directory and start them all. Use a new `--initial-cluster-token` so the restored cluster can't accidentally join the old one.

## Full process comparison

| Step | etcd as a service | kubeadm (static pod) |
|---|---|---|
| Stop the API server | `systemctl stop kube-apiserver` | `mv kube-apiserver.yaml /tmp/` |
| Restore the snapshot | `etcdutl snapshot restore --data-dir=...` | `etcdutl snapshot restore --data-dir=...` |
| Fix ownership | `chown -R etcd:etcd <dir>` (if etcd runs as the `etcd` user) | Not needed (runs as root) |
| Update config | Edit `--data-dir` in `etcd.service` | Edit the `etcd-data` **hostPath** in `etcd.yaml` (leave `--data-dir` alone) |
| Reload | `systemctl daemon-reload` | Not needed: kubelet detects the change |
| Restart etcd | `systemctl restart etcd` | Kubelet restarts it automatically |
| Start the API server | `systemctl start kube-apiserver` | `mv kube-apiserver.yaml` back |

## Key rules to remember

- **etcdctl**: for backups (live, needs certs).
- **etcdutl**: for restores and status checks (offline, no certs).
- **Backup**: no downtime. `etcdctl snapshot save` is safe on a running cluster.
- **Restore**: always stop the API server first so nothing writes to etcd during the restore.
- **Snapshot restore** always creates a **new** data directory and never overwrites existing data.
- After restoring, point etcd at the new directory: `--data-dir` in `etcd.service`, or the **hostPath** in `etcd.yaml`.
- On a systemd setup, **check ownership** of the restored directory.
- **Protect snapshots**: they contain every Secret. Use `chmod 600` and store them off the node.
- On kubeadm, use the **etcd CA** (`pki/etcd/ca.crt`), not the cluster CA.
- `ETCDCTL_API=3` is the default from etcd 3.4. You only need it on older versions.
- The revision in the restore output (`restore compact to XXXXX`) shows which snapshot point was used.
