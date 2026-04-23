# Atuin Sync Server — Diamond K8s (argus)

Deploys [Atuin](https://github.com/atuinsh/atuin) as a personal shell history sync server on the Diamond Light Source Kubernetes cluster.

## Prerequisites

- `module load argus` done and `kubectl` in your PATH
- Default namespace set to your fedid: `kubectl config set-context --current --namespace=rto52325`

## Deploy

### 1. Create the PVC

Create the PVC once and keep it separate from the server deployment. This ensures `kubectl delete -f atuin_server.yaml` never touches your data:

```bash
kubectl apply -f atuin_pvc.yaml
kubectl get pvc atuin-data
# Wait until STATUS = Bound before continuing
```

### 2. Deploy the server

```bash
kubectl apply -f atuin_server.yaml
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

SQLite database is stored in a 1 Gi PersistentVolumeClaim (`atuin-data`) on the default NFS-backed `netapp` StorageClass, defined in `atuin_pvc.yaml`. Data persists across pod restarts and redeployments of `atuin_server.yaml`, but is **not backed up automatically** — see the [Diamond storage docs](https://dev-guide.diamond.ac.uk/kubernetes/tutorials/storage/) for backup options.

## Teardown

To remove the Deployment and Service while **preserving** the database:

```bash
kubectl delete -f atuin_server.yaml
```

To also permanently destroy the database:

```bash
kubectl delete -f atuin_pvc.yaml
```

> **Warning:** Deleting the PVC will permanently delete the database and all sync history. This cannot be undone.
