---
layout: post
title: Scheduling pods on nodes in Kubernetes using labels
published: true
categories: [Kubernetes]
description: Blog post on how to assign pods to specific nodes on Kubernetes
author_github: rahulmahale
---

This post assumes that you have basic understanding of
[Kubernetes](http://kubernetes.io/)
terms like
[pods](http://kubernetes.io/docs/user-guide/pods/),
[deployments](http://kubernetes.io/docs/user-guide/deployments/)
and
[nodes](https://kubernetes.io/docs/concepts/architecture/nodes/).


A Kubernetes cluster can have many nodes.
Each node in turn can run multiple pods.
By default Kubernetes manages which pod
will run on which node and this is something
we do not need to worry about it.

However sometimes we want to ensure that
certain pods do not run on the same node.
For example we have an application called _wheel_.
We have both staging and production version of this app
and we want to ensure that production pod and staging pod
are not on the same host.

To ensure that certain pods do not run on the same host
we can use
**nodeSelector**
constraint in **PodSpec** to schedule pods on nodes.

### Kubernetes cluster

We will use [kops](https://github.com/kubernetes/kops/) to provision
cluster.
We can check the health of cluster using
`kops validate-cluster`.

```
$ kops validate cluster
Using cluster from kubectl context: test-k8s.nodes-staging.com

Validating cluster test-k8s.nodes-staging.com

INSTANCE GROUPS
NAME              ROLE   MACHINETYPE MIN MAX SUBNETS
master-us-east-1a Master m4.large    1   1 us-east-1a
master-us-east-1b Master m4.large    1   1 us-east-1b
master-us-east-1c Master m4.large    1   1 us-east-1c
nodes-wheel-stg   Node   m4.large    2   5 us-east-1a,us-east-1b
nodes-wheel-prd   Node   m4.large    2   5 us-east-1a,us-east-1b

NODE STATUS
           NAME                ROLE   READY
ip-192-10-110-59.ec2.internal  master True
ip-192-10-120-103.ec2.internal node   True
ip-192-10-42-9.ec2.internal    master True
ip-192-10-73-191.ec2.internal  master True
ip-192-10-82-66.ec2.internal   node   True
ip-192-10-72-68.ec2.internal   node   True
ip-192-10-182-70.ec2.internal  node   True

Your cluster test-k8s.nodes-staging.com is ready
```

Here we can see that there are two instance groups for nodes: _nodes-wheel-stg_  and _nodes-wheel-prd_.

_nodes-wheel-stg_ might have application pods like _pod-wheel-stg-sidekiq_, _pod-wheel-stg-unicorn_ and _pod-wheel-stg-redis_.
Smilarly
_nodes-wheel-prd_ might have application pods like _pod-wheel-prd-sidekiq_, _pod-wheel-prd-unicorn_ and _pod-wheel-prd-redis_.


As we can see the **Max number of nodes** for instance group _nodes-wheel-stg_ and _nodes-wheel-prd_ is 5. It means if
new nodes are created in future then based on the instance group the newly created nodes will automatically
be labelled and no manual work is required.

### Labelling a Node

We will use [kubernetes labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)
to label a node.
To add a label we need to edit instance group using kops.

```
$ kops edit ig nodes-wheel-stg
```

This will open up instance group configuration file,
we will add following label in instance group spec.

```
nodeLabels:
   type: wheel-stg
 ```

Complete `ig` configuration looks like this.

```
apiVersion: kops/v1alpha2
kind: InstanceGroup
metadata:
  creationTimestamp: 2017-10-12T06:24:53Z
  labels:
    kops.k8s.io/cluster: k8s.nodes-staging.com
  name: nodes-wheel-stg
spec:
  image: kope.io/k8s-1.7-debian-jessie-amd64-hvm-ebs-2017-07-28
  machineType: m4.large
  maxSize: 5
  minSize: 2
  nodeLabels:
    type: wheel-stg
  role: Node
  subnets:
  - us-east-1a
  - us-east-1b
  - us-east-1c
```

Similarly, we can label for instance group
_nodes-wheel-prod_ with label _type wheel-prod_.

After making the changes update cluster using
`kops rolling update cluster --yes --force`.
This will update the cluster with specified labels.

New nodes added in future will have
labels based on respective `instance groups`.

Once nodes are labeled
we can verify using `kubectl describe node`.

```
$ kubectl describe node ip-192-10-82-66.ec2.internal
Name:               ip-192-10-82-66.ec2.internal
Roles:              node
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/instance-type=m4.large
                    beta.kubernetes.io/os=linux
                    failure-domain.beta.kubernetes.io/region=us-east-1
                    failure-domain.beta.kubernetes.io/zone=us-east-1a
                    kubernetes.io/hostname=ip-192-10-82-66.ec2.internal
                    kubernetes.io/role=node
                    type=wheel-stg
  ```

In this way we have our node labeled using kops.

### Labelling nodes using kubectl

We can also label node using `kubectl`.

```
$ kubectl label node ip-192-20-44-136.ec2.internal type=wheel-stg
```

After labeling a node, we will add `nodeSelector` field
to our `PodSpec` in deployment template.

We will add the following block in deployment manifest.

```
nodeSelector:
  type: wheel-stg
```

We can add this configuration in original deployment manifest.

```
apiVersion: v1
kind: Deployment
metadata:
  name: test-staging-node
  labels:
    app: test-staging
  namespace: test
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: test-staging
    spec:
      containers:
      - image: <your-repo>/<your-image-name>:latest
        name: test-staging
        imagePullPolicy: Always
        - name: REDIS_HOST
          value: test-staging-redis
        - name: APP_ENV
          value: staging
        - name: CLIENT
          value: test
        ports:
        - containerPort: 80
      nodeSelector:
        type: wheel-stg
      imagePullSecrets:
        - name: registrykey
```

Let's launch this deployment and check where the pod is scheduled.

```
$ kubectl apply -f test-deployment.yml
deployment "test-staging-node" created
```

We can verify that our pod is running on node `type=wheel-stg`.

```
kubectl describe pod test-staging-2751555626-9sd4m
Name:           test-staging-2751555626-9sd4m
Namespace:      default
Node:           ip-192-10-82-66.ec2.internal/192.10.82.66
...
...
Conditions:
  Type           Status
  Initialized    True
  Ready          True
  PodScheduled   True
QoS Class:       Burstable
Node-Selectors:  type=wheel-stg
Tolerations:     node.alpha.kubernetes.io/notReady:NoExecute for 300s
                 node.alpha.kubernetes.io/unreachable:NoExecute for 300s
Events:          <none>
```

Similarly we run _nodes-wheel-prod_ pods on nodes labeled with `type: wheel-prod`.

Please note that when we specify `nodeSelector` and no node matches label
then pods are in `pending` state as they
dont find node with matching label.

In this way we schedule our pods to
run on specific nodes for certain use-cases.
