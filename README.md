# cluster-wordpress-mysql

Kubernetes manifests for a WordPress + MySQL environment.

## Prerequisites

This project is meant to run locally via [minikube](https://minikube.sigs.k8s.io/docs/start/). Install:

- **kubectl** — `brew install kubectl`
- **minikube** — `brew install minikube`
- A VM/container driver for minikube (e.g. Docker Desktop, `brew install --cask docker`)

Start the cluster:

```bash
minikube start
kubectl config current-context   # must be "minikube"
```

### Required addons

`wordpress-hpa.yaml` (CPU-based autoscaling) depends on the **metrics-server** addon:

```bash
minikube addons enable metrics-server
minikube addons list | grep metrics-server   # confirm "enabled"
```

Without this addon, `kubectl get hpa` shows `TARGETS: <unknown>` and autoscaling does not work.

### Optional tools (used in the troubleshooting guide)

- **stern** — tails logs from multiple pods at once: `brew install stern`
- **hey** — HTTP load generator to test the HPA: `brew install hey`

## About `mysql-secret.yaml`

The `mysql-secret.yaml` file is intentionally committed to this repository. It holds credentials for a **local/study environment**, not production, so versioning it is fine.

If this project evolves into a real environment (staging/production), generate the secret from environment variables or a secrets vault (e.g. `kubectl create secret ... --dry-run=client -o yaml`, Sealed Secrets, SOPS, Vault) and stop committing the file with plain-text values.
