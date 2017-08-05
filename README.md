# NOTE: This document is a work in progress

# lizardfs-flexvolume

This Kubernetes (>=1.7.0) FlexVolume driver aims to provide an implementation for provisioning LizardFS volumes to a Kubernetes POD resource.

It supports the following additional features over previous incantations of this driver:
1. Supports mfspassword via spec options and also k8s secrets
1. Supports mfs* options via spec (_TODO: use ConfigMap_)

# Installation

## Ready-2-run hyperkube image with pre-installed LizardFS clients and FlexVolume driver
https://quay.io/TerraTech/hyperkube

This image contains the lizardfs-client binaries and fq~lizardfs FlexVolume driver.
It is also custom patched to disable the SELinux relabeling requirement.
Do not use this image if you are using FlexVolume driver for anything non-fuse based, e.g. LVM mounts

## Manual install
**WARNING**: This driver will not work in the stock hyperkube image!
LizardFS is fuse based and the pod will never become ready as the mount cannot be SELinux relabeled and will fail: https://github.com/lizardfs/lizardfs/issues/581  
Until this is fixed, use the above hyperkube image

* If /usr is a **ReadWrite** partition, you can use the default k8s FlexVolume driver location.  Just make sure that your kubelet.service mounts this directory into your container.

```bash
cp -p lizardfs /usr/libexec/kubernetes/kubelet-plugins/volume/exec/fq~lizardfs/
chmod +x /usr/libexec/kubernetes/kubelet-plugins/volume/exec/fq~lizardfs/lizardfs
```
*If /usr is **ReadOnly**, like CoreOS, more work is needed
```bash
mkdir -p /etc/kubernetes/kubelet-plugins/volume/exec/fq~lizardfs
cp -p lizardfs /etc/kubernetes/kubelet-plugins/volume/exec/fq~lizardfs/lizardfs
chmod +x /etc/kubernetes/kubelet-plugins/volume/exec/fq~lizardfs/lizardfs
systemctl edit --full kubelet.service

** Add the following kubelet option: --volume-plugin-dir=/etc/kubernetes/kubelet-plugins/volume/exec/fq~lizardfs/
** This path was chosen because hyperkube already mounts /etc/kubernetes into the container
```

Then proceed to restart the Kubelet service

Obviously you'd need to have a functional running LizardFS cluster reachable by the Kubernetes minions.

This drivers uses the jq binary on your minions in order to parse JSON options sent by the controller. 
Ensure jq is installed on your minion's system
https://github.com/coreos/bugs/issues/1706 

```bash
apt-get install -y jq
```

If you'd want to use the provided YML example files in order to test the feature :
```bash
kubectl create -f examples/provision-volumes.yml
```

Three PersistentVolumes should be created plus one PVC, bound to one of your PVs (could be any of those 3) :
```bash
kubectl get pv
NAME           CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS      CLAIM                                   STORAGECLASS   REASON    AGE
mfs-volume     100Gi      RWX           Retain          Bound       default/test-1                                                   3h
mfs-volume-1   100Gi      RWX           Retain          Available                                                                    3h
mfs-volume-2   100Gi      RWX           Retain          Available                                                                    3h
```

Then proceed with a basic POD creation using this PVC __test-1__ :

```bash 
kubectl create -f examples/deployment-using-lizardfs.yml
```

# CREDITS
https://github.com/lowet84 for the first implementation as seen on https://github.com/lizardfs/lizardfs/issues/486
https://github.com/M0nsieurChat (updates for Kubernetes >=1.6.3 compatibility)
