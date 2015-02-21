---
layout: post
title: NAS Server in the Cloud?
---
### What does it mean, really?

Cloud evangelists forecast the future of data center are in the Cloud, yet I am not convinced that leads to the demise of 
storage servers. I believe storage vendors will find a new home for their products: **Cloud**. 

NetApp, a storage vendor who sells NAS boxes transforms itself to an AWS server image provider. The server image just provides the same function as the NAS boxes do. 

### Why people still need storage servers, even in Cloud?

#### Compatibility

Cloud storage like S3 and Swift are object store, while most enterprise applications still work with file and block based storage. 
Shifting data center into the Cloud must first deal with this API level differences.

#### Portability

Cloud storage technologies may vary from one vendor to another, with absence of industry wide protocols. This poses a migration risk for
Cloud hoppers. In constrast, NFS/CIFS/iSCSI/FC protocols are found in all storage servers, as long as in-cloud storage server exists,
such migration risk is much deminished.

#### Value

It is undeniable that storage vendors like EMC, NetApp, HP, Hitachi, and IBM pride themselves on technologies (and patents) that Cloud storage don't yet have. Their value proposition won't evaporate any time soon.

### How does it look like?

My rough component level comparision is illustrated here.

![_config.yml]({{ site.baseurl }}/images/cloud-server.png)
