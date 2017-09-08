# SIG-Windows

## Introduction

Sig-windows is a community effort to bring windows server container workloads to Kubernetes.  The current goal is not to have a windows node be the master, but to have it run the Kubelet and be able to run pods with windows server containers as part of a larger cluster with a Linux master node.  There are a couple of streams of work  with varying levels of activity currently in progress to achieve this.

Some of the remaining work is:

* Networking: Multi-container pod networking
* Metrics
* Secrets
* Config Maps
* Volumes

**Note: This document is a work in progress and is intended to be updated with community input regularly as progress is made.  If you have any additions or changes you would like to add to this document, please submit a PR.**

## Networking

Networking was one of the first major hurdles.  Many of the current technologies used for cluster networking are not cross platform, and do not run on Windows.  One solution that does fulfill this requirement is [Open vSwitch](http://openvswitch.org/).

### Open vSwitch

https://github.com/openvswitch/ovs

Open vSwitch (OVS/OVN) is an [open-source](https://en.wikipedia.org/wiki/Open-source) implementation of a distributed virtual [multilayer switch](https://en.wikipedia.org/wiki/Multilayer_switch). 
The main purpose of Open vSwitch is to provide a [switching](https://en.wikipedia.org/wiki/Network_switch) stack for [hardware virtualization](https://en.wikipedia.org/wiki/Hardware_virtualization) environments, 
while supporting multiple protocols and standards used in [computer networks](https://en.wikipedia.org/wiki/Computer_network).  Open Virtual Network (OVN) is a part of OVS that was added to provide higher level abstractions over OVS, building logical networks with logical switches, logical routers, logical datapaths, and logical ports.  Documentation can be found at [ovn-architecture](http://openvswitch.org/support/dist-docs/ovn-architecture.7.html)

The basic architecture is that there is a master node, which contains the ovn northbound db & ovn southbound db, which contain the desired logical network, and it's corresponding datapath flows respectively.  Each of the other nodes will run an ovn-controller which will will receive configuration changes in the southbound db and be able to update the physical flows accordingly.

### ovn-kubernetes

https://github.com/openvswitch/ovn-kubernetes

The ovn-kubernetes project is a wrapper around Open vSwitch which creates the necessary logical network for a heterogeneous Kubernetes cluster.  It is currently written in Python, though there is a port to Go which has been started.  Each node that is setup has an ovn-k8s-overlay init command run (master-init|minion-init|gateway-init) with some of the networking configuration passed in which sets up the networking for that node by making calls to through to ovn and ovs.

#### ovn-kubernetes watchers/services

##### Master node - ovn-k8s-watcher
This watcher runs on the master node, and watches kubernetes events.  When a traditional linux based pod is deployed, it is responsible for creating logical ports required for pod networking.  Each pod will get a logical port added to it's worker node logical switch.  The watcher is a service that runs at startup via systemd, and logs to /var/log/openvswitch/ovn-k8s-watcher.log .

##### Gateway node - ovn-k8s-gateway-helper
Not explicitly a watcher, but another service that runs at startup to de-multiplex traffic on the gateway when a single interface is used for both management traffic (ssh) and the cluster's North-South traffic.

##### Windows Worker node - ovn-k8s service
This service runs in the background on windows, and intercepts docker calls for container creation and creates the logical networking necessary for the pod.


## Metrics

Stats and metrics are another area where Windows and Linux diverge.  Usually, stats are populated from CAdvisor, which reads them directly from cgroups on Linux. For Windows, CAdvisor is not supported and there are no plans to support it on Windows in the future https://github.com/google/cadvisor/issues/1394.

There is some work to provide status via the cadvisor shim on windows using a combination of native windows perf monitoring subsystem (PerfCounters to get node stats) and calling out to the docker engine REST API to get container stats.  The proposal can be found at https://github.com/kubernetes/kubernetes/issues/49398 .

There is also ongoing work to allow kubelet to consume stats via the CRI.  Once this is finalized, the work in the cadvisor shim is intended to be ported to the CRI implementation.


## Secrets

A Secret is an object that contains a small amount of sensitive data such as a password, a token, or a key. Such information might otherwise be put in a Pod specification or in an image; putting it in a Secret object allows for more control over how it is used, and reduces the risk of accidental exposure.

Users can create secrets, and the system also creates some secrets.

To use a secret, a pod needs to reference the secret. A secret can be used with a pod in three ways: as files in a volume mounted on one or more of its containers, environment variables within the container, or used by kubelet when pulling images for the pod.

You can create a secret a number of ways, here we will use a literal value

`kubectl create secret generic dbpass --from-literal=dbpass=SomeSecretPassword`


### Secrets as environment variables

To add the secret as an environment variable to a Windows container, the process is exactly the same as a Linux container.  The following will map the value of the secret created above to the environment variable DB_PASSWORD of the test-container created.

secret-pod.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-var
spec:
  containers:
  - image: redis:3.0-nanoserver
    name: test-container
    env: 
      - name: DB_PASSWORD
        valueFrom:
          secretKeyRef:
            name: dbpass
            key: dbpass
  nodeSelector:
    beta.kubernetes.io/os: windows
```


Investigation into other methods of using secrets is still ongoing.

## Config Maps

ConfigMaps allow you to decouple configuration artifacts from image content to keep containerized applications portable.

### Literal Value config map

```kubectl create configmap special-config --from-literal=special.how=very```


### Acccessing config as environment variable

The following yaml file will take the special.how value from the config map and create an environment variable named HOW_SPECIAL in the container with the value `very`.

configmap-env.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd-hp3
spec:
  containers:
  - image: redis:3.0-nanoserver
    name: test-container
    env: 
      - name: HOW_SPECIAL
        valueFrom:
          configMapKeyRef:
            name: special-config
            key: special.how
  nodeSelector:
    beta.kubernetes.io/os: windows
```

### Accessing config as volume

This appears to be blocked by an issue where symlinks created outside of docker containers do not work within the container on windows.

Investigation into this feature is ongoing.
## Volumes

### HostPath Volumes

A hostPath volume mounts a file or directory from the host node’s filesystem into your pod. This is not something that most Pods will need, but it offers a powerful escape hatch for some applications.

For example, some uses for a hostPath are:

* running a container that needs access to Docker internals; use a hostPath of /var/lib/docker
* running cAdvisor in a container; use a hostPath of /dev/cgroups

Watch out when using this type of volume, because:

* pods with identical configuration (such as created from a podTemplate) may behave differently on different nodes due to different files on the nodes
* when Kubernetes adds resource-aware scheduling, as is planned, it will not be able to account for resources used by a hostPath
    the directories created on the underlying hosts are only writable by root.
* You either need to run your process as root in a privileged container or modify the file permissions on the host to be able to write to a hostPath volume

hostpath.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd-hp4
spec:
  containers:
  - image: redis:3.0-nanoserver
    name: test-container
    volumeMounts:
    - mountPath: C:/test-pd
      name: test-volume
  volumes:
  - name: test-volume
    hostPath:
      # directory location on host
      path: C:/data
  nodeSelector:
    beta.kubernetes.io/os: windows
```


### Empty

An emptyDir volume is first created when a Pod is assigned to a Node, and exists as long as that Pod is running on that node. As the name says, it is initially empty. Containers in the pod can all read and write the same files in the emptyDir volume, though that volume can be mounted at the same or different paths in each container. When a Pod is removed from a node for any reason, the data in the emptyDir is deleted forever. NOTE: a container crashing does NOT remove a pod from a node, so the data in an emptyDir volume is safe across container crashes.

Some uses for an emptyDir are:

* scratch space, such as for a disk-based merge sort
* checkpointing a long computation for recovery from crashes
* holding files that a content-manager container fetches while a webserver container serves the data

By default, emptyDir volumes are stored on whatever medium is backing the machine - that might be disk or SSD or network storage, depending on your environment. However, you can set the emptyDir.medium field to "Memory" to tell Kubernetes to mount a tmpfs (RAM-backed filesystem) for you instead. While tmpfs is very fast, be aware that unlike disks, tmpfs is cleared on machine reboot and any files you write will count against your container’s memory limit.

empty.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: redis:3.0-nanoserver
    name: test-container
    volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir: {}
  nodeSelector:
    beta.kubernetes.io/os: windows
```

### Secret

A secret volume is used to pass sensitive information, such as passwords, to pods. You can store secrets in the Kubernetes API and mount them as files for use by pods without coupling to Kubernetes directly. secret volumes are backed by tmpfs (a RAM-backed filesystem) so they are never written to non-volatile storage.

Important: You must create a secret in the Kubernetes API before you can use it

Secrets are described in more detail [here](https://kubernetes.io/docs/user-guide/secrets).

As of right now this does not appear to work with Windows due to https://github.com/moby/moby/issues/28401

### Persistent Volume Claim 

A persistentVolumeClaim volume is used to mount a [PersistentVolume](https://kubernetes.io/docs/user-guide/persistent-volumes) into a pod. PersistentVolumes are a way for users to “claim” durable storage (such as a GCE PersistentDisk or an iSCSI volume) without knowing the details of the particular cloud environment.

See the [PersistentVolumes example](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) for more details.


First, create the persistent volume.  This uses gcePersistentDisk for mounting GCE persistent disk.  https://kubernetes.io/docs/concepts/storage/volumes/#gcepersistentdisk

Then add that disk to your windows node, format and mount that disk (example here mounts it to D:\).  The following will add the disk to k8s as a persistent volume.

windows-pv.yaml
```yaml
apiVersion: "v1"
kind: "PersistentVolume"
metadata:
  name: "gce-windows-pv"
spec:
  capacity:
    storage: "1Gi"
  accessModes:
    - "ReadWriteOnce"
  hostPath:
    path: "D:/"
```

The next yaml file will create the persistent volume claim that a pod can use to mount the volume.

windows-pv-claim.yaml
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: windows-pv-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

The following creates a pod and mounts the persistent volume to C:\test-pv in the redis container.

redis-pvc.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd-hp5
spec:
  containers:
  - image: redis:3.0-nanoserver
    name: test-container2
    volumeMounts:
    - mountPath: C:/test-pd
      name: pv1
  volumes:
    - name: pv1
      persistentVolumeClaim:
        claimName: task-pv-claim
  nodeSelector:
    beta.kubernetes.io/os: windows
```

Persistent Volume Claims allow a pod to use a persistent volume without knowing the underlying mechanics of the volume (is it a local/network/gce disk).  The mechanics of Persistent Volumes and Persistent Volume Claims appear to work fine with windows containers.




# Demo

The demo at [kubernetes-ovn-heterogeneous-cluster](https://github.com/apprenda/kubernetes-ovn-heterogeneous-cluster) walks through the manual provisioning and setup of an cluster on Google Cloud Platform (though the steps to setup on different cloud provider would be very similar). 

The cluster consists of:

* 1 Linux Kubernetes master node, which also doubles as the ovn/ovs master
* 1 Linux Kubernetes worker node, which will run Linux based pods
* 1 Windows Kubernetes worker node, which will run Windows Server Container pods
* 1 Linux Gateway node which acts as the gateway between the logical network provided by OVS/OVN, and the internet


# Get Involved

For general developer questions: Use the #sig-windows slack channel

Sign up for a free account at: http://slack.kubernetes.io 

Go to the #sig-windows channel and ask for help. e.g. I'm having problems deploying pods to my Windows node  

Bi-Weekly Standup (Every other Tuesday, 16:30 UTC) - http://www.zoom.us/my/sigwindows

# Links

Sig-Windows community page - https://github.com/kubernetes/community/tree/master/sig-windows 

ovn-kubernetes project page - https://github.com/openvswitch/ovn-kubernetes

Open vSwitch project page - https://github.com/openvswitch/ovs

Heterogeneous Cluster Demo - https://github.com/apprenda/kubernetes-ovn-heterogeneous-cluster