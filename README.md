# Deploying Audiobookshelf with GitOps

This document outlines the steps to deploy Audiobookshelf using a GitOps workflow with FluxCD, including persistence and public access via Cloudflare Tunnel. The directory structure follows the established pattern for this cluster.

---

## Prerequisites

-   A Kubernetes cluster managed by FluxCD.
-   A configured Git repository connected to Flux.
-   A persistent storage solution (e.g., Longhorn, Ceph) with a default or named StorageClass.
-   A Cloudflare account and the `cloudflared` CLI tool.
-   SOPS configured with an AGE key for secret encryption within Flux.
-   The directory structure matching the project's layout (e.g., `clusters/pi-cluster/apps/staging/`, `clusters/pi-cluster/apps/base/`, `clusters/pi-cluster/infrastructure/base/`).

---

## Stage 1: Basic Deployment

This stage sets up the core application components without persistence.

1.  **Manifests Location:** `clusters/pi-cluster/apps/staging/audiobookshelf/base/`
2.  **Create Files:**
    * `namespace.yaml`: Defines the `audiobookshelf` namespace.
    * `configmap.yaml`: Creates `audiobookshelf-config` with `ABS_PORT="3005"` and `TZ`.
    * `deployment.yaml`: Defines the Audiobookshelf deployment, referencing the ConfigMap and setting the container port to `3005`. Initially has empty `volumeMounts` and `volumes`.
    * `service.yaml`: Creates a `ClusterIP` service named `audiobookshelf` exposing port `3005`.
    * `kustomization.yaml`: Lists all the above resources.
3.  **Flux Kustomization Definition:**
    * Create `clusters/pi-cluster/apps/base/audiobookshelf/kustomization.yaml`.
    * This file defines the `Kustomization` resource for Flux (`metadata.name: audiobookshelf-app`).
    * Sets `spec.path` to `./clusters/pi-cluster/apps/staging/audiobookshelf/base`.
    * Sets `spec.targetNamespace` to `audiobookshelf`.
    * References your GitRepository source (`spec.sourceRef`).
4.  **Update Cluster App Aggregator:**
    * Edit `clusters/pi-cluster/apps/base/kustomization.yaml`.
    * Add `./audiobookshelf` to the `resources` list. This tells the main app Kustomization (e.g., `cluster-apps`) to apply the Flux Kustomization defined in the previous step.
5.  **Commit & Push:** Commit all new/modified files and push to your Git repository.
6.  **Verify:**
    * `flux get kustomizations audiobookshelf-app -n flux-system`
    * `kubectl get pods -n audiobookshelf`
    * `kubectl port-forward svc/audiobookshelf -n audiobookshelf 3005:3005`
    * Access `http://localhost:3005`, complete initial setup.

---

## Stage 2: Adding Persistence

This stage configures persistent storage for configuration and media.

1.  **Manifests Location:** `clusters/pi-cluster/apps/staging/audiobookshelf/base/`
2.  **Create `pvc.yaml`:**
    * Define `PersistentVolumeClaim` resources for:
        * `audiobookshelf-config-pvc` (e.g., 1Gi)
        * `audiobookshelf-metadata-pvc` (e.g., 2Gi)
        * `audiobookshelf-audiobooks-pvc` (e.g., 50Gi, adjust size)
        * `audiobookshelf-podcasts-pvc` (e.g., 20Gi, adjust size)
    * **Important:** Ensure `spec.storageClassName` matches an available StorageClass in your cluster or is omitted to use the default.
3.  **Update `deployment.yaml`:**
    * Add `volumeMounts` to the container for `/config`, `/metadata`, `/audiobooks`, and `/podcasts`.
    * Add `volumes` to the pod spec, linking each name to the corresponding `persistentVolumeClaim.claimName`.
4.  **Update `kustomization.yaml` (in `base`):**
    * Add `pvc.yaml` to the `resources` list.
5.  **Commit & Push:** Commit changes and push.
6.  **Verify:**
    * Wait for Flux reconciliation (deployment will restart).
    * `kubectl get pvc -n audiobookshelf` (Check status is `Bound`).
    * Port-forward, upload/configure data, delete the pod (`kubectl delete pod -l app.kubernetes.io/name=audiobookshelf -n audiobookshelf`), wait for the new pod, port-forward again, and verify data persists.

---

## Stage 3: Non-Root User & Public Access

This stage enhances security by running as a non-root user and exposes the application publicly via Cloudflare Tunnel.

### Cloudflare Tunnel Setup

1.  **Create Tunnel:**
    ```bash
    cloudflared tunnel create audiobookshelf
    ```
    * Note the Tunnel ID and locate the generated `TUNNEL-ID.json` credentials file (usually in `~/.cloudflared/`).
