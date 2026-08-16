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

## Guide: Adding a UNAS Pro share as Kubernetes persistent storage

This is the pattern already used for `nfs-cluster` / `nfs-cluster-media` above — a Ubiquiti **UNAS Pro** exposing an NFS share (via UniFi Drive) that Kubernetes consumes through `csi-driver-nfs`. Use this if you're adding a *new* share/StorageClass, or setting this up on a fresh cluster.

### 1. Create and expose the share on the UNAS Pro

1. In the UniFi OS console for the UNAS Pro, go to **UniFi Drive** and create (or pick) a volume/folder to export, e.g. a `cluster` or `cluster_media` folder.
2. Enable **NFS** access for that share (UniFi Drive → share settings → Network Shares → NFS). Note the exported path shown there — it will look like:
   `/volume/<volume-uuid>/.srv/.unifi-drive/<share-name>/.data`
3. Under the NFS export settings, allow access from your cluster's node subnet (e.g. `192.168.0.0/24`), and set permissions so the CSI driver's mount UID/GID (or `nobody`/`no_root_squash`, depending on how strict you want to be) can read/write.
4. Note the UNAS Pro's LAN IP (in this repo: `192.168.0.5`) — that's the NFS `server` value.

> UNAS Pro NFS defaults to NFSv3-style behavior for these UniFi Drive exports; the existing StorageClasses here use `mountOptions: [nfsvers=3, nolock]` because of this. If you enable NFSv4 on a share, drop those options.

### 2. Install `csi-driver-nfs` on the cluster

```bash
helm repo add csi-driver-nfs https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
helm repo update
helm install csi-driver-nfs csi-driver-nfs/csi-driver-nfs \
  --namespace kube-system \
  --set kubeletDir=/var/lib/kubelet
```

Notes for Talos:
- Talos ships with in-kernel NFS client support, so no extra system extension is normally required just to *mount* NFS from pods.
- `kubeletDir` defaults to `/var/lib/kubelet` on most distros; Talos also uses this path, but double-check on your Talos version/config if the CSI node pods fail to start.
- Confirm the driver is healthy before moving on: `kubectl -n kube-system get pods -l app=csi-nfs-node`.

### 3. Define a StorageClass pointing at the UNAS Pro share

Add a new StorageClass (or reuse the pattern in `infrastructure/storage/storageclasses.yaml`):

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-<your-share-name>
provisioner: nfs.csi.k8s.io
parameters:
  server: 192.168.0.5                                   # UNAS Pro LAN IP
  share: /volume/<volume-uuid>/.srv/.unifi-drive/<share-name>/.data
reclaimPolicy: Retain     # or Delete — Retain is safer for media/important data
volumeBindingMode: Immediate
mountOptions:
  - nfsvers=3
  - nolock
```

Apply it:

```bash
kubectl apply -f infrastructure/storage/storageclasses.yaml
```

### 4. Claim a volume

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: <claim-name>
  namespace: <namespace>
spec:
  accessModes: ["ReadWriteMany"]   # NFS supports RWX, useful for shared app data
  storageClassName: nfs-<your-share-name>
  resources:
    requests:
      storage: 100Gi                # nominal — NFS doesn't enforce quotas
```

### 5. Mount it in an app

For `app-template`-based Helm values (matching the style used in `apps/*.yaml`):

```yaml
persistence:
  mydata:
    enabled: true
    existingClaim: <claim-name>
    globalMounts:
      - path: /data
```

Or as a raw Pod volume:

```yaml
volumes:
  - name: mydata
    persistentVolumeClaim:
      claimName: <claim-name>
```

### Troubleshooting

- **PVC stuck `Pending`**: check `kubectl describe pvc <name>` and the `csi-nfs-node`/`csi-nfs-controller` pod logs — usually a bad `server`/`share` path or the UNAS Pro NFS export not allowing the node's IP.
- **Mount denied / permission errors**: revisit the NFS export's allowed-network and squash settings on the UNAS Pro; container `PUID`/`PGID` need to line up with what the export permits.
- **Slow or flaky I/O**: this NAS is on the LAN, not local disk — fine for media/config, but avoid it for latency-sensitive workloads (databases, etc.) unless tuned/tested for it.
