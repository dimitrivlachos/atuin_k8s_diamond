# Atuin Sync Server — Diamond K8s (argus)

Deploys [Atuin](https://github.com/atuinsh/atuin) as a personal shell history sync server on the Diamond Light Source Kubernetes cluster.

## Prerequisites

- `module load argus` done and `kubectl` in your PATH
- Default namespace set to your fedid: `kubectl config set-context --current --namespace=rto52325`
- `envsubst` available (`gettext` package on most distros)

## Configuration

Copy `.env.example` to `.env` and fill in your values:

```bash
cp .env.example .env
```

`.env` is gitignored and must not be committed.

## Deploy

### 1. Source your environment

```bash
set -a && source .env && set +a
```

### 2. Create the PVC

Create the PVC once and keep it separate from the server deployment. This ensures `kubectl delete -f atuin_server.yaml` never touches your data:

```bash
kubectl apply -f atuin_pvc.yaml
kubectl get pvc atuin-data
# Wait until STATUS = Bound before continuing
```

### 3. Find the node and update `.env`

The PVC will be bound to a specific NVMe node. Find it and set `ATUIN_NODE_NAME` in `.env`:

```bash
kubectl get pv -l storageclass=db-nvme-storage -o wide
# Note the NODE column, update ATUIN_NODE_NAME in .env, then re-source it
set -a && source .env && set +a
```

### 4. Deploy the server

```bash
envsubst < atuin_server.yaml | kubectl apply -f -
```

Wait for the pod to be ready:

```bash
kubectl get pods -w
```

Get the external IP:

```bash
kubectl get service atuin
```

The `EXTERNAL-IP` (172.23.x.x) is your sync endpoint.

## Register your account

Open registration is enabled by default. Register once, then disable it:

```bash
atuin register -u <username> -e <email> --server http://<EXTERNAL-IP>:8888
kubectl set env deployment/atuin ATUIN_OPEN_REGISTRATION=false
```

## Configure Atuin clients

Add to `~/.config/atuin/config.toml` on each machine:

```toml
sync_address = "http://<EXTERNAL-IP>:8888"
```

Then log in and sync:

```bash
atuin login -u <username>
atuin sync
```

## Storage

SQLite database is stored in a 1 Gi PersistentVolumeClaim (`atuin-data`) on the `db-nvme-storage` StorageClass, backed by a RAID1 NVMe pair local to the node. The pod is pinned to that node via `nodeAffinity`. Data persists across pod restarts and redeployments of `atuin_server.yaml`, but is **not backed up automatically** — see the [Diamond storage docs](https://dev-guide.diamond.ac.uk/kubernetes/tutorials/storage/) for backup options.

## Teardown

To remove the Deployment and Service while **preserving** the database:

```bash
kubectl delete deployment atuin
kubectl delete service atuin
```

To also permanently destroy the database:

```bash
kubectl delete pvc atuin-data
```

> **Warning:** Deleting the PVC will permanently delete the database and all sync history. This cannot be undone.
