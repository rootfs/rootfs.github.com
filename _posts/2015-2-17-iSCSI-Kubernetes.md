---
layout: post
title: iSCSI as Persistent Storage for Kubernetes and Docker Container
---
### iSCSI Storage
[iSCSI](http://en.wikipedia.org/wiki/ISCSI) has been widely adopted in data centers. It is the default implementation for [OpenStack Cinder](https://wiki.openstack.org/wiki/Cinder). Cinder defines a common block storage interface so storage vendors can supply their own plugins to present their storage products to Nova compute. As it happens, most of the vendor supplied plugins use iSCSI (https://wiki.openstack.org/wiki/CinderSupportMatrix).

### Containers: How to Persist Data?
Persisting Data inside a container can be done in two ways.
 1. A Container sets up iSCSI initiator, the iSCSI channel goes through Docker NAT to external iSCSI target. This approach doesn't require host's support and is thus portal. However, the Container is likely to suffer suboptimal performance, because Docker NAT doesn't deliver good performance, as reseachers at IBM [found](http://domino.research.ibm.com/library/cyberdig.nsf/papers/0929052195DD819C85257D2300681E7B/$File/rc25482.pdf). Since iSCSI is highly senstive to network performance, delay or jitters will cause iSCSI connection timieout and retries. This approach is thus not preferred for mission-critical services.
 2. Host initiates iSCSI connection, attaches iSCSI disk, and mounts the filesystem on the disk to a local directory, and shares the filesystem with Container. This approach doesn't need Docker NAT and is conceivably higher performing than the first approach.
 
In fact, this approach is the foundation of the iSCSI persistent storage for Kubernetes, discussed in the following.

### What is Kubernetes
[Kubernetes](https://github.com/GoogleCloudPlatform/kubernetes/) is an open source Linux Container orchestrator developed by Google, Red Hat, etc. Kubernetes creates, schedules, minotors, and deletes containers across a cluster of Linux hosts. Kubernetes defines Containers as "pod", which is declared in a set of json files. 

### How Containers Persist Data in Kubernetes 
A Container running MySQL wants persistent storage so the database can survive. The persistent storage can either be on local host or ideally a shared storage that the host clusters can all access so that when the container is migrated, it can find the persisted data on the new host.

Currently Kubernetes provides [three storage volume types](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/volumes.md): empty_dir, host_dir, and GCE Persistent Disk. 

1. empty_dir. empty_dir is not meant to be long lasting. When the pod is deleted, the data on empty_dir is lost.
2. host_dir. host_dir presents a directory on the host to the container. Container sees this directory through a local mountpoint. Steve Watts has written an excellent [blog](http://www.emergingafrican.com/2015/02/enabling-docker-volumes-and-kubernetes.html) on provisioning NFS to containers by way of host_dir. 
3. GCE Persistent Disk. You can also use the persistent storage service available at Google Compute Engine. Kubernetes allows containers to access data residing on GCE Persisent Disk. 

### iSCSI Disk: a New Persistent Storage for Kubernetes
Since on-premise enterprise data centers and OpenStack providers have already invested in iSCSI storage. When they deploy Kubernetes, it is logical that they want Containers access data living on iSCSI storage. It is thus desirable for Kubernetes to support iSCSI disk based persistent volume

### Implementation
[My Kubernetes pull request](https://github.com/GoogleCloudPlatform/kubernetes/pull/4612) provides a solution to this end. 

As seen in this  high level architecture ![_config.yml]({{ site.baseurl }}/images/architecture.png)

When *kubelete* creates the pod on the *node*(previously known as *minion*), it logins into iSCSI target, and mounts the specified disks to the container's volumes. Containers can then access the data on the persistent storage. Once the container is deleted and iSCSI disks are not used, *kubelet* logs out of the target.

A Kubernetes pod can use iSCSI disk as persistent storage for read and write. As exhibited in this pod [example](https://github.com/rootfs/kubernetes/blob/iscsi-pd-merge/examples/iscsi-pd/iscsi-pd.json), this pod declares two containers: both uses iSCSI LUNs. Container *iscsipd-ro* mounts the read-only ext4 filesystem backed by iSCSI LUN 0 to _/mnt/iscsipd_, and Container *iscsipd-ro* mounts the read-write xfs filesystem backed by iSCSI LUN 1 to _/mnt/iscsipd_. 

### How to Use it?
Here is my setup to setup Kubernetes with iSCSI persistent storage. I use Fedora 21 on Kubernetes node. 

First get my github repo

    # git clone -b iscsi-pd-merge https://github.com/rootfs/kubernetes
   
then build and install on the Kubernetes master and node.

Install iSCSI initiator on the node:

    # yum -y install iscsi-initiator-utils
   
   
then edit */etc/iscsi/initiatorname.iscsi* and */etc/iscsi/iscsid.conf* to match your iSCSI target configuration.

I mostly follow these [instructions](http://www.server-world.info/en/note?os=Fedora_21&p=iscsi&f=2) to setup iSCSI initiator and these [instructions](http://www.server-world.info/en/note?os=Fedora_21&p=iscsi) to setup iSCSI target.

Once you have installed iSCSI initiator and new Kubernetes, you can create a pod based on my [example](https://github.com/rootfs/kubernetes/blob/iscsi-pd-merge/examples/iscsi-pd/iscsi-pd.json). In the pod JSON, you need to provide *portal* (the iSCSI target's **IP** address and *port* if not the default port 3260), target's *iqn*, *lun*, and the type of the filesystem that has been created on the lun, and *readOnly* boolean. 

Once your pod is created, run it on the Kubernetes master:

    #cluster/kubectl.sh create -f your_new_pod.json

Here is my command and output:

    # cluster/kubectl.sh create -f examples/iscsi-pd/iscsi-pd.json 
    current-context: ""
    Running: cluster/../cluster/gce/../../_output/local/bin/linux/amd64/kubectl create -f examples/iscsi-pd/iscsi-pd.json
    iscsipd
    
On the Kubernetes node, I got these in mount output


```javascript
/dev/sdb on /var/lib/kubelet/plugins/kubernetes.io/iscsi-pd/iscsi/10.16.154.81:3260/iqn.2014-12.world.server:storage.target1/lun/0 type ext4 (ro,relatime,stripe=1024,data=ordered)
/dev/sdb on /var/lib/kubelet/pods/74695f6c-b86d-11e4-a4a4-d4bed9b39058/volumes/kubernetes.io~iscsi-pd/iscsipd-ro type ext4 (ro,relatime,stripe=1024,data=ordered)
/dev/sdc on /var/lib/kubelet/plugins/kubernetes.io/iscsi-pd/iscsi/10.16.154.81:3260/iqn.2014-12.world.server:storage.target1/lun/1 type xfs (rw,relatime,attr2,inode64,noquota)
/dev/sdc on /var/lib/kubelet/pods/74695f6c-b86d-11e4-a4a4-d4bed9b39058/volumes/kubernetes.io~iscsi-pd/iscsipd-rw type xfs (rw,relatime,attr2,inode64,noquota)
```
