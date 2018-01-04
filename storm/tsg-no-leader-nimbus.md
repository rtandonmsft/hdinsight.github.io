---
title: TSG: Could not find leader nimbus from seed hosts [headnodehost] | Microsoft Docs
description: Use the Storm FAQ for answers to common questions on Storm on Azure HDInsight platform.
keywords: Azure HDInsight, Storm, FAQ, troubleshooting guide, log4j
services: Azure HDInsight
documentationcenter: na
author: rtandon
manager: ''
editor: ''

ms.assetid: d7c14702-948b-41db-bb45-7185fde247f1
ms.service: multiple
ms.workload: na
ms.tgt_pltfrm: na
ms.devlang: na
ms.topic: article
ms.date: 08/02/2017
ms.author: rtandon
---
## TSG: Could not find leader nimbus from seed hosts [headnodehost] on HDInsight with version 3.3 for Windows

### Issue:
**Applies to: HDInsight Storm for Windows**
**Does NOT apply to: HDInsight Storm for Linux**

HDInsight version 3.3 for Windows had a version of Storm which did not have HA support for Nimbus. 
The Nimbus HA support was later added into Storm but is NOT available on HDInsight Storm cluster for Windows.

High Availiblity of services in HDInsight Windows clusters is acheived by service called failover-controller which has called failover-master and failover-slave. 

When one of the headnode goes down, services on the other head node are started by slave component and the host entry for **headnodehost** is updated.

**To acheive HA support in Storm, a custom component was deployed that copies topologies artifacts from the active Nimbus to the other Nimbus node every 5 minutes.**
The storm.local.dir is configured as ```c:\hdistorm``` on HDInsight clusters. And the topology artifacts are located at:
1. Nimbus Node (headnode): ```c:\hdistorm\nimbus\stormdist```
2. Supervisor Node (workernode): ```c:\hdistorm\supervisor\stormdist```

In very rare cases, a situation may arise that the headnode which is the active Nimbus may lose topology artifacts. This can cause Nimbus to fail while starting up resulting in the error below.

```
java.lang.RuntimeException: Could not find leader nimbus from seed hosts [headnodehost]. Did you specify a valid list of nimbus hosts for config nimbus.seeds
```

Cases:
1. When a nimbus node undergoes a re-image shortly after a new topology is submitted (and before the scheduled copy task could complete), both nimbus nodes may end up not having the topology code locally.
2. After a headnode failover, both the headnodes may have gone through a Windows Azure Hosted Service re-image operation resulting in wipe of the resource drive ```C:\```, with neither of the Nimbus nodes having topology artifacts.


### Symptoms:

1. You are able to open the **STORM UI** using the cluster URL but the **STORM UI** is displaying this message instead of your running topologies.
  i. **NOTE**: Please be assured that the topologies are still running on Supervisor (worker) nodes and it's only Nimbus service that manages the topology is impacted.

2. Login into any of the headnodes, check the active headnode by browsing the **Hadoop Service Availability** webpage shortcut on the desktop. The verification and fix should be applied on that active headnode.

3. Check the Nimbus logs found under the local storm folder on the active headnode at ```C:\apps\dist\%STORM_HOME%\logs\nimbus.log``` for errors during start-up. The symptoms will be as follows:
```
2017-07-29 17:40:52.302 b.s.zookeeper [INFO] headnode0.********.a5.internal.chinacloudapp.cn gained leadership, checking if it has all the topology code locally.
2017-07-29 17:40:52.309 b.s.zookeeper [INFO] active-topology-ids [TOPOLOGY-NAME-1498805833,TOPOLOGY-2-NAME-1500962748] local-topology-ids [TOPOLOGY-3-NAME-1498553914,TOPOLOGY-4-NAME-1498614910] diff-topology [TOPOLOGY-3-NAME-1498805833,TOPOLOGY-NAME-4-1500962748]
2017-07-29 17:40:52.309 b.s.zookeeper [INFO] code for all active topologies not available locally, giving up leadership.
```

or

```
2017-12-30 22:29:59.318 b.s.zookeeper [INFO] headnode0..********..d9.internal.cloudapp.net gained leadership, checking if it has all the topology code locally.
2017-12-30 22:29:59.320 b.s.zookeeper [INFO] active-topology-ids [TOPOLOGY-NAME-10-1513919768,TOPOLOGY-NAME-2-1492675419] local-topology-ids [] diff-topology [TOPOLOGY-NAME-10-1513919768,TOPOLOGY-NAME-2-1492675419]
2017-12-30 22:29:59.321 b.s.zookeeper [INFO] code for all active topologies not available locally, giving up leadership.
2017-12-30 22:30:16.369 b.s.d.nimbus [INFO] [req 1] Access from:  principal: op:getClusterInfo
2017-12-30 22:30:16.433 o.a.t.s.AbstractNonblockingServer$FrameBuffer [ERROR] Unexpected throwable while invoking!
java.lang.RuntimeException: No nimbus leader participant host found, have you started your nimbus hosts?
```

3. Additionally, verify that the Topology information found in Zookeeper via the zookeeper command line tool.
```
[zk: localhost:2181(CONNECTED) 5] get /storm/storms
[TOPOLOGY-NAME-1498805833, TOPOLOGY-2-NAME-1500962748]
```

### Solution:

1. To fix this situation, one needs to manually copy the topology artifacts for the all **diff-topology** listed in error above from the ```c:\hdistorm\supervisor\stormdist``` 
directory in one of the supervisor (worker) nodes to both the nimbus (head) nodes at ```c:\hdistorm\nimbus\stormdist``` through remote desktop or any other means.
  

2. Once the copy is done, restart the Storm nimbus service through ```services.msc```. The issue should now be fixed as seen from Nimbus logs:
```
2017-08-01 02:10:26.040 b.s.zookeeper [INFO] headnode0.cmi-cdp-storm-cd-prd.a5.internal.chinacloudapp.cn gained leadership, checking if it has all the topology code locally.
2017-08-01 02:10:26.043 b.s.zookeeper [INFO] active-topology-ids [TOPOLOGY-NAME-1498805833,TOPOLOGY-NAME-1500962748] local-topology-ids [TOPOLOGY-NAME-1498805833,TOPOLOGY-NAME-1498553914,TOPOLOGY-NAME-6-1498614910,TOPOLOGY-NAME-1500962748] diff-topology []
2017-08-01 02:10:26.043 b.s.zookeeper [INFO] Accepting leadership, all active topology found localy.
```

3. Verify in **STORM UI** that all your topologies are showing up again.
