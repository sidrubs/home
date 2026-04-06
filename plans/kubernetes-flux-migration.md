# Plan: Migrate kubernetes/ to Flux GitOps

## Context

The `kubernetes/` directory contains plain manifests and local Helm wrapper charts manually applied with `kubectl`. The goal is to migrate to a Flux CD GitOps structure matching the pattern in `example-structure/`, adapted for a single cluster (`skuxnet`). Secrets are already SOPS-encrypted with an age key; the `.sops.yaml` at the repo root already covers the `**/secrets/**` path pattern. IHateMoney is being removed. A fresh cluster bootstrap is acceptable, with a VolSync-based data restore for Wallabag.

---

## Target Directory Structure

```
kubernetes/
├── clusters/
│   └── skuxnet/
│       ├── flux-system/          # Created by `flux bootstrap`
│       ├── ks-crds.yaml
│       ├── ks-infrastructure.yaml
│       └── ks-apps.yaml
├── infrastructure/
│   ├── sources/                  # HelmRepository definitions
│   │   ├── kustomization.yaml
│   │   ├── traefik.yaml          # https://traefik.github.io/charts
│   │   ├── longhorn.yaml         # https://charts.longhorn.io
│   │   └── volsync.yaml          # https://backube.github.io/helm-charts/
│   ├── crds/
│   │   ├── kustomization.yaml
│   │   ├── gateway-api.yaml      # Applies upstream gateway-api CRD bundle from URL
│   │   └── volumesnapshot-crds.yaml
│   ├── traefik/
│   │   ├── kustomization.yaml
│   │   ├── namespace.yaml
│   │   ├── helmrelease.yaml      # Replaces Chart.yaml + values.yaml wrapper
│   │   ├── gateway.yaml          # From traefik-gateway.yaml
│   │   └── secrets/
│   │       └── traefik-tls-secret.yaml  # From traefik-tls-secrets.yaml (already SOPS encrypted)
│   ├── longhorn/
│   │   ├── kustomization.yaml
│   │   ├── namespace.yaml
│   │   ├── helmrelease.yaml      # Replaces Chart.yaml + values.yaml wrapper (un-nest from longhorn:)
│   │   ├── encrypted-storage-class.yaml
│   │   ├── snapshot-class.yaml
│   │   └── secrets/
│   │       └── longhorn-crypto-secret.yaml  # Already SOPS encrypted
│   ├── volsync/
│   │   ├── kustomization.yaml
│   │   └── helmrelease.yaml      # Replaces Chart.yaml wrapper
│   ├── tailscale/
│   │   ├── kustomization.yaml
│   │   ├── namespace.yaml
│   │   ├── rbac.yaml
│   │   ├── deployment-dns.yaml
│   │   ├── deployment-reverse-proxy.yaml
│   │   └── secrets/
│   │       ├── tailscale-auth.yaml          # Already SOPS encrypted
│   │       └── tailscale-auth-dns.yaml      # Already SOPS encrypted
│   ├── adguard/
│   │   ├── kustomization.yaml
│   │   ├── namespace.yaml
│   │   ├── deployment.yaml
│   │   ├── service-web.yaml
│   │   ├── service-dns.yaml
│   │   ├── pvc-conf.yaml
│   │   ├── pvc-work.yaml
│   │   └── route.yaml
│   └── kustomization.yaml        # Composite: sources + crds + all components
└── apps/
    ├── namespace.yaml             # apps namespace
    ├── wallabag/
    │   ├── kustomization.yaml
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   ├── pvc-data.yaml
    │   ├── pvc-images.yaml
    │   ├── route.yaml
    │   ├── volsync-backup-data.yaml     # ReplicationSource (ongoing backup)
    │   ├── volsync-backup-images.yaml   # ReplicationSource (ongoing backup)
    │   ├── volsync-restore-data.yaml    # ReplicationDestination (manual trigger, bootstrap only)
    │   ├── volsync-restore-images.yaml  # ReplicationDestination (manual trigger, bootstrap only)
    │   └── secrets/
    │       ├── wallabag-secret.yaml          # Already SOPS encrypted
    │       ├── volsync-data-secret.yaml      # Already SOPS encrypted
    │       └── volsync-images-secret.yaml    # Already SOPS encrypted
    ├── jellyfin/
    │   ├── kustomization.yaml
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   ├── pvc-config.yaml
    │   ├── pvc-cache.yaml
    │   ├── pvc-media.yaml
    │   └── route.yaml
    └── kustomization.yaml         # Composite: wallabag + jellyfin
```

