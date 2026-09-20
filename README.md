# cluster-wordpress-mysql

Kubernetes manifests for a WordPress + MySQL (MariaDB) environment.

## Contents

| File | Purpose |
|---|---|
| `mysql-secret.yaml` | Database credentials: `root-password` for administration, `user`/`password` for the WordPress account (local/study use only, see below) |
| `mysql-pvc.yaml` | 2Gi volume for the database (default `standard` storage class) |
| `mysql-deployment.yaml` | MariaDB 10.6, single replica, with probes and resource limits |
| `mysql-service.yaml` | Internal `ClusterIP` service (`mysql:3306`) used by WordPress |
| `wordpress-pvc.yaml` | 2Gi volume for `/var/www/html` (`csi-hostpath-sc`, `ReadWriteMany`) |
| `wordpress-deployment.yaml` | WordPress (`wordpress:6-apache`) with probes and resource limits |
| `wordpress-service.yaml` | `NodePort` service exposing WordPress |
| `wordpress-hpa.yaml` | Autoscaling by CPU (1 to 5 replicas, target 50%) |

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

Enable all three before deploying:

```bash
minikube addons enable metrics-server
minikube addons enable csi-hostpath-driver
minikube addons enable volumesnapshots
minikube addons list | grep enabled
```

Why each one is needed:

| Addon | Why | Symptom if missing |
|---|---|---|
| `metrics-server` | `wordpress-hpa.yaml` scales on CPU usage and needs the metrics API | `kubectl get hpa` shows `TARGETS: <unknown>` and autoscaling never happens |
| `csi-hostpath-driver` | Creates the `csi-hostpath-sc` storage class used by `wordpress-pvc.yaml` | `wordpress-pvc` stays `Pending`, and so does the WordPress pod |
| `volumesnapshots` | The CSI hostpath driver depends on the snapshot CRDs | The driver fails to install cleanly |

Addons are enabled **per cluster**. After a `minikube delete` you must enable them again.

### Optional tools

- **stern** — tails logs from multiple pods at once: `brew install stern`
- **hey** — HTTP load generator to test the HPA: `brew install hey`
- **k9s**, **kubectx/kubens** — terminal UI and fast context/namespace switching

## Deploying

Apply in this order and wait for the database before starting WordPress:

```bash
kubectl apply -f mysql-secret.yaml -f mysql-pvc.yaml -f wordpress-pvc.yaml
kubectl apply -f mysql-deployment.yaml -f mysql-service.yaml
kubectl rollout status deployment/mysql-deployment

kubectl apply -f wordpress-deployment.yaml -f wordpress-service.yaml -f wordpress-hpa.yaml
kubectl rollout status deployment/wordpress-deployment
```

Why this order: the Secret and the PVCs must exist before the pods that reference them, otherwise the pods stay `Pending` or fail with `CreateContainerConfigError`. WordPress needs MySQL reachable to finish its first-run setup, so waiting for the database avoids a noisy restart loop.

Check the result:

```bash
kubectl get pods,pvc
```

Both PVCs must be `Bound` (meaning a real volume was attached to each claim) and both pods `Running` and `1/1`.

## Accessing the site

```bash
minikube service wordpress --url
```

**Keep that terminal open.** With the Docker driver on macOS, minikube does not expose the NodePort directly. It opens a tunnel to `127.0.0.1` on a random port, and the tunnel exists only while the command is running. If you close it (or press Ctrl+C), the port stops answering and the browser shows `ERR_CONNECTION_REFUSED`. Running the command again gives you a **different port**.

Alternative with a fixed port:

```bash
kubectl port-forward service/wordpress 8080:80
```

Then open `http://localhost:8080`.

### Why the site URL follows the request host

WordPress stores its address (`home` and `siteurl`) in the database at install time and redirects any request that uses a different host or port. Since the tunnel port changes on every run, a site installed on `127.0.0.1:50001` would redirect to that dead port later and fail.

To avoid this, `wordpress-deployment.yaml` sets `WORDPRESS_CONFIG_EXTRA`, which defines `WP_HOME` and `WP_SITEURL` from the host of each request. These constants take precedence over the values stored in the database, so the site works on whatever port the tunnel or port-forward uses.

