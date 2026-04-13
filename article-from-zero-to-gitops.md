# From Zero to GitOps: Building a k3s Homelab on a Raspberry Pi with Flux & SOPS

I wanted a Kubernetes cluster I could break, rebuild, and learn from — without a cloud bill. A Raspberry Pi, a Git repo, and a few evenings later, I had a fully GitOps-managed homelab running real applications behind Cloudflare Tunnels.

This guide walks through the entire process: from a bare Pi to encrypted secrets in a public Git repo, with Flux automatically deploying everything.

---

## What We're Building

By the end, you'll have:

- A **k3s** cluster running on a Raspberry Pi
- **FluxCD** watching a Git repo and auto-deploying changes
- **SOPS + age** encrypting secrets so they're safe in a public repo
- **Audiobookshelf** and **Linkding** deployed as real workloads
- **Cloudflare Tunnels** exposing them to the internet (no port forwarding)
- **kube-prometheus-stack** for monitoring (Grafana + Prometheus)
- **Renovate** running as a CronJob to keep container images up to date

Here's what the final directory structure looks like:

```
pi-cluster/
  clusters/staging/
    flux-system/          # Flux bootstrap (auto-generated)
    apps.yaml             # Flux Kustomization for apps
    infrastructure.yaml   # Flux Kustomization for infra
    monitoring.yaml       # Flux Kustomization for monitoring
    .sops.yaml            # SOPS encryption rules
  apps/
    base/                 # Shared manifests
      audiobookshelf/
      linkding/
    staging/              # Environment-specific overlays
      audiobookshelf/
      linkding/
  infrastructure/
    controllers/
      base/renovate/      # Renovate CronJob
      staging/
  monitoring/
    controllers/          # Helm chart for kube-prometheus-stack
    configs/              # Grafana TLS secret
```

---

## Part 1: Setting Up the Pi

### Flash the OS