2.  **Create Secret Manifest:**
    * Generate a base64-encoded secret manifest using `kubectl create secret generic --dry-run`:
        ```bash
        kubectl create secret generic tunnel-credentials \
          --from-file=credentials.json=/path/to/your/TUNNEL-ID.json \
          --namespace=cloudflare # Or the namespace cloudflared will run in
          --dry-run=client -o yaml > clusters/pi-cluster/infrastructure/base/cloudflare-secret.yaml
        ```
    * Ensure the secret `metadata.name` (`tunnel-credentials`) matches the `volumes.secret.secretName` in your `cloudflared` Deployment.
    * Ensure the key inside `data` is `credentials.json` (due to `--from-file=credentials.json=...`).
3.  **Encrypt Secret:**
    * Encrypt `cloudflare-secret.yaml` using SOPS and your AGE key:
        ```bash
        # Ensure AGE_PUBLIC_KEY env var is set or use --age flag directly
        sops --encrypt --age $(cat clusters/pi-cluster/.agekey | grep -o 'age1[a-z0-9]*') \
          --encrypted-regex '^(data|stringData)$' \
          --in-place clusters/pi-cluster/infrastructure/base/cloudflare-secret.yaml
        ```
    * Rename to `cloudflare-secret.sops.yaml`.
4.  **Add Secret to Infrastructure:**
    * Add `cloudflare-secret.sops.yaml` to the `resources` in `clusters/pi-cluster/infrastructure/base/kustomization.yaml`.
    * Verify the main infrastructure Flux Kustomization (e.g., `infrastructure.yaml`) enables SOPS `decryption` using your `sops-age` secret.
5.  **Configure DNS:** Point your desired public hostname (e.g., `audiobookshelf.your-domain.com`) to your tunnel using a CNAME record in Cloudflare DNS or the `cloudflared` command:
    ```bash
    cloudflared tunnel route dns audiobookshelf audiobookshelf.your-domain.com # Use your actual tunnel name/ID and desired hostname
    ```

### Cloudflared Deployment

* Ensure your `cloudflared` Deployment manifest (e.g., `clusters/pi-cluster/infrastructure/base/cloudflared.yaml`) exists and is included in the infrastructure Kustomization.
* The Deployment should mount the `tunnel-credentials` secret into `/etc/cloudflared/creds`.
* The associated `cloudflared` ConfigMap (`config.yaml` data) should reference `credentials-file: /etc/cloudflared/creds/credentials.json`.
* Crucially, the `ingress` rules in the `config.yaml` data must route traffic correctly:
    ```yaml
    ingress:
      - hostname: audiobookshelf.your-domain.com # Your public hostname
        service: [http://audiobookshelf.audiobookshelf:3005](http://audiobookshelf.audiobookshelf:3005) # K8s Service FQDN (service.namespace:port)
      # Optional catch-all rule
      - service: http_status:404
    ```

### Audiobookshelf Updates

1.  **Run As Non-Root:**
    * Modify `clusters/pi-cluster/apps/staging/audiobookshelf/base/deployment.yaml`.
    * Add a `securityContext` block at the `spec.template.spec` level (`runAsUser: 1000`, `runAsGroup: 1000`, `fsGroup: 1000`, `runAsNonRoot: true`). Adjust UID/GID if needed.
    * Add a `securityContext` block inside the container spec (`allowPrivilegeEscalation: false`, `capabilities: { drop: ["ALL"] }`).
2.  **Add Ingress (Optional but Recommended):**
    * Create `clusters/pi-cluster/apps/staging/audiobookshelf/base/ingress.yaml`.
    * Set the `host` to your public domain (e.g., `audiobookshelf.your-domain.com`).
    * Set the `backend.service.name` to `audiobookshelf` and `port.number` to `3005`.
    * **Crucially**, set the appropriate `annotations` (like `kubernetes.io/ingress.class: "none"` or specific Cloudflare annotations) based *exactly* on how your `cloudflared` deployment discovers services. If your `cloudflared` doesn't use Ingress objects, you might skip this file.
    * Add `ingress.yaml` to the `resources` in `clusters/pi-cluster/apps/staging/audiobookshelf/base/kustomization.yaml`.
3.  **Add Dependency:**
    * Edit the **Flux Kustomization** definition file `clusters/pi-cluster/apps/base/audiobookshelf/kustomization.yaml`.
    * Add a `dependsOn` section to ensure infrastructure (including the secret) is applied first:
        ```yaml
        spec:
          # ... other spec fields ...
          dependsOn:
            - name: cluster-infrastructure # Use the name of your infrastructure Flux Kustomization
        ```

### Commit & Push

* Add all modified/new files (`deployment.yaml`, secret, ingress, kustomizations).
* Commit and push.

### Verification

1.  Monitor Flux: `flux get kustomizations --all-namespaces`. Check `cluster-infrastructure` and `audiobookshelf-app`.
2.  Check Pods: `kubectl get pods -n audiobookshelf` and `kubectl get pods -n cloudflare` (or wherever cloudflared runs).
3.  Check Secret: `kubectl get secret tunnel-credentials -n cloudflare -o yaml` (verify it exists, data is encrypted).
4.  Check Logs: `kubectl logs -l app=cloudflared -n cloudflare` (look for tunnel connection messages).
5.  Test DNS: `nslookup audiobookshelf.your-domain.com`.
6.  Access: Browse to `https://audiobookshelf.your-domain.com`.

---