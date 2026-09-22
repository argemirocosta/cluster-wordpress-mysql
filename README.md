# wordpress-mysql-k8s

Kubernetes manifests for a WordPress + MySQL (MariaDB) environment, organised with [Kustomize](https://kustomize.io/) into a shared `base/` and two environments: **staging** and **production**.

## Repository layout

```
base/                       # manifests shared by every environment
  kustomization.yaml
  mysql-*.yaml              # secret, pvc, deployment, service
  wordpress-*.yaml          # pvc, deployment, service, hpa
overlays/
  staging/                  # namespace wordpress-staging
  production/               # namespace wordpress-production
```

### Base

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

All resources carry the recommended labels `app.kubernetes.io/name`, `app.kubernetes.io/component` and `app.kubernetes.io/part-of: wordpress-stack`, so the whole project can be listed at once:

```bash
kubectl get all,pvc,secret -n <namespace> -l app.kubernetes.io/part-of=wordpress-stack
```

### Environments

Each overlay takes the base, moves it into its own namespace and changes only what differs:

| | Base (`default` namespace) | Staging | Production |
|---|---|---|---|
| Namespace | `default` | `wordpress-staging` | `wordpress-production` |
| WordPress replicas | 1 | 1 | 2 |
| HPA (min to max) | 1 to 5 | 1 to 2 | 2 to 5 |
| MySQL CPU / memory | 500m / 1024Mi | 250m-500m / 512Mi | same as base |
| WordPress CPU / memory | 250m-500m / 512Mi-1024Mi | 100m-250m / 256Mi-512Mi | same as base |
| Storage (each volume) | 2Gi | 1Gi | 5Gi |
| MySQL replicas | 1 | 1 | 1 |

Notes on these choices:

- **Production keeps `minReplicas: 2` in the HPA.** Without it the autoscaler would shrink production back to a single replica when the load is low, defeating the purpose of running two.
- **MySQL always has one replica.** MariaDB is not clustered here, and two pods writing to the same volume would corrupt the data. This means the database is a single point of failure, also in production.
- **Two WordPress replicas work because `wordpress-pvc` is `ReadWriteMany`**, so both pods can mount the same volume.
- **Selectors are never changed by overlays.** A Deployment selector is immutable, so the `environment` label is added to metadata only.

To see exactly what an overlay produces, without applying anything:

```bash
kubectl kustomize overlays/staging
kubectl kustomize overlays/production
```

## Prerequisites

This project is meant to run locally via [minikube](https://minikube.sigs.k8s.io/docs/start/). Install:

- **kubectl** — `brew install kubectl` (includes Kustomize through `kubectl apply -k`)
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

### Memory

Every environment you run adds pods to the same minikube. Running `default`, staging and production together needs a lot of memory (the base alone requests about 1.5Gi). Check what minikube has (`minikube config get memory`) and delete an environment you are not using (see [Resetting](#resetting-the-environment)) if it gets tight.

### Optional tools

- **stern** — tails logs from multiple pods at once: `brew install stern`
- **hey** — HTTP load generator to test the HPA: `brew install hey`
- **k9s**, **kubectx/kubens** — terminal UI and fast context/namespace switching

## Deploying

Pick one target. Each is applied with a single command, because Kustomize renders all resources together:

```bash
kubectl apply -k base                  # namespace "default"
kubectl apply -k overlays/staging      # namespace "wordpress-staging"
kubectl apply -k overlays/production   # namespace "wordpress-production"
```

Then wait for the database and WordPress (use the namespace of the target you chose; omit `-n` for `base`):

```bash
kubectl rollout status deployment/mysql-deployment -n wordpress-staging
kubectl rollout status deployment/wordpress-deployment -n wordpress-staging
kubectl get pods,pvc -n wordpress-staging
```

Both PVCs must be `Bound` (meaning a real volume was attached to each claim) and the pods `Running` and `1/1`.

Why a single `apply -k` is enough: the Secret and the PVCs are created in the same request as the pods that reference them, and Kubernetes keeps the pods `Pending` until those dependencies exist. WordPress may restart once or twice while MySQL finishes starting; that is normal and settles by itself.

Running `kubectl apply -k` again is safe. Unchanged resources are reported as `unchanged`. The Secret may be reported as `configured`, because its `stringData` is converted to `data` by the API server.

## Accessing the site

```bash
minikube service wordpress -n wordpress-staging --url
```

Replace the namespace as needed (omit `-n` for the `default` namespace). **Keep that terminal open.** With the Docker driver on macOS, minikube does not expose the NodePort directly. It opens a tunnel to `127.0.0.1` on a random port, and the tunnel exists only while the command is running. If you close it (or press Ctrl+C), the port stops answering and the browser shows `ERR_CONNECTION_REFUSED`. Running the command again gives you a **different port**.

Alternative with a fixed port:

```bash
kubectl port-forward service/wordpress 8080:80 -n wordpress-staging
```

Then open `http://localhost:8080`.

Each environment has its own database, so each one needs its own WordPress installation in the browser the first time.

### Why the site URL follows the request host

WordPress stores its address (`home` and `siteurl`) in the database at install time and redirects any request that uses a different host or port. Since the tunnel port changes on every run, a site installed on `127.0.0.1:50001` would redirect to that dead port later and fail.

To avoid this, `base/wordpress-deployment.yaml` sets `WORDPRESS_CONFIG_EXTRA`, which defines `WP_HOME` and `WP_SITEURL` from the host of each request. These constants take precedence over the values stored in the database, so the site works on whatever port the tunnel or port-forward uses.

Things to know:

- This relies on the `Host` header sent by the client. That is fine for a local study environment. In a real environment, set the real public URL explicitly instead.
- It requires the WordPress **6.x** image. The old `wordpress:4.8-apache` image ignores `WORDPRESS_CONFIG_EXTRA`.
- The setting is read on every request, so it also works on an existing volume. No file needs to be regenerated.

## Health checks

Both deployments define the same probe timings (check every 10s, 3 failures to trip); only the checks differ:

| | Liveness (restart the container) | Readiness (receive traffic) |
|---|---|---|
| WordPress | TCP connect to port 80, after 5s | `GET /wp-login.php` on port 80, after 5s |
| MySQL | TCP connect to port 3306, after 30s | `mysqladmin ping -h 127.0.0.1`, after 5s |

Why `/wp-login.php` for WordPress readiness: it runs PHP and needs the database, so if MySQL goes down the pod is marked `0/1` and removed from the Service instead of serving errors. A static file such as `license.txt` would keep reporting "ready" with the database down.

To see it working, stop the database and watch WordPress leave the Service:

```bash
kubectl get pods -w -n wordpress-staging                                      # terminal 1
kubectl scale deployment/mysql-deployment --replicas=0 -n wordpress-staging   # terminal 2
kubectl get endpoints wordpress -n wordpress-staging                          # empty while MySQL is down
kubectl scale deployment/mysql-deployment --replicas=1 -n wordpress-staging   # restore
```

Scaling to 0 keeps the volume, so no data is lost. With a single WordPress replica the site is down anyway while the database is down; the benefit is larger with several replicas.

## Resetting the environment

Choose the level that matches what you want to lose.

### Level 1: reset one environment (keeps the cluster and addons)

```bash
kubectl delete -k overlays/staging      # or overlays/production, or base for "default"
kubectl get all,pvc,secret -n wordpress-staging
```

For the staging and production overlays this also deletes the namespace, since the overlay creates it. Deleting one environment never touches the others.

**Risk: this deletes the database and all WordPress content of that environment.** The storage classes use `RECLAIMPOLICY: Delete`, so removing a PVC also removes the underlying volume and its data. There is no undo.

If a resource is stuck in `Terminating`, a pod is still using the PVC. Delete that pod (`kubectl delete pod <name> -n <namespace>`) and the PVC finishes deleting on its own. Avoid removing finalizers manually: it can leave orphaned volumes behind.

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

- `minikube delete` removes **everything** in the default profile, including every environment here and workloads from any other project you run in it. List profiles first with `minikube profile list`, or use a dedicated profile (`minikube start -p <name>`).
- Nothing is backed up by either level. If the data matters, dump the database first:
  ```bash
  kubectl exec deploy/mysql-deployment -n wordpress-production -- sh -c 'mysqldump -uroot -p"$MYSQL_ROOT_PASSWORD" wordpress' > backup.sql
  ```
  Uploaded files live in the `wordpress-pvc` volume and are not covered by that dump.
- `minikube delete --purge` also deletes `~/.minikube`, including cached images, so the next start downloads everything again.

When to use which: Level 1 for day-to-day resets and re-testing the install. Level 2 when the cluster itself is in a strange state (stuck volumes, broken addons) and you want a guaranteed clean slate.

### Changing the WordPress version on an existing volume

Do not just change the image tag on a volume that already holds WordPress files. The image only copies the WordPress core into `/var/www/html` when the directory is empty, so a newer PHP running old core files crashes with fatal errors (for example `__autoload() is no longer supported`). Reset the volume (Level 1) when moving between major versions on a lab environment, or back up and replace the core files deliberately.

## Database user

WordPress does not connect as `root`. `base/mysql-deployment.yaml` creates a dedicated `wordpress` user (from the `user` and `password` keys of `mysql-secret`) with privileges only on the `wordpress` database. `root` stays available for administration through `root-password`.

Why: if the WordPress application is compromised (a vulnerable plugin, for example), the attacker gets access to a single database instead of the whole server, and cannot create users or read other databases.

Check it:

```bash
kubectl exec deploy/mysql-deployment -n wordpress-staging -- sh -c 'mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SHOW GRANTS FOR wordpress@\"%\""'
```

Important: the MariaDB image creates `MYSQL_USER` only on the **first initialization of an empty volume**. If you change `user` or `password` in the Secret later, the existing database keeps the old account and WordPress fails with `Access denied for user`. Either change the password inside MariaDB (`ALTER USER`) or reset that environment (Level 1 in [Resetting the environment](#resetting-the-environment)), which recreates the volume and loses the data.

## Time zone

Containers use the `TZ` environment variable (`America/Sao_Paulo`) instead of mounting a host time zone file, so timestamps in logs and in the database do not depend on the node.

Check it:

```bash
kubectl exec deploy/mysql-deployment -n wordpress-staging -- date
kubectl exec deploy/mysql-deployment -n wordpress-staging -- sh -c 'mysql -uroot -p"$MYSQL_ROOT_PASSWORD" -e "SELECT NOW(), @@system_time_zone"'
```

What `TZ` does **not** change:

- MariaDB stores `TIMESTAMP` columns in UTC internally and only converts them when reading, so the stored values are the same whatever `TZ` is.
- The WordPress time zone (Settings → General) is independent of the container `TZ`. Times shown in the WordPress admin can therefore differ from the times in the container logs until both are set to the same zone.

## About `mysql-secret.yaml`

The `base/mysql-secret.yaml` file is intentionally committed to this repository, and **all three targets (`base`, staging and production) currently use it**. It holds credentials for a **local/study environment**, so versioning it is fine here.

Do not treat the production overlay as production-ready until this changes: real production credentials must not live in Git. Generate the secret outside the repository (for example `kubectl create secret generic`, with values typed at a prompt so they stay out of the shell history) or use a secrets tool (Sealed Secrets, SOPS, External Secrets, Vault), and remove the Secret from the production overlay.