Use the [Raspberry Pi Imager](https://www.raspberrypi.com/software/) to write **Raspberry Pi OS (64-bit Lite)** to your SD card. In the imager settings:

- Set a hostname (e.g., `k3s-node`)
- Create a user (e.g., `pi-admin`)
- Configure WiFi (SSID + password)
- Enable SSH

### Assign a Static IP

Log into your router and create a DHCP reservation for the Pi's MAC address. For example, assign `192.168.1.11`. This ensures the Pi always gets the same IP.

### Enable Cgroups

This step is critical. k3s needs cgroups for container resource management and **will crash on startup without it**.

SSH into the Pi:

```bash
ssh pi-admin@k3s-node.local
```

Edit the boot config:

```bash
sudo nano /boot/firmware/cmdline.txt
```

> **Note:** On older Raspberry Pi OS versions, the path may be `/boot/cmdline.txt` instead.

Append to the **single existing line** (don't create a new line):

```
cgroup_memory=1 cgroup_enable=memory
```

Reboot:

```bash
sudo reboot
```

---

## Part 2: Installing k3s

SSH back in and run the installer:

```bash
curl -sfL https://get.k3s.io | sh -
```

Verify it's running:

```bash
sudo systemctl status k3s.service
```

You should see `Active: active (running)`.

### Remote Access from Your Mac

Copy the kubeconfig from the Pi:

```bash
scp pi-admin@k3s-node.local:/etc/rancher/k3s/k3s.yaml /tmp/k3s-kubeconfig.yaml
```

Edit the file — change the server address from `127.0.0.1` to your Pi's IP:

```bash
sed -i '' 's/127.0.0.1/192.168.1.11/g' /tmp/k3s-kubeconfig.yaml
```

Test the connection:

```bash
KUBECONFIG=/tmp/k3s-kubeconfig.yaml kubectl get nodes
```

```
NAME        STATUS   ROLES                  AGE   VERSION
k3s-node   Ready    control-plane,master   5m    v1.34.6+k3s1
```

---

## Part 3: Bootstrapping Flux

[FluxCD](https://fluxcd.io/) watches your Git repo and applies changes to the cluster automatically. Push a manifest, Flux deploys it. Delete a manifest, Flux removes it.

### Install the Flux CLI

```bash
brew install fluxcd/tap/flux
```

### Bootstrap

Set your GitHub token (needs `repo` scope):

```bash
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-username>
```

Bootstrap Flux:

```bash
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=pi-cluster \
  --branch=main \
  --path=./clusters/staging \
  --personal
```

This creates the `pi-cluster` repo (if it doesn't exist), installs Flux on the cluster, and configures it to watch `./clusters/staging` for changes.

Verify:

```bash
flux get kustomizations
```

```
NAME          REVISION     READY  MESSAGE
flux-system   main@sha1:…  True   Applied revision: main@sha1:…
```

---

## Part 4: SOPS + age for Secret Encryption

We want to commit Kubernetes Secrets to Git without leaking passwords and credentials. SOPS encrypts the `data` fields while leaving metadata readable.

### Generate a Key Pair

```bash
age-keygen -o age.agekey
```

Output:

```
# created: 2025-10-13T10:00:00+05:30
# public key: age1abc123...
AGE-SECRET-KEY-1...
```

Save the public key (starts with `age1...`). Keep `age.agekey` safe — anyone with this file can decrypt your secrets.

### Add the Private Key to the Cluster

```bash
cat age.agekey | kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=/dev/stdin
```

This lets Flux's kustomize-controller decrypt SOPS-encrypted files at apply time.

### Configure SOPS Rules

Create `clusters/staging/.sops.yaml`:

```yaml
creation_rules:
  - path_regex: .*.yaml
    encrypted_regex: ^(data|stringData)$
    age: age1abc123...  # your public key
```

`encrypted_regex` tells SOPS to only encrypt the `data` and `stringData` fields in YAML files — leaving `apiVersion`, `kind`, and `metadata` readable. This is important because Flux needs the unencrypted metadata to know what kind of resource it's deploying.

### Enable Decryption in Flux Kustomizations

Add the `decryption` block to each Flux Kustomization that references encrypted secrets. For example, in `clusters/staging/apps.yaml`:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 1m0s
  retryInterval: 1m
  timeout: 5m
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./apps/staging
  prune: true
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```

The key part is the `decryption` section at the bottom. Do the same for `infrastructure.yaml` and any monitoring kustomization that references encrypted secrets.

> **Important:** The `decryption` block goes in the **Flux Kustomization** resources (the ones with `apiVersion: kustomize.toolkit.fluxcd.io/v1`), not in the Kustomize `kustomization.yaml` files (the ones with `apiVersion: kustomize.config.k8s.io/v1beta1`). These are two different things with confusingly similar names.

---

## Part 5: Deploying Linkding

[Linkding](https://github.com/sissbruecker/linkding) is a self-hosted bookmark manager. We'll deploy it with persistent storage, a Cloudflare Tunnel, and encrypted secrets.

### Directory Structure

```
apps/
  base/linkding/
    namespace.yaml
    deployment.yaml
    service.yaml
    storage.yaml
    ingress.yaml
    secret-cloudflare.yaml    # SOPS encrypted
    secret-superuser.yaml     # SOPS encrypted
    kustomization.yaml
  staging/linkding/
    cloudflare.yaml           # Cloudflared deployment + config
    kustomization.yaml
```

### Base Manifests

**namespace.yaml**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: linkding
```

**deployment.yaml** — Runs as non-root (UID 33) with privilege escalation disabled:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: linkding
spec:
  replicas: 1
  selector:
    matchLabels:
      app: linkding
  template:
    metadata:
      labels:
        app: linkding
    spec:
      securityContext:
        runAsUser: 33
        runAsGroup: 33
        fsGroup: 33
      containers:
        - name: linkding
          image: sissbruecker/linkding:1.31.0
          ports:
            - containerPort: 9090
          securityContext:
            allowPrivilegeEscalation: false
          envFrom:
            - secretRef:
                name: linkding-superuser
          volumeMounts:
            - name: linkding-data
              mountPath: /etc/linkding/data
      volumes:
        - name: linkding-data
          persistentVolumeClaim:
            claimName: linkding-data-pvc
```

**service.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: linkding
spec:
  ports:
    - port: 9090
  selector:
    app: linkding
  type: ClusterIP
```

**storage.yaml** — k3s includes a built-in `local-path` StorageClass that auto-provisions PersistentVolumes. You don't need to create PVs manually. Just create a PVC:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: linkding-data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

**kustomization.yaml**

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - deployment.yaml
  - storage.yaml
  - service.yaml
  - secret-cloudflare.yaml
  - secret-superuser.yaml
  - ingress.yaml
```

### Creating Encrypted Secrets

**Superuser credentials:**

```bash
kubectl create secret generic linkding-superuser \
  --from-literal=LD_SUPERUSER_NAME=your-username \
  --from-literal=LD_SUPERUSER_PASSWORD=your-password \
  --namespace=linkding \
  --dry-run=client -o yaml > apps/base/linkding/secret-superuser.yaml
```

Encrypt it:

```bash
sops --encrypt \
  --age $(grep 'age1' clusters/staging/.sops.yaml | awk '{print $NF}') \
  --encrypted-regex '^(data|stringData)$' \
  --in-place apps/base/linkding/secret-superuser.yaml
```

The file now looks like this — the `data` values are encrypted, but everything else is readable:

```yaml
apiVersion: v1
data:
    LD_SUPERUSER_NAME: ENC[AES256_GCM,data:abc123...,type:str]
    LD_SUPERUSER_PASSWORD: ENC[AES256_GCM,data:def456...,type:str]
kind: Secret
metadata:
    name: linkding-superuser
sops:
    age:
        - recipient: age1abc123...
          enc: |
            -----BEGIN AGE ENCRYPTED FILE-----
            ...
            -----END AGE ENCRYPTED FILE-----
    encrypted_regex: ^(data|stringData)$
```

This is safe to commit to a public repo. Without the `age.agekey` private key, the encrypted values are unrecoverable.

### Setting Up the Cloudflare Tunnel

Cloudflare Tunnels let you expose services to the internet without opening ports on your router. Traffic goes: `User -> Cloudflare Edge -> Tunnel -> Your Pi`.

**Create a tunnel:**

```bash
cloudflared tunnel create ldpi
```

This generates a credentials JSON file in `~/.cloudflared/<TUNNEL_ID>.json`.

**Route DNS:**

```bash
cloudflared tunnel route dns ldpi ldpi.yourdomain.com
```

**Create and encrypt the tunnel secret:**

```bash
kubectl create secret generic tunnel-credentials \
  --from-file=credentials.json=~/.cloudflared/<TUNNEL_ID>.json \
  --namespace=linkding \
  --dry-run=client -o yaml > apps/base/linkding/secret-cloudflare.yaml

sops --encrypt \
  --age $(grep 'age1' clusters/staging/.sops.yaml | awk '{print $NF}') \
  --encrypted-regex '^(data|stringData)$' \
  --in-place apps/base/linkding/secret-cloudflare.yaml
```

### Staging Overlay — Cloudflared Deployment

Create `apps/staging/linkding/cloudflare.yaml`. This deploys `cloudflared` as a sidecar that connects to Cloudflare's edge:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudflared
spec:
  selector:
    matchLabels:
      app: cloudflared
  replicas: 2
  template:
    metadata:
      labels:
        app: cloudflared
    spec:
      containers:
        - name: cloudflared
          image: cloudflare/cloudflared:latest
          args:
            - tunnel
            - --config
            - /etc/cloudflared/config/config.yaml
            - run
          livenessProbe:
            httpGet:
              path: /ready
              port: 2000
            failureThreshold: 1
            initialDelaySeconds: 10
            periodSeconds: 10
          volumeMounts:
            - name: config
              mountPath: /etc/cloudflared/config
              readOnly: true
            - name: creds
              mountPath: /etc/cloudflared/creds
              readOnly: true
      volumes:
        - name: creds
          secret:
            secretName: tunnel-credentials
        - name: config
          configMap:
            name: cloudflared
            items:
              - key: config.yaml
                path: config.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: cloudflared
data:
  config.yaml: |
    tunnel: ldpi
    credentials-file: /etc/cloudflared/creds/credentials.json
    metrics: 0.0.0.0:2000
    no-autoupdate: true

    ingress:
      - hostname: ldpi.yourdomain.com
        service: http://linkding.linkding:9090
      - service: http_status:404
```

The `ingress` section maps hostnames to internal Kubernetes services. The service URL follows the pattern `http://<service>.<namespace>:<port>`.

**Staging kustomization** (`apps/staging/linkding/kustomization.yaml`):

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: linkding
resources:
  - ../../base/linkding
  - cloudflare.yaml
```

### Deploy

```bash
git add . && git commit -m "Add linkding with Cloudflare tunnel" && git push
```

Flux picks it up automatically. Watch it happen:

```bash
flux get kustomizations --watch
```

Verify:

```bash
kubectl get pods -n linkding
```

```
NAME                          READY   STATUS    RESTARTS   AGE
cloudflared-5b8987fbcf-8hqq2  1/1    Running   0          2m
cloudflared-5b8987fbcf-dkzl5  1/1    Running   0          2m
linkding-5ff7c4c46f-g8j92     1/1    Running   0          2m
```

Visit `https://ldpi.yourdomain.com` and log in with the superuser credentials you set.

---

## Part 6: Deploying Audiobookshelf

[Audiobookshelf](https://www.audiobookshelf.org/) is a self-hosted audiobook and podcast server. The setup follows the same pattern as Linkding.

### Base Manifests

```
apps/base/audiobookshelf/
  namespace.yaml
  deployment.yaml
  service.yaml
  configmap.yaml
  storage.yaml
  kustomization.yaml
```

**deployment.yaml** — Runs as non-root (UID 1000) with three persistent volumes:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: audiobookshelf
spec:
  replicas: 1
  selector:
    matchLabels:
      app: audiobookshelf
  template:
    metadata:
      labels:
        app: audiobookshelf
    spec:
      securityContext:
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
      containers:
        - name: audiobookshelf
          image: ghcr.io/advplyr/audiobookshelf:2.17.2
          envFrom:
            - configMapRef:
                name: audiobookshelf-configmap
          ports:
            - containerPort: 3005
              protocol: TCP
          volumeMounts:
            - mountPath: /config
              name: audiobookshelf-config
            - mountPath: /metadata
              name: audiobookshelf-metadata
            - mountPath: /audiobooks
              name: audiobookshelf-audiobooks
      volumes:
        - name: audiobookshelf-config
          persistentVolumeClaim:
            claimName: audiobookshelf-config
        - name: audiobookshelf-metadata
          persistentVolumeClaim:
            claimName: audiobookshelf-metadata
        - name: audiobookshelf-audiobooks
          persistentVolumeClaim:
            claimName: audiobookshelf-audiobooks
```

**storage.yaml** — Three PVCs for config, metadata, and audiobook files:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: audiobookshelf-config
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: audiobookshelf-metadata
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: audiobookshelf-audiobooks
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

**configmap.yaml**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: audiobookshelf-configmap
data:
  PORT: "3005"
```

### Staging Overlay

The audiobookshelf staging overlay adds a Cloudflare tunnel (same pattern) and the encrypted tunnel credentials:

```yaml
# apps/staging/audiobookshelf/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: audiobookshelf
resources:
  - ../../base/audiobookshelf/
  - secret.yaml      # SOPS-encrypted Cloudflare tunnel credentials
```

Create a separate Cloudflare tunnel for audiobookshelf:

```bash
cloudflared tunnel create audiobooks
cloudflared tunnel route dns audiobooks audiobooks.yourdomain.com
```

Then create and encrypt the secret the same way as before.

---

## Part 7: Monitoring with kube-prometheus-stack

The kube-prometheus-stack Helm chart gives you Grafana, Prometheus, and Alertmanager in one shot.

### Flux HelmRelease

Create `monitoring/controllers/base/kube-promethues-stack/release.yaml`:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: kube-prometheus-stack
  namespace: monitoring
spec:
  interval: 30m
  chart:
    spec:
      chart: kube-prometheus-stack
      version: "66.x"
      sourceRef:
        kind: HelmRepository
        name: kube-prometheus-stack
        namespace: monitoring
      interval: 12h
  install:
    crds: Create
  upgrade:
    crds: CreateReplace
  driftDetection:
    mode: enabled
    ignore:
      - paths: ["/metadata/annotations/prometheus-operator-validated"]
        target:
          kind: PrometheusRule
  values:
    grafana:
      adminPassword: your-password
      ingress:
        enabled: true
        ingressClassName: traefik
        hosts:
          - grafana.yourdomain.com
        tls:
          - secretName: grafana-tls-secret
            hosts:
              - grafana.yourdomain.com
```

**repository.yaml**

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: kube-prometheus-stack
  namespace: monitoring
spec:
  interval: 24h
  url: https://prometheus-community.github.io/helm-charts
```

Add a Flux Kustomization in `clusters/staging/monitoring.yaml` — note that monitoring-configs (which includes the Grafana TLS secret) depends on monitoring-controllers:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: monitoring-controllers
  namespace: flux-system
spec:
  interval: 1m0s
  retryInterval: 1m
  timeout: 5m
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./monitoring/controllers/staging
  prune: true
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: monitoring-configs
  namespace: flux-system
spec:
  interval: 1m0s
  retryInterval: 1m
  timeout: 5m
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./monitoring/configs/staging
  prune: true
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```

To expose Grafana via Cloudflare Tunnel, add it to your existing cloudflared config's ingress rules:

```yaml
ingress:
  - hostname: ldpi.yourdomain.com
    service: http://linkding.linkding:9090
  - hostname: audiobooks.yourdomain.com
    service: http://audiobookshelf.audiobookshelf:3005
  - hostname: grafana.yourdomain.com
    service: http://kube-prometheus-stack-grafana.monitoring:80
  - service: http_status:404
```

Then add the DNS route:

```bash
cloudflared tunnel route dns ldpi grafana.yourdomain.com
```

---

## Part 8: Automated Dependency Updates with Renovate

[Renovate](https://docs.renovatebot.com/) scans your repo for outdated container image tags and opens PRs to update them. We run it as a Kubernetes CronJob.

```yaml
# infrastructure/controllers/base/renovate/cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: renovate
  namespace: renovate
spec:
  schedule: "@hourly"
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: renovate
              image: renovate/renovate:latest
              args:
                - your-github-username/pi-cluster
              envFrom:
                - secretRef:
                    name: renovate-container-env
                - configMapRef:
                    name: renovate-configmap
          restartPolicy: Never
```

The secret contains a GitHub Personal Access Token:

```bash
kubectl create secret generic renovate-container-env \
  --from-literal=RENOVATE_TOKEN=ghp_your_token_here \
  --namespace=renovate \
  --dry-run=client -o yaml > infrastructure/controllers/base/renovate/renovate-container-env.yaml

sops --encrypt \
  --age $(grep 'age1' clusters/staging/.sops.yaml | awk '{print $NF}') \
  --encrypted-regex '^(data|stringData)$' \
  --in-place infrastructure/controllers/base/renovate/renovate-container-env.yaml
```

Add a `renovate.json` at the repo root so Renovate knows the config:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json"
}
```

---

## Putting It All Together

The Flux Kustomizations in `clusters/staging/` tie everything together:

```
clusters/staging/
  apps.yaml             -> ./apps/staging          (linkding + audiobookshelf)
  infrastructure.yaml   -> ./infrastructure/...    (renovate)
  monitoring.yaml       -> ./monitoring/...        (prometheus + grafana)
```

Each Flux Kustomization watches a path in the repo. When you push a change to any file under that path, Flux reconciles within ~1 minute.

The full workflow:

```
1. Edit a manifest locally (e.g., bump an image tag)
2. git add, commit, push
3. Flux detects the new commit
4. Flux applies the changes to the cluster
5. Pods roll out with the new config
```

No `kubectl apply`. No SSH into the Pi. Just Git.

---

## Troubleshooting

### ImagePullBackOff

**Cause 1: Wrong architecture.** The Raspberry Pi is ARM64. If you reference an `amd64`-only image, it won't work. Check Docker Hub or GHCR for multi-arch or `arm64`/`aarch64` tags. The `ghcr.io` and `lscr.io` (LinuxServer.io) registries often provide multi-arch images.

**Cause 2: DNS issues on the Pi.** The Pi can't resolve the container registry.

```bash
ssh pi-admin@k3s-node.local
ping google.com
nslookup ghcr.io
```

If DNS fails, set static DNS servers:

```bash
sudo nano /etc/dhcpcd.conf
```

Add:

```
static domain_name_servers=8.8.8.8 1.1.1.1
```

Reboot.

### Pod Stuck in Pending (PVC Issues)

k3s ships with a `local-path` StorageClass that auto-provisions PersistentVolumes. In most cases, just creating a PVC works:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-app-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

k3s handles the rest. You don't need to manually create PersistentVolume resources.

If you need explicit control over the storage location, you can create a hostPath PV:

```bash
sudo mkdir -p /mnt/data/my-app && sudo chown nobody:nogroup /mnt/data/my-app
```

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-app-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: "/mnt/data/my-app"
```

Then set `storageClassName: manual` in your PVC. Note that hostPath ties data to a specific node — fine for a single-node homelab, but won't work in a multi-node cluster.

### Connection Refused from Remote kubectl

The k3s service on the Pi is down. SSH in and check:

```bash
sudo systemctl status k3s.service
sudo journalctl -u k3s.service -f
```

Common causes:
1. Missing cgroups fix (check `/boot/firmware/cmdline.txt`)
2. Pi running out of memory — run `htop` to check

### SOPS Decryption Failures in Flux

If you see errors like `cannot get sops data key` or `no identity matched any of the recipients`:

1. Verify the `sops-age` secret exists in `flux-system`:
   ```bash
   kubectl -n flux-system get secret sops-age
   ```
2. Verify the public key in `.sops.yaml` matches the private key in the secret
3. Make sure your Flux Kustomization has the `decryption` block (see Part 4)
4. If you rotated keys, re-encrypt all secrets with the new public key:
   ```bash
   sops --encrypt --age <new-public-key> --encrypted-regex '^(data|stringData)$' \
     --in-place path/to/secret.yaml
   ```

---

## Final State

With everything deployed, here's what's running on the Pi:

```bash
$ kubectl get pods --all-namespaces
```

```
NAMESPACE        NAME                                          READY   STATUS
audiobookshelf   audiobookshelf-7b57f8d745-pf9gv               1/1     Running
linkding         cloudflared-5b8987fbcf-8hqq2                  1/1     Running
linkding         cloudflared-5b8987fbcf-dkzl5                  1/1     Running
linkding         linkding-5ff7c4c46f-g8j92                     1/1     Running
monitoring       kube-prometheus-stack-grafana-7865f4fc6d-...   3/3     Running
monitoring       prometheus-kube-prometheus-stack-prometheus-0  2/2     Running
monitoring       alertmanager-kube-prometheus-stack-...         2/2     Running
flux-system      source-controller-...                         1/1     Running
flux-system      kustomize-controller-...                      1/1     Running
flux-system      helm-controller-...                           1/1     Running
flux-system      notification-controller-...                   1/1     Running
```

Publicly accessible:
- `https://audiobooks.yourdomain.com` — Audiobookshelf
- `https://ldpi.yourdomain.com` — Linkding
- `https://grafana.yourdomain.com` — Grafana

All secrets encrypted in a public Git repo. All deployments managed by Flux. Push to Git, and the cluster updates itself.

---

## Resources

- [FluxCD Documentation](https://fluxcd.io/flux/)
- [SOPS](https://github.com/getsops/sops)
- [age encryption](https://github.com/FiloSottile/age)
- [Cloudflare Tunnels](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
- [k3s Documentation](https://docs.k3s.io/)
- [Source repo: pi-cluster](https://github.com/tiwari91/pi-cluster)
