---
layout: post
title: iSCSI as Kubernetes Persistent Storage
---

![_config.yml]({{ site.baseurl }}/images/architecture.png)

### What is Kubernetes
[Kubernetes](https://github.com/GoogleCloudPlatform/kubernetes/) is an open source Linux Container orchestrator developed by Google, Red Hat, etc. Kubernetes creates, schedules, minotors, and deletes containers across a cluster of Linux hosts. Kubernetes defines Containers as "pod", which is declared in a set of json files. 

### How Containers Persist Data in Kubernetes 
A Container running MySQL wants persistent storage so the database can survive. The persistent storage can either be on local host or ideally a shared storage that the host clusters can all access so that when the container is migrated, it can find the persisted data on the new host.

Currently Kubernetes provides [three storage volume types](https://github.com/GoogleCloudPlatform/kubernetes/blob/master/docs/volumes.md): empty_dir, host_dir, and GCE Persistent Disk. 

1. empty_dir. empty_dir is not meant to be long lasting. When the pod is deleted, the data on empty_dir is lost.
2. host_dir. host_dir presents a directory on the host to the container. Container sees this directory through a local mountpoint. Steve Watts has written an excellent [blog](http://www.emergingafrican.com/2015/02/enabling-docker-volumes-and-kubernetes.html) on provisioning NFS to containers by way of host_dir. 
3. GCE Persistent Disk. You can also use the persistent storage service available at Google Compute Engine. Kubernetes allows containers to access data residing on GCE Persisent Disk. 

### How Needs New Storage and Why iSCSI?





