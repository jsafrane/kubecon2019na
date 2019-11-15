
name: inverse
class: center, middle, inverse
layout: true

---

layout: true
class: top, left
<!-- the default layout -->

---

class: center, middle

# Storage on Kubernetes
## Learning From Failures
Hemant Kumar, Jan Šafránek<br/>
Red Hat

---

# TODO: agenda / intro

---

template: inverse
# Data lost during migration

---

# Data lost during migration
## What happened?

1. User moves PV an PVC objects from "testing" to "production" clusters.
    ```shell
    $ kubectl get pv -o yaml > pvs.yaml
    $ kubectl get pvc -o yaml > pvcs.yaml
    
    $ kubectl apply -f pvs.yaml
    $ kubectl apply -f pvcs.yaml
    ```
2. **Kubernetes deletes PV and the volume in storage backend.**
--

## Why?
* `PersistentVolumeReclaimPolicy: Delete`.
  * *"Delete the volume when this PV is not needed any longer."*
  * *"Any longer"* = PVC does not exist.

---

# Data lost during migration
## It's not a bug, it's a feature!

* Do regular backups!
* Use dedicated tools for migration, such as Ark / Velero.
  * *How to Backup and Restore Your Kubernetes Cluster - Annette Clewett & Dylan Murray, Tuesday 4:25pm.*
* Consider using `PersistentVolumeReclaimPolicy: Retain`.
  * Perhaps with a custom controller / operator that deletes the volumes after review, backup and / or grace period.

--

## Lessons learned: better documentation!

---

template: inverse
# CVE-2017-1002101
# *Subpath volume mount handling allows arbitrary file access in host filesystem*

---

# CVE-2017-1002101
## What happened?

*"Subpath volume mount handling allows arbitrary file access in host filesystem"*

A pod can get access to full host filesystem, including:
  * Container runtime socket.
  * Any Secrets present on the node.
  * Any Pod volume present on the node.
  * ...

--
## Why?
* Symlinks created *in a pod* were evaluated *outside of the pod*.

---

# CVE-2017-1002101
## How we fixed it?

KubeCon NA 2018: [How Symlinks Pwned Kubernetes (And How We Fixed It) - Michelle Au, Google & Jan Šafránek, Red Hat](https://events19.linuxfoundation.org/events/kubecon-cloudnativecon-north-america-2018/schedule/).

--
## Lessons learned

* Don't trust user.
* Containers can introduce security issues not seen before.
* Kubernetes Security Response Team (aka Product Security Committee) works and is helpful. 

---

template: inverse
# Corrupted filesystem on ReadWriteOnce volumes

---

# What happens when a block volume gets mounted on different nodes at the same time?

* For longest time Kubernetes did not enforce AccessMode of a volume and leaving this to storage provider.
  1. Lacking no control-plane based fencing mechanism, this can cause file system corruption and some very hard to track errors.
  2. In some cases where Storage Provider did not expect it, it caused instances to freeze or refusing to start on reboot.
* We implemented control-plane based enforcing of AccessMode for attachable volume types.

---

# So problem solved?

* Not really - not all volume types are attachable. 
* Volume types which are attachable:
  - AWS EBS
  - GCE PD
  - vSphere disks
* Volume types which are not attachable:
  - iSCSI
  - Ceph-RBD
* A CSI volume type is considered attachable if it implements `ControllerPublishVolume` RPC call.

---

# Solution for non-attachable volume types

* Implement a dummy `Attach` and `Detach` interface which is basically a NO-OP for `iSCSI` and `Ceph-RBD`.
* Recommendation for CSI driver:
  - If your driver exposes block volume types and has no `ControllerPublishVolume` RPC call, do not disable
    attach/detach from `CSIDriver` object.

---

template: inverse
# Data on `PersistentVolume` wiped after kubelet restart

---

# Data on PV wiped after kubelet restart
## What happened?

* Kubelet is offline and a running pod is deleted in the API server.
* Newly (re)started kubelet deletes all data on a volume that the pod used.

--

## Why?

* Newly (re)started kubelet does not see the pod in API server.
  * kubelet did not unmount the volume.
  * Orphan directory scan removed all files in presumably empty pod directory. 

---

# Data on PV wiped after kubelet restart
## How we fixed it?

* Verify all `os.RemoveAll` in Kubernetes.
  * Never delete orphan directories across filesystem boundary.
* Introduce *reconstruction* - scan `/var/lib/kubelet/pods` on kubelet start and reconstruct caches.
  * Still fixing bugs there :-).

--

## Lessons learned

* Introduced `[Distuptive]` tests for kubelet restart.

---

template: inverse
# Data on `PersistentVolume` wiped after kubelet restart again

---

# Data on PV wiped after kubelet restart *again*
## What happened?

* A directory on the root disk used as local volume wiped out.
* Same scenario as above.

--

## Why?

* Root disk used as a local volume does not introduce filesystem boundary.
* The local volume was used with `SubPath` feature.

## How we fixed it?

* Check for SubPath volumes before removing orphan directories.

--

## Lessons learned

* Introduce `[Disruptive]` tests for kubelet restart with `SubPath`.

---

template: inverse
# Volumes are recycled while they are used by pods

---

# Volumes are recycled while they are used by pods
## What happened?

* User deletes PVC while it's still used by a pod.

## Why?

* Kubernetes has no referential integrity.
* `PersistentVolumeReclaimPolicy: Recycle`: the volume is wiped!
* `PersistentVolumeReclaimPolicy: Delete`: Kubernetes tries to delete the volume. 

--

## How we fixed it?
* Using `Finalizers`.
* StorageInUseProtection admission plugin and controller.

---

class: center, middle
# Not fixable issues

---

# `PersistentVolumeClaim` naming

* `Pod` is not `CPUAndMemoryClaim`.
--
* `Service` is not `LoadBalancerClaim`.
--
* `Volume` **is** `PersistentVolumeClaim` ???

--

"Fixed" in `VolumeSnapshot` & `VolumeSnapshotContent`.

---

# `AccessModes`

* `ReadWriteOne`, `ReadWriteMany`, `ReadOnlyMany`
* Enforced only lightly in A/D controller!
  * Multiple pods can still use single `ReadWriteOne` volume on the same node.
* Fix would break behavior.

---

class: center, middle
# Open Issues

---

# Recursive `chown`

```shell
$ kubectl explain pod.spec.securityContext.fsGroup

FIELD:    fsGroup <integer>

DESCRIPTION:
     A special supplemental group that applies to all containers in a pod. Some
     volume types allow the Kubelet to change the ownership of that volume to be
     owned by the pod [...]
```

* kubelet does recursive `chown` to set ownership of **all** files on the volume.
  * Slow on large volumes.
* Design in progress.
  * Take shortcuts? Some files may have wrong owner.
  * Make `chown` optional? Requires API change.
  * Use overlay FS? Requires the overlay installed on nodes.

---

# Shutdown nodes

TODO(hekumar): fill details

---

# Volume reconstruction

* kubelet reconstructs caches from `/var/lib/kubelet/pods`.
  * TODO: add example?
  * Mostly works and is actively supported!
* There should be a real database / checkpointing.
  * Current kubelet checkpoints do not include PVCs / PVs.