Things to know:

- This relies on the `Host` header sent by the client. That is fine for a local study environment. In a real environment, set the real public URL explicitly instead.
- It requires the WordPress **6.x** image. The old `wordpress:4.8-apache` image ignores `WORDPRESS_CONFIG_EXTRA`.
- The setting is read on every request, so it also works on an existing volume. No file needs to be regenerated.

## Resetting the environment

Choose the level that matches what you want to lose.

### Level 1: reset only the project (keeps the cluster and addons)

```bash
kubectl delete -f .
kubectl get all,pvc,secret      # only service/kubernetes should remain
```

**Risk: this deletes the database and all WordPress content.** The storage classes use `RECLAIMPOLICY: Delete`, so removing a PVC also removes the underlying volume and its data. There is no undo.

If a resource is stuck in `Terminating`, a pod is still using the PVC. Delete that pod (`kubectl delete pod <name>`) and the PVC finishes deleting on its own. Avoid removing finalizers manually: it can leave orphaned volumes behind.

### Level 2: reset 100% (new cluster)

```bash
# 1. Close any terminal running "minikube service ..." (Ctrl+C)

# 2. Delete the whole cluster
minikube delete

# 3. Create a new one
minikube start

# 4. Enable the addons again (they do not survive a delete)
minikube addons enable metrics-server
minikube addons enable csi-hostpath-driver
minikube addons enable volumesnapshots

# 5. Check
kubectl config current-context   # must be "minikube"
minikube addons list | grep enabled
```

Then deploy again following the [Deploying](#deploying) section.

**Risks:**

- `minikube delete` removes **everything** in the default profile, including workloads from any other project you run in it. List profiles first with `minikube profile list`, or use a dedicated profile (`minikube start -p <name>`).
- Nothing is backed up by either level. If the data matters, dump the database first:
  ```bash
  kubectl exec deploy/mysql-deployment -- sh -c 'mysqldump -uroot -p"$MYSQL_ROOT_PASSWORD" wordpress' > backup.sql
  ```
  Uploaded files live in the `wordpress-pvc` volume and are not covered by that dump.
- `minikube delete --purge` also deletes `~/.minikube`, including cached images, so the next start downloads everything again.

When to use which: Level 1 for day-to-day resets and re-testing the install. Level 2 when the cluster itself is in a strange state (stuck volumes, broken addons) and you want a guaranteed clean slate.

### Changing the WordPress version on an existing volume

Do not just change the image tag on a volume that already holds WordPress files. The image only copies the WordPress core into `/var/www/html` when the directory is empty, so a newer PHP running old core files crashes with fatal errors (for example `__autoload() is no longer supported`). Reset the volume (Level 1) when moving between major versions on a lab environment, or back up and replace the core files deliberately.

## Database user

WordPress does not connect as `root`. `mysql-deployment.yaml` creates a dedicated `wordpress` user (from the `user` and `password` keys of `mysql-secret`) with privileges only on the `wordpress` database. `root` stays available for administration through `root-password`.

Why: if the WordPress application is compromised (a vulnerable plugin, for example), the attacker gets access to a single database instead of the whole server, and cannot create users or read other databases.

Check it:

```bash
kubectl exec deploy/mysql-deployment -- sh -c 'mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SHOW GRANTS FOR wordpress@\"%\""'
```

Important: the MariaDB image creates `MYSQL_USER` only on the **first initialization of an empty volume**. If you change `user` or `password` in the Secret later, the existing database keeps the old account and WordPress fails with `Access denied for user`. Either change the password inside MariaDB (`ALTER USER`) or reset the project (Level 1 in [Resetting the environment](#resetting-the-environment)), which recreates the volume and loses the data.

## About `mysql-secret.yaml`

The `mysql-secret.yaml` file is intentionally committed to this repository. It holds credentials for a **local/study environment**, not production, so versioning it is fine.

If this project evolves into a real environment (staging/production), generate the secret from environment variables or a secrets vault (e.g. `kubectl create secret ... --dry-run=client -o yaml`, Sealed Secrets, SOPS, Vault) and stop committing the file with plain-text values.
