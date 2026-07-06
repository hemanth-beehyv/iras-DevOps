# NFS server setup — shared storage for the k3s cluster

A single NFS server exporting one share (`irasNFS`) over the same
`192.168.0.0/24` subnet used by the [k3s cluster](K8s-setup.md), for use
as shared/persistent storage across nodes.

## Server plan

| Item          | Value                        |
|---------------|-------------------------------|
| NFS host      | digitnfs01 (192.168.0.x)      |
| Export path   | `/mnt/nfs01/irasNFS`          |
| Allowed clients | `192.168.0.0/24`            |
| Filesystem    | on root LV (no dedicated disk) — see `df -h` below |

    $ df -h
    Filesystem                         Size  Used Avail Use% Mounted on
    tmpfs                              794M  1.3M  793M   1% /run
    /dev/mapper/ubuntu--vg-ubuntu--lv 1006G  7.4G  957G   1% /
    tmpfs                              3.9G     0  3.9G   0% /dev/shm
    tmpfs                              5.0M     0  5.0M   0% /run/lock
    /dev/sda2                          2.0G  133M  1.7G   8% /boot
    tmpfs                              794M  4.0K  794M   1% /run/user/1000

`/mnt/nfs01/irasNFS` lives on the root LV (`ubuntu-vg/ubuntu-lv`, ~1TB) —
there is no separate disk/partition mounted for it. If this share is
expected to grow, plan for a dedicated volume instead.

## Step 1 — Install the NFS server package

    sudo apt update
    sudo apt install nfs-kernel-server

Verify the service is up:

    sudo systemctl status nfs-server

## Step 2 — Create the export directory

    sudo mkdir -p /mnt/nfs01/irasNFS
    sudo chown -R nobody:nogroup /mnt/nfs01/irasNFS
    sudo chmod 777 /mnt/nfs01/irasNFS

`chmod 777` + `chown nobody:nogroup` makes the share writable by any
client/uid — simplest option for a lab/internal network, not appropriate
if the network isn't trusted (see Security notes below).

## Step 3 — Configure the export

    sudo nano /etc/exports

Add:

    /mnt/nfs01/irasNFS 192.168.0.0/24(rw,sync,fsid=1,no_subtree_check,insecure,no_root_squash,anonuid=1001,anongid=1001)

| Option              | Meaning                                                              |
|---------------------|-----------------------------------------------------------------------|
| `rw`                | clients can read and write                                            |
| `sync`              | server acknowledges writes only after they hit disk                   |
| `fsid=1`            | stable filesystem id for this export (required for some NFS setups)   |
| `no_subtree_check`  | skip subtree checking — faster, fine for exporting a whole filesystem/dir |
| `insecure`          | allow client requests from ports > 1024 (needed by some NFS clients)  |
| `no_root_squash`    | root on a client keeps root privileges on the share (see security note) |
| `anonuid`/`anongid` | maps anonymous requests to uid/gid 1001 instead of `nobody`/`nogroup` |

Apply the export without restarting the service:

    sudo exportfs -a

Verify:

    sudo exportfs -v
    showmount -e localhost

## Client mount (reference)

On each client (e.g. k3s nodes) that needs the share:

    sudo apt install nfs-common
    sudo mkdir -p /mnt/irasNFS
    sudo mount -t nfs 192.168.0.x:/mnt/nfs01/irasNFS /mnt/irasNFS

To persist across reboots, add to `/etc/fstab`:

    192.168.0.x:/mnt/nfs01/irasNFS  /mnt/irasNFS  nfs  defaults  0  0

## Step 4 — Install the NFS CSI driver in Kubernetes

Once the NFS server/export above is up, install the [csi-driver-nfs](../../deploy-as-code/helm/charts/backbone-services/csi-driver-nfs)
chart so the cluster can dynamically provision PVs against the `irasNFS` share.

Chart path: `deploy-as-code/helm/charts/backbone-services/csi-driver-nfs`

    cd deploy-as-code/helm/charts/backbone-services/csi-driver-nfs
    helm install csi-driver-nfs . \
      --namespace kube-system \
      --set storageClass.create=true \
      --set storageClass.name=nfs-csi \
      --set-string storageClass.annotations."storageclass\.kubernetes\.io/is-default-class"="true" \
      --set storageClass.parameters.server=192.168.0.87 \
      --set storageClass.parameters.share=/mnt/nfs01/irasNFS \
      --set storageClass.reclaimPolicy=Retain

This creates the `nfs-csi` StorageClass (set as the cluster default) pointing at
the `192.168.0.87:/mnt/nfs01/irasNFS` export, with `Retain` as the reclaim policy
so PVs aren't deleted when their PVC is removed.

## Troubleshooting

| Symptom                              | Check                                                        |
|---------------------------------------|---------------------------------------------------------------|
| `mount.nfs: access denied by server`  | Client IP not in the allowed subnet in `/etc/exports`; re-run `sudo exportfs -a` after edits |
| `showmount -e` shows nothing          | `sudo systemctl status nfs-server`; confirm `/etc/exports` syntax |
| Client can mount but not write        | Check `chmod`/`chown` on the export dir, and `no_root_squash`/`anonuid` mapping |
| Changes to `/etc/exports` not applied | `sudo exportfs -ra` (re-export all) or `sudo systemctl restart nfs-server` |

## Security notes

- `no_root_squash` + `insecure` + `chmod 777` is a permissive combination
  suitable for a trusted internal lab network — do **not** use this
  combination on a share reachable from an untrusted network.
- Restrict the CIDR in `/etc/exports` to only the hosts that need access
  (currently the full `/24`).
