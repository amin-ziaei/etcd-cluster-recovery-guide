# Recovering a Corrupted 3-Node etcd Cluster (Manual Installation)

## Overview

This document describes the recovery procedure for a manually installed three-node etcd cluster after data corruption or an unclean shutdown, where one or more members fail to start with errors such as:

- `walpb: crc mismatch`
- `failed to recover v3 backend from snapshot`
- `snapshot file doesn't exist`
- Continuous leader election

Kubernetes nodes stuck with:

```
node "<node-name>" not found
```

because the API server cannot connect to etcd.

---

## Symptoms

### Node 1

`raft: starting a new election` continuously appears, and no leader is elected.

Example:

```
context deadline exceeded
failed to publish local member
```

### Node 2

```
failed to recover v3 backend from snapshot
panic: failed to recover v3 backend from snapshot
```

or

```
snapshot file doesn't exist
```

### Node 3

```
walpb: crc mismatch
```

indicating WAL corruption.

---

## Root Cause

The etcd cluster lost quorum because multiple members had corrupted local databases.

Since the cluster could no longer elect a leader, Kubernetes API Server became unavailable, preventing kubelets from registering nodes.

---

## Recovery Strategy

Recover the cluster by:

1. Force-creating a new one-node cluster from the healthy member.
2. Remove corrupted members.
3. Add them back one at a time.
4. Allow each node to replicate data from the healthy leader.

---

## Step 1 — Recover the Healthy Node

Choose the healthiest node (Node-1 in this example).

Modify its systemd unit. Temporarily configure:

```
--initial-cluster node-1=https://192.168.50.11:2380
--initial-cluster-state new
--force-new-cluster
```

Restart:

```bash
systemctl daemon-reload
systemctl restart etcd
```

Verify:

```bash
etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/pki/ca.crt \
  --cert=/etc/etcd/pki/etcd.crt \
  --key=/etc/etcd/pki/etcd.key \
  endpoint health
```

Expected output:

```
is healthy
```

---

## Step 2 — Verify Current Cluster

```bash
etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/pki/ca.crt \
  --cert=/etc/etcd/pki/etcd.crt \
  --key=/etc/etcd/pki/etcd.key \
  member list -w table
```

Initially only Node-1 should exist.

---

## Step 3 — Add Node-2

On Node-1:

```bash
etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/pki/ca.crt \
  --cert=/etc/etcd/pki/etcd.crt \
  --key=/etc/etcd/pki/etcd.key \
  member add node-2 \
  --peer-urls=https://192.168.50.12:2380
```

Example output:

```
ETCD_NAME="node-2"
ETCD_INITIAL_CLUSTER="node-2=https://192.168.50.12:2380,node-1=https://192.168.50.11:2380"
ETCD_INITIAL_CLUSTER_STATE="existing"
```

---

## Step 4 — Prepare Node-2

Stop etcd:

```bash
systemctl stop etcd
```

Delete the corrupted database:

```bash
rm -rf /var/lib/etcd/*
```

Configure the systemd unit using the values returned by `member add`. Most importantly:

```
--initial-cluster-state existing
```

Start:

```bash
systemctl start etcd
```

Verify:

```bash
etcdctl member list -w table
```

Both nodes should be **started**.

---

## Step 5 — Add Node-3

Repeat the same process.

On Node-1:

```bash
etcdctl member add node-3 \
  --peer-urls=https://192.168.50.13:2380
```

On Node-3:

1. Stop etcd
2. Delete `/var/lib/etcd/*`
3. Update unit file
4. Start etcd

---

## Step 6 — Verify Cluster

```bash
etcdctl member list -w table
```

Expected:

| Member | Status  |
|--------|---------|
| node-1 | started |
| node-2 | started |
| node-3 | started |

Check leader:

```bash
etcdctl endpoint status -w table
```

Example:

| Endpoint | IS LEADER |
|----------|-----------|
| node-1   | true      |
| node-2   | false     |
| node-3   | false     |

> Only one member should report `IS LEADER=true`.

---

## Step 7 — Restore Normal Configuration

After the cluster is healthy, remove the recovery configuration.

- **Delete:** `--force-new-cluster`
- **Change:** `--initial-cluster-state new` → `--initial-cluster-state existing`
- **Restore** the full cluster definition:

```
--initial-cluster=node-1=https://192.168.50.11:2380,node-2=https://192.168.50.12:2380,node-3=https://192.168.50.13:2380
```

Restart:

```bash
systemctl daemon-reload
systemctl restart etcd
```

---

## Safe Shutdown Procedure

To avoid future corruption:

1. Stop workloads.
2. Stop kubelet.
3. Stop etcd.
4. Power off the server.

Example:

```bash
systemctl stop kubelet
systemctl stop etcd
shutdown -h now
```

---

## Startup Procedure

Bring the cluster online in this order:

1. Boot all control-plane nodes.
2. Wait until etcd elects a leader.
3. Verify:
   ```bash
   etcdctl endpoint status -w table
   ```
4. Confirm all members are healthy:
   ```bash
   etcdctl endpoint health
   ```
5. Verify Kubernetes:
   ```bash
   kubectl get nodes
   kubectl get pods -A
   ```

---

## Useful Verification Commands

| Command | Description |
|---------|-------------|
| `etcdctl endpoint health` | Cluster health |
| `etcdctl endpoint status -w table` | Leader status |
| `etcdctl member list -w table` | Member list |
| `etcdctl alarm list` | Alarm status |
| `etcdctl snapshot save backup.db` | Snapshot backup |

---

## Lessons Learned

- **Never** leave `--force-new-cluster` configured after recovery.
- `--force-new-cluster` should only be used for **emergency recovery**.
- Existing members must always start with `--initial-cluster-state=existing`.
- Corrupted members should **not** be repaired in place; remove their data directory and allow them to resynchronize from the healthy leader.
- **Always** maintain regular etcd snapshots to reduce recovery time and minimize the risk of data loss.