---

## Key Implementation Details

### Flux Kustomization Files (`clusters/skuxnet/`)

**`ks-crds.yaml`** — CRD layer, no prune (protect CRDs from accidental deletion):
```yaml
spec:
  interval: 1h
  prune: false
  path: ./kubernetes/infrastructure/crds
```

**`ks-infrastructure.yaml`** — Infrastructure, depends on CRDs, with SOPS decryption:
```yaml
spec:
  interval: 10m
  prune: true
  path: ./kubernetes/infrastructure
  dependsOn: [crds]
  decryption:
    provider: sops
    secretRef:
      name: sops-age   # age private key stored in flux-system namespace
```

**`ks-apps.yaml`** — Apps, depends on infrastructure:
```yaml
spec:
  interval: 5m
  prune: true
  path: ./kubernetes/apps
  dependsOn: [infrastructure]
  decryption:
    provider: sops
    secretRef:
      name: sops-age
```

### HelmRelease Migration

The three Helm wrapper charts (traefik, longhorn, volsync) are replaced with direct `HelmRelease` CRDs:

- **Traefik**: Remove `traefik:` nesting from values. Pin to `38.0.1`.
- **Longhorn**: Remove `longhorn:` nesting from values. Pin to `1.10.1`.
- **VolSync**: Simple HelmRelease, pin to `0.14.0`. Target namespace `volsync-system`.

All HelmRepositories go in `infrastructure/sources/` in the `flux-system` namespace.

The composite `infrastructure/kustomization.yaml` includes `sources/` and each component directory. `crds/` is managed by the dedicated `ks-crds` Flux Kustomization, so it is excluded from the infrastructure kustomization.

### VolSync Restore for Wallabag (Bootstrap Only)

`volsync-restore-data.yaml` and `volsync-restore-images.yaml` are `ReplicationDestination` resources with `trigger: manual`. They reference the same secrets as the backup sources. They are included in git but only triggered manually during bootstrap.

Restore procedure during bootstrap:
```bash
# Trigger the restore
kubectl annotate replicationdestination wallabag-restore-data \
  -n apps volsync.backube/trigger-immediate=true
kubectl annotate replicationdestination wallabag-restore-images \
  -n apps volsync.backube/trigger-immediate=true

# Wait for completion
kubectl wait replicationdestination wallabag-restore-data \
  -n apps --for=condition=Synchronizing=False --timeout=10m
```

After restore, the PVCs `wallabag-claim-data` and `wallabag-claim-images` are populated and the Wallabag deployment can start.

### Files to Delete

- `infrastructure/traefik/Chart.yaml`, `Chart.lock`, `charts/traefik-38.0.1.tgz`, `values.yaml`
- `infrastructure/longhorn/Chart.yaml`, `values.yaml`
- `infrastructure/volsync/Chart.yaml`
- `apps/ihatemoney/` (entire directory)
- `apps/README.md`

### Files Being Carried Over (Content Unchanged)

- All SOPS-encrypted secrets (already use the right age key)
- `infrastructure/longhorn/encrypted-storage/secrets/longhorn-encrypted-storage-secrets.yaml`
- `infrastructure/longhorn/snapshot-functionality/longhorn-snapshot-class.yaml`
- `infrastructure/longhorn/encrypted-storage/longhorn-encrypted-storage-class.yaml`
- `infrastructure/adguard/` manifests
- `infrastructure/tailscale/` manifests
- `apps/wallabag/` manifests (minus `volsync-backup-*.yaml` which get renamed/moved)
- `apps/jellyfin/` manifests

