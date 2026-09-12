# k8s-manifests-git

K8s config files for my Talos homelab cluster — reconciled by [Flux](https://fluxcd.io/).

## Repo layout

```
clusters/homelab/
  flux-system/           Flux's own bootstrap manifests — generated, don't hand-edit
  infrastructure.yaml    Flux Kustomization -> ./infrastructure
  apps.yaml              Flux Kustomization -> ./apps (depends on infrastructure)

apps/                                    Flux HelmReleases, one dir per app
  sources/
    helmrepository-bjws.yaml             HelmRepository: bjw-s app-template chart
    helmrepository-portainer.yaml        HelmRepository: Portainer chart
  portainer/helmrelease.yaml
  jellyfin/helmrelease.yaml
  radarr/helmrelease.yaml
  sonarr/helmrelease.yaml
  prowlarr/helmrelease.yaml
  qbittorrent/helmrelease.yaml
  flaresolverr/helmrelease.yaml
  splunk/helmrelease.yaml
  kustomization.yaml

infrastructure/
  kustomization.yaml
  monitoring/
    ingress.yaml                        MetalLB IPAddressPool + L2Advertisement
    app-ingress/grafana-ingress.yaml     Ingress for kube-prometheus-stack Grafana
  storage/
    storageclasses.yaml                 NFS StorageClasses (backed by UNAS Pro)
    mediapvc.yaml                       Shared media PVC (ReadWriteMany)
  syncthing/
    syncthing-namespace.yaml            Namespace only, work in progress
```

**This repo is the source of truth — Flux reconciles it automatically.** Push to `main` and Flux applies the change within `interval` (10m for Kustomizations, or immediately via `flux reconcile kustomization apps --with-source`). There's no more manual `kubectl apply`/`helm install` step for anything listed above.

- Apps are Helm-deployed via `HelmRelease` resources (mostly the [bjw-s `app-template`](https://github.com/bjw-s-labs/helm-charts) chart, pinned to `5.0.1`), with the exact same values that were previously passed by hand.
- Infra manifests (storage, MetalLB pool, Grafana ingress, syncthing namespace) are plain Kustomize resources under `infrastructure/`.
- **Out of scope for now / still manual Helm installs, not tracked here**: `ingress-nginx`, `metallb` (the core install, not just the pool config), `csi-driver-nfs`, `kube-prometheus-stack`, `loki`, `promtail`. Bringing these under Flux is a good next step but needs its own values-discovery pass first.
- Sops decryption (for future secrets) isn't wired up yet — the age private key isn't on the machine that bootstrapped this. Once it is: create the `sops-age` secret in `flux-system` and uncomment the `decryption` block noted in `clusters/homelab/*.yaml`.

## Dashboard

[Portainer](https://www.portainer.io/) is deployed (via Flux, `apps/portainer/helmrelease.yaml`) at `http://portainer.home.lab` for cluster visibility (pods/logs/events across all nodes) and emergency manual actions (exec, restart, edit YAML) — it is **not** the way changes get made day-to-day; that's still git commits to this repo.

## Cluster overview

- **OS**: [Talos Linux](https://www.talos.dev/)
- **Nodes**: referenced by hostname via `nodeSelector` (`g3`, `g4`, ...)
- **Ingress controller**: `nginx` (`ingressClassName: nginx`), hosts on the `*.home.lab` domain
- **LoadBalancer**: [MetalLB](https://metallb.universe.tf/) in L2 mode, address pool `192.168.0.200-192.168.0.210` (`infrastructure/monitoring/ingress.yaml`)
- **CSI storage**: [csi-driver-nfs](https://github.com/kubernetes-csi/csi-driver-nfs) (`nfs.csi.k8s.io`) backed by an NFS share on a Ubiquiti **UNAS Pro**

## Apps

| App | Namespace context | Node | Host | Config storage | Data storage |
|---|---|---|---|---|---|
| Jellyfin | media | g3 | jellyfin.home.lab | hostPath `/var/lib/media-config/jellyfin` | PVC `media-data` |
| Prowlarr | media | g3 | prowlarr.home.lab | hostPath `/var/lib/media-config/prowlarr` | - |
| Radarr | media | g3 | radarr.home.lab | hostPath `/var/lib/media-config/radarr` | PVC `media-data` |
| Sonarr | media | g3 | sonarr.home.lab | hostPath `/var/lib/media-config/sonarr` | PVC `media-data` |
| qBittorrent | media | g3 | qbittorrent.home.lab | hostPath `/var/lib/media-config/qbittorrent` | PVC `media-data` |
| FlareSolverr | media | g3 | - | - | - |
| Splunk | logging | g4 | splunk.home.lab | hostPath `/var/lib/splunk/etc` | hostPath `/var/lib/splunk/var` |

All the `*arr` apps + qBittorrent share the same `media-data` PVC (`ReadWriteMany`, `nfs-cluster-media` StorageClass) mounted at `/data`, so they can all see the same library/download paths. Config directories are still `hostPath`, pinned to node `g3` via `nodeSelector` — that's why these controllers are single-node/non-portable today.

Common conventions used across the app values files:
- `PUID`/`PGID`: `977`/`988`
- `TZ`: `Australia/Sydney`
- Images pinned to `:latest` (no digest/version pinning yet)

## Monitoring

Grafana/Prometheus/Loki (kube-prometheus-stack + Loki) are installed via Helm directly and aren't tracked in this repo beyond the Grafana `Ingress` in `infrastructure/monitoring/app-ingress/grafana-ingress.yaml`, which points at the `kube-prometheus-stack-grafana` service in the `monitoring` namespace.

## Storage

Two NFS-backed StorageClasses are defined in `infrastructure/storage/storageclasses.yaml`, both provisioned by `nfs.csi.k8s.io` against the same NAS (`192.168.0.5`):

| StorageClass | Reclaim policy | Share path | Used by |
|---|---|---|---|
| `nfs-cluster` | Delete | `.../cluster/.data` | general cluster volumes |
| `nfs-cluster-media` | Retain | `.../cluster_media/.data` | `media-data` PVC (arr stack) |

`infrastructure/storage/mediapvc.yaml` claims a `4Ti` (nominal — NFS doesn't enforce quotas) `ReadWriteMany` volume from `nfs-cluster-media`, named `media-data` in the `media` namespace.

---

## Guide: Adding UNAS Pro as Kubernetes persistent storage

The UNAS Pro (`192.168.0.5`) is already the cluster's NFS backend — `infrastructure/storage/storageclasses.yaml` and `infrastructure/storage/mediapvc.yaml` are the working example, and Jellyfin, Radarr, Sonarr, and qBittorrent already consume it. This section documents that existing setup and how to extend it.

### How storage flows, end to end

```
UNAS Pro (192.168.0.5), NFS export
  └─ StorageClass (nfs.csi.k8s.io provisioner)  →  infrastructure/storage/storageclasses.yaml
       └─ PersistentVolumeClaim (media-data)     →  infrastructure/storage/mediapvc.yaml
            └─ existingClaim in app HelmRelease    →  apps/jellyfin/helmrelease.yaml, apps/radarr/helmrelease.yaml,
                                                       apps/sonarr/helmrelease.yaml, apps/qbittorrent/helmrelease.yaml
```

**1. StorageClasses** (`infrastructure/storage/storageclasses.yaml`) — both provisioned by `nfs.csi.k8s.io` against the same UNAS Pro server, different shares:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-cluster-media
provisioner: nfs.csi.k8s.io
parameters:
  server: 192.168.0.5
  share: /volume/a9c17917-c5c5-436a-abcc-77081270b2b6/.srv/.unifi-drive/cluster_media/.data
reclaimPolicy: Retain
volumeBindingMode: Immediate
mountOptions:
  - nfsvers=3
  - nolock
```

The sibling `nfs-cluster` StorageClass in the same file points at a different share (`.../cluster/.data`) and uses `reclaimPolicy: Delete` instead — that's the general-purpose class, `nfs-cluster-media` is dedicated to media so it's set to `Retain` (deleting the PVC won't delete the files on the NAS).

**2. The PVC** (`infrastructure/storage/mediapvc.yaml`) claims a `ReadWriteMany` volume from `nfs-cluster-media`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: media-data
  namespace: media
spec:
  accessModes: ["ReadWriteMany"]
  storageClassName: nfs-cluster-media
  resources:
    requests:
      storage: 4Ti   # NFS-backed, size is nominal not enforced
```

**3. Apps consume it** by referencing the same `existingClaim` — this is what lets Jellyfin/Radarr/Sonarr/qBittorrent all see the same files for imports/hardlinks. From `apps/jellyfin/helmrelease.yaml`'s `spec.values` (identical block in `radarr/helmrelease.yaml`, `sonarr/helmrelease.yaml`, `qbittorrent/helmrelease.yaml`):

```yaml
persistence:
  media:
    enabled: true
    existingClaim: media-data
    globalMounts:
      - path: /data
```

App *config* (Sonarr's DB, Jellyfin's library metadata, etc.) is separate — it's `hostPath` under `/var/lib/media-config/<app>`, pinned to node `g3`, not on the UNAS Pro. Only the media library itself goes through NFS.

### Adding another app onto the existing media share

No new StorageClass or PVC needed — just add the same block to the new app's `HelmRelease` `spec.values`, and keep it on node `g3` with `PUID`/`PGID` `977`/`988` so ownership matches the rest of the stack:

```yaml
persistence:
  media:
    enabled: true
    existingClaim: media-data
    globalMounts:
      - path: /data
```

### Adding a separate UNAS Pro share (its own StorageClass + PVC)

For a workload that shouldn't share `media-data` (different retention needs, different dataset), follow the same pattern with a new share:

1. Export a new NFS share from the UNAS Pro (UniFi Drive) and note its path — it'll follow the same shape as the existing ones: `/volume/a9c17917-c5c5-436a-abcc-77081270b2b6/.srv/.unifi-drive/<share-name>/.data`.
2. Add a StorageClass to `infrastructure/storage/storageclasses.yaml` copying the `nfs-cluster-media` block above, swapping `metadata.name` and `parameters.share`.
3. Add a PVC copying `mediapvc.yaml`, pointed at the new StorageClass.
4. Reference it from the app via `existingClaim`, as above.

### Troubleshooting

- **PVC stuck `Pending`**: `kubectl describe pvc <name>` and check the `csi-nfs-node`/`csi-nfs-controller` pod logs (`nfs.csi.k8s.io` driver) — usually a wrong `server`/`share`, or the UNAS Pro export not allowing the node's subnet.
- **Permission denied on mount**: check the NFS export's allowed-network/squash settings on the UNAS Pro side, and confirm `PUID`/`PGID` (`977`/`988`) match what the export permits.
- **New app can't see the existing library**: confirm it's mounting `media-data` (not a fresh PVC) at `/data` — the *arr apps rely on matching paths across containers for imports/hardlinks to work.
