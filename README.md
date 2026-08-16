# k8s-manifests-git

K8s config files for my Talos homelab cluster.

## Repo layout

```
apps/                                   Helm values for app workloads (bjw-s app-template style)
  flaresolverr-helm.yaml
  jellyfin-values.yaml
  prowlarr-helm.yaml
  qbittorent-helm.yaml
  radarr-helm.yaml
  sonarr-helm.yaml
  splunk-helm.yaml

infrastructure/
  monitoring/
    ingress.yaml                        MetalLB IPAddressPool + L2Advertisement
    app-ingress/grafana-ingress.yaml     Ingress for kube-prometheus-stack Grafana
  storage/
    storageclasses.yaml                 NFS StorageClasses (backed by UNAS Pro)
    mediapvc.yaml                       Shared media PVC (ReadWriteMany)
  syncthing/
    syncthing-namespace.yaml            Namespace only, work in progress
```

This repo is not wired to Flux/Argo — there's no `Kustomization`/`HelmRelease` reconciliation here. Manifests are applied manually (`kubectl apply -f ...`) and Helm values files are passed to `helm install/upgrade -f ...` against their respective charts (mostly the [bjw-s `app-template`](https://github.com/bjw-s-labs/helm-charts) chart, judging by the `controllers.main.pod/containers` shape).

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
            └─ existingClaim in app Helm values   →  apps/jellyfin-values.yaml, radarr-helm.yaml,
                                                       sonarr-helm.yaml, qbittorent-helm.yaml
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

**3. Apps consume it** by referencing the same `existingClaim` — this is what lets Jellyfin/Radarr/Sonarr/qBittorrent all see the same files for imports/hardlinks. From `apps/jellyfin-values.yaml` (identical block in `radarr-helm.yaml`, `sonarr-helm.yaml`, `qbittorent-helm.yaml`):

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

No new StorageClass or PVC needed — just add the same block to the new app's Helm values, and keep it on node `g3` with `PUID`/`PGID` `977`/`988` so ownership matches the rest of the stack:

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
