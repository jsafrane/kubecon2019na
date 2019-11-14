
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

TODO: add agenda / list of issues covered in the talk

---

template: inverse
# Data lost during migration

---

# Data lost during migration: What?
1. User moves PV an PVC objects from "testing" to "production" clusters.
    ```shell
    $ kubectl get pv -o yaml > pvs.yaml
    $ kubectl get pvc -o yaml > pvcs.yaml
    
    $ kubectl apply -f pvs.yaml
    $ kubectl apply -f pvcs.yaml
    ```

2. **Kubernetes deletes PV and the volume in storage backend.**

---

# Data lost during migration: Why?

* PV has `ReclaimPolicy: Delete`.
  * *"Delete the volume when this PV is not needed any longer."*
  * *"Any longer"* = PVC does not exist.
--
* PVs were restored with `pv.spec.claimRef.UID`, i.e. as fully bound!
  * PVC did not exist (yet).
  * -> `ReclaimPolicy` was executed.

---

# Data lost during migration: How?
## It's not a bug, it's a feature!

* Do regular backups!
* Use dedicated tools for migration, such as Ark / Velero.
  * *How to Backup and Restore Your Kubernetes Cluster - Annette Clewett & Dylan Murray, Tuesday 4:25pm.*
* Consider using `ReclaimPolicy: Retain`.
  * Perhaps with a custom controller / operator that deletes the volumes after review, backup and / or grace period.

---

# Data lost during migration: Lessons learned

* Better documentation!

---

template: inverse
# CVE-2017-1002101
# *Subpath volume mount handling allows arbitrary file access in host filesystem*

---

# CVE-2017-1002101: What?

*"Subpath volume mount handling allows arbitrary file access in host filesystem"*

A pod can get access to full host filesystem, including:
  * Container runtime socket.
  * Any Secrets present on the node.
  * Any Pod volume present on the node.
  * ...

KubeCon NA 2018: [How Symlinks Pwned Kubernetes (And How We Fixed It) - Michelle Au, Google & Jan Šafránek, Red Hat](https://events19.linuxfoundation.org/events/kubecon-cloudnativecon-north-america-2018/schedule/).

---

# CVE-2017-1002101: Why?

Symlinks created *in a pod* were evaluated *outside of the pod*.

* `/mnt/foo` -> `/` has a very different target depending on who is looking.

---

# CVE-2017-1002101: How?

* Resolve symlinks in a *safe* way.
* https://github.com/kubernetes/kubernetes/issues/60813
* KubeCon NA 2018: [How Symlinks Pwned Kubernetes (And How We Fixed It) - Michelle Au, Google & Jan Šafránek, Red Hat](https://events19.linuxfoundation.org/events/kubecon-cloudnativecon-north-america-2018/schedule/).

---

# CVE-2017-1002101: Lessons learned

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
# Data on PV wiped after kubelet restart: What?

* Kubelet is offline and a running pod is deleted in the API server.
* Newly (re)started kubelet deletes all data on a volume that the pod used.

---

# Data on PV wiped after kubelet restart: Why?

* Newly (re)started kubelet does not see the pod in API server.
  * kubelet did not unmount the volume.
  * Orphan directory scan did not check mounts and deleted across filesystems. 

---

# Data on PV wiped after kubelet restart: How?

* Never delete orphan directories across filesystem boundary.
* Introduce *reconstruction* - scan `/var/lib/kubelet/pods` on kubelet start and reconstruct caches.
  * Still fixing bugs there :-).

---

# Data on PV wiped after kubelet restart: Lessons learned

* Verify all `os.RemoveAll` in Kubernetes.
* Introduce `[Distuptive]` tests for kubelet restart.

---

# Data on PV wiped after kubelet restart again: What?

* A directory on the root disk used as local volume wiped out.

---

# Data on PV wiped after kubelet restart again: Why?

* Root disk used as a local volume does not introduce filesystem boundary.
* The local volume was used with `SubPath` feature.

---

# Data on PV wiped after kubelet restart again: Lessons learned

* Introduce `[Distuptive]` tests for kubelet restart with `SubPath`.

---

class: center, middle
# Not fixable issues

---

# `PersistentVolumeClaim` naming

<center><img src="marvin.jpg" width="25%"></center>

--

* `Pod` is not `CPUAndMemoryClaim`.
* `Service` is not `LoadBalancerClaim`.
* `Volume` **is** `PersistentVolumeClaim` ?

---

# `PersistentVolumeClaim` naming

* `PersistentVolumeClaim` -> `PersistentVolume`.
* `PersistentVolume` -> some implementation detail.
* "Fixed" in `VolumeSnapshot` & `VolumeSnapshotContent`.

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