---

## Bootstrap Procedure (Fresh Cluster)

1. **Pre-bootstrap**: Store the age private key in the cluster:
   ```bash
   kubectl create namespace flux-system
   kubectl create secret generic sops-age \
     -n flux-system \
     --from-file=age.agekey=<path-to-age-private-key>
   ```

2. **Flux Bootstrap**:
   ```bash
   flux bootstrap github \
     --owner=<github-username> \
     --repository=home \
     --branch=main \
     --path=clusters/skuxnet \
     --personal
   ```
   This creates `clusters/skuxnet/flux-system/` and commits it to the repo.

3. **Flux reconciles CRDs → Infrastructure** (automated via `dependsOn`):
   - CRDs applied first (gateway-api, volumesnapshot)
   - Then: Longhorn, VolSync, Traefik, Tailscale, AdGuard

4. **Restore Wallabag data** (manual step, before apps reconcile):
   ```bash
   kubectl annotate replicationdestination wallabag-restore-data \
     -n apps volsync.backube/trigger-immediate=true
   kubectl annotate replicationdestination wallabag-restore-images \
     -n apps volsync.backube/trigger-immediate=true
   # Wait for restore to complete before apps Kustomization is healthy
   ```

5. **Apps deploy** (automated once restore PVCs are ready):
   - Wallabag uses restored PVCs
   - Jellyfin gets fresh PVCs

---

## File Docstrings

Every new file created (including all YAML manifests) gets a short comment block at the top explaining what it does. For YAML files this is a `#` comment. Examples:

```yaml
# HelmRelease for Traefik ingress controller.
# Pulls from the traefik HelmRepository in flux-system and deploys to the traefik namespace.
```

```yaml
# Flux Kustomization entry point for the infrastructure layer.
# Depends on ks-crds and reconciles every 10 minutes with SOPS decryption enabled.
```

Existing files that are carried over unchanged are not modified.

---

## Final Stage: README.md

After all manifests are in place, create `kubernetes/README.md` covering:

1. **Cluster overview** — what runs here (Traefik, Longhorn, VolSync, Tailscale, AdGuard, Wallabag, Jellyfin), the network topology (Tailscale → Traefik → apps, AdGuard DNS for `*.skuxnet`), and storage strategy (Longhorn encrypted PVCs, VolSync restic backups for Wallabag). A mermaid diagram would be nice.

2. **Prerequisites** — tools needed locally (`flux`, `kubectl`, `age`, `sops`), age key location, and GitHub personal access token for bootstrap.

3. **Fresh bootstrap steps** — ordered, numbered:
   1. Create `flux-system` namespace and `sops-age` secret from age private key
   2. Run `flux bootstrap github ...`
   3. Wait for CRDs and infrastructure to reconcile
   4. Trigger VolSync restore for Wallabag (both PVCs), wait for completion
   5. Apps reconcile automatically

4. **Ongoing operations** — how to check reconciliation status (`flux get kustomizations`, `flux get helmreleases -A`), how to force a reconcile (`flux reconcile`), and how VolSync backups are scheduled (Wallabag: weekly Sundays 3am).

5. **Secret management** — how SOPS works here (age key, `.sops.yaml` path rules), how to encrypt a new secret (`sops -e -i`), and that the age private key must never be committed.

---

## Verification

- `flux get kustomizations` — all should show `Applied revision`
- `flux get helmreleases -A` — traefik, longhorn, volsync should be `Ready`
- `kubectl get pods -A` — all pods running
- `curl -k https://wallabag.skuxnet` — Wallabag accessible with data intact
- `curl -k https://jellyfin.skuxnet` — Jellyfin accessible
- `curl -k https://adguard.skuxnet` — AdGuard accessible
- `kubectl get replicationsource -n apps` — backup sources healthy
