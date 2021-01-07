---
title: "CIDR allocation failed for Kubernetes"
date: 2020-05-06T22:09:56+09:00
draft: false
aliases:
  - /posts/kubernetes-cidr/
tags: [Kubernetes]
categories: [Kubernetes]
---

What to do when the Pod does not start due to CIDR allocation failure.

<!--more-->

## Context

We are using [kube-aws](https://github.com/kubernetes-incubator/kube-aws) to build Kubernetes cluster.
One day, I saw that the Pod was in a lot of errors and would not start.
This has only occurred on certain nodes.

```text
NAMESPACE     NAME                         READY   STATUS              RESTARTS   AGE
kube-system   canal-node-g2vfn             2/3     CrashLoopBackOff    20         79m
kube-system   kube-proxy-8n2lj             1/1     Running             0          79m
monitor       datadog-lr74g                0/3     Init:0/2            0          79m
production    foo-9d6944774-9jlls          0/2     ContainerCreating   0          55s
...
```

## Check flannel logs

flannel had an error message like the following:

```text
I0502 02:42:02.830771       1 main.go:514] Determining IP address of default interface
I0502 02:42:02.830960       1 main.go:527] Using interface with name eth0 and address 10.12.2.236
I0502 02:42:02.830972       1 main.go:544] Defaulting external address to interface address (10.12.2.236)
I0502 02:42:02.849385       1 kube.go:126] Waiting 10m0s for node controller to sync
I0502 02:42:02.849412       1 kube.go:309] Starting kube subnet manager
I0502 02:42:03.849506       1 kube.go:133] Node controller sync successful
I0502 02:42:03.849550       1 main.go:244] Created subnet manager: Kubernetes Subnet Manager - foo-node
I0502 02:42:03.849557       1 main.go:247] Installing signal handlers
I0502 02:42:03.849682       1 main.go:386] Found network config - Backend type: vxlan
I0502 02:42:03.849719       1 vxlan.go:120] VXLAN config: VNI=1 Port=0 GBP=false DirectRouting=false
E0502 02:42:03.849902       1 main.go:289] Error registering network: failed to acquire lease: node "foo-node" pod cidr not assigned
I0502 02:42:03.849937       1 main.go:366] Stopping shutdownHandler...
```

## Check controller-manager logs

controller-manager had an error message like the following:

> CIDR allocation failed; there are no remaining CIDRs left to allocate in the accepted range

```text
E0502 03:11:40.819569       1 controller_utils.go:282] Error while processing Node Add/Delete: failed to allocate cidr: CIDR allocation failed; there are no remaining CIDRs left to allocate in the accepted range
I0502 03:11:40.820095       1 event.go:221] Event(v1.ObjectReference{Kind:"Node", Namespace:"", Name:"foo-node", UID:"2e10b7b3-8c14-11ea-a498-066c4ca4071a", APIVersion:"", ResourceVersion:"", FieldPath:""}): type: 'Normal' reason: 'CIDRNotAvailable' Node foo-node status is now: CIDRNotAvailable
```

Kubernetes assigns a range of IP addresses (CIDR blocks) to each node so that a unique IP address is specified for each Pod.
By default, Kubernetes assigns /24 CIDR blocks (256 addresses) to each node.
This error message is displayed if the Node could not be assigned a CIDR blocks.

This explanation is detailed in GKE documentation by GCP.

> 📝[Optimizing IP address allocation  |  Kubernetes Engine Documentation](https://cloud.google.com/kubernetes-engine/docs/how-to/flexible-pod-cidr)

## Check cluster CIDR blocks

Checking CIDR block for a cluster, run the following command on master node:

```bash
ps aux | grep cluster-cidr
```

Run it, you will get the following results:

```bash
root      3159  0.0  0.5 818604 89064 ?        Ssl  May02   1:53 /hyperkube controller-manager --cloud-provider=aws --cluster-name=foo-cluster --kubeconfig=/etc/kubernetes/kubeconfig/kube-controller-manager.yaml --authentication-kubeconfig=/etc/kubernetes/kubeconfig/kube-controller-manager.yaml --authorization-kubeconfig=/etc/kubernetes/kubeconfig/kube-controller-manager.yaml --leader-elect=true --root-ca-file=/etc/kubernetes/ssl/ca.pem --service-account-private-key-file=/etc/kubernetes/ssl/service-account-key.pem --use-service-account-credentials --cluster-signing-cert-file=/etc/kubernetes/ssl/worker-ca.pem --cluster-signing-key-file=/etc/kubernetes/ssl/worker-ca-key.pem --allocate-node-cidrs=true --cluster-cidr=10.2.0.0/16 --configure-cloud-routes=false --service-cluster-ip-range=10.3.0.0/16
core      6843  0.0  0.0   2820   844 pts/0    S+   14:07   0:00 grep --colour=auto cluster-cidr
```

Checking the value of the `--cluster-cidr` option:

> `--cluster-cidr=10.2.0.0/16`

Kubernetes grants each node a /24 CIDR blocks by default for use by the nodes Pods. Because this cluster assigns Pod IP addresses from a /16 CIDR blocks (`cluster-cidr`), there can be up to 256 nodes (24-16 = 8, 2^8 = 256).

The reason why the Pods didn't start this time was that the HPA scaled so many Pods that there were more than 265 nodes.

## Change CIDR blocks assigned to node

To solve this problem I made change to the CIDR blocks assigned to the nodes.
Changing the CIDR blocks from /24 to /25 reduces the number of Pods per nodes, but allowed will be 512 nodes（25-16 = 9、2^9 = 512).

To change the CIDR blocks, run controller-manager with the following options:

> `--node-cidr-mask-size=25`

Also, we limited the number of Pods that can be launched with the kubelet option to 64.
For the basis of this number 64, see the [GKE document](https://cloud.google.com/kubernetes-engine/docs/how-to/flexible-pod-cidr).

> `--max-pods=64`

When running a large Kubernetes cluster, you need to take care of the CIDR ranges.

### Refs

* [Optimizing IP address allocation  |  Kubernetes Engine Documentation](https://cloud.google.com/kubernetes-engine/docs/how-to/flexible-pod-cidr)
* [Many Kubernetes Clusters | SRCco.de](https://srcco.de/posts/many-kubernetes-clusters.html)
