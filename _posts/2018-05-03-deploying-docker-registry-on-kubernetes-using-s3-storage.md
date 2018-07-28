---
layout: post
title: Deploying Docker Registry on Kubernetes using S3 Storage
published: true
categories: [Kubernetes]
description: Deploying self-hosted Docker Registry on kubernetes using S3 storage.
author_github: rahulmahale
---

In today's era of containerization,
no matter what container we are using
we need an image to run the container.
Docker images are stored on container registries
like Docker hub(cloud),
Google Container Registry(GCR), AWS ECR, quay.io etc.

We can also self-host docker registry on any docker platform.
In this blog post, we will see
how to deploy docker registry
on kubernetes using storage driver S3.

#### Pre-requisite:

 *  Access to working kubernetes cluster.

 * Understanding of [Kubernetes](http://kubernetes.io/)
terms like [pods](http://kubernetes.io/docs/user-guide/pods/),
[deployments](http://kubernetes.io/docs/user-guide/deployments/),
[services](https://kubernetes.io/docs/concepts/services-networking/service/),
[configmap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)
and
[ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/).


As per docker registry [documentation](https://docs.docker.com/registry/deploying/),
We can simply start the registry using docker image `registry`.

Basic parameters when deploying production registry are:

 * Authentication
 * SSL
 * Storage

We will use **htpasswd** authentication for this post though registry image
supports **silly** and **token** based authentication as well.

Docker registry requires applications to use SSL certificate and key in the registry.
We will use kubernetes service, which terminates SSL on ELB level using annotations.

For registry storage, we can use filesystem, s3, azure, swift etc. For the complete list of options please visit [docker site](https://docs.docker.com/registry/configuration/#storagedocker) site.

We need to store the docker images pushed to the registry.
We will use S3 to store these docker images.

#### Steps for deploying registry on kubernetes.

Get the `ARN` of the SSL certificate to be used for SSL.

If you don't have SSL on AWS IAM, upload it using the following command.

```
$aws iam upload-server-certificate --server-certificate-name registry --certificate-body file://registry.crt --private-key file://key.pem

```
Get the `arn` for the certificate using the command.

```
$aws iam get-server-certificate --server-certificate-name registry  | grep Arn

```

Create S3 bucket which will be used to store docker images using s3cmd or aws s3.

```

$s3cmd mb s3://myregistry
Bucket 's3://myregistry/' created

```

Create a separate namespace, configmap, deployment and service
for registry using following templates.

{% highlight yaml %}

---
apiVersion: v1
kind: Namespace
metadata:
name: container-registry

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: auth
  namespace: container-registry
data:
  htpasswd: |
    admin:$2y$05$TpZPzI7U7cr3cipe6jrOPe0bqohiwgEerEB6E4bFLsUf7Bk.SEBRi

---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: registry
  name: registry
  namespace: container-registry
spec:
  replicas: 1
  strategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: registry
    spec:
      containers:
      - env:
        - name: REGISTRY_AUTH
          value: htpasswd
        - name: REGISTRY_AUTH_HTPASSWD_PATH
          value: /auth/htpasswd
        - name: REGISTRY_AUTH_HTPASSWD_REALM
          value: Registry Realm
        - name: REGISTRY_STORAGE
          value: s3
        - name: REGISTRY_STORAGE_S3_ACCESSKEY
          value: <your-s3-access-key>
        - name: REGISTRY_STORAGE_S3_BUCKET
          value: <your-registry-bucket>
        - name: REGISTRY_STORAGE_S3_REGION
          value: us-east-1
        - name: REGISTRY_STORAGE_S3_SECRETKEY
          value: <your-secret-s3-key>
        image: registry:2
        name: registry
        ports:
        - containerPort: 5000
        volumeMounts:
          - name: auth
            mountPath: /auth
      volumes:
        - name: auth
          configMap:
            name: auth
---
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: <your-iam-certificate-arn>
    service.beta.kubernetes.io/aws-load-balancer-instance-protocol: http
    service.beta.kubernetes.io/aws-load-balancer-ssl-ports: '443'
  labels:
    app: registry
  name: registry
  namespace: container-registry
spec:
  ports:
  - name: "443"
    port: 443
    targetPort: 5000
  selector:
    app: registry
type: LoadBalancer

{% endhighlight %}

Let's launch this manifest using `kubectl apply`.

{% highlight bash %}

kubectl apply -f registry-namespace.yml registry-configmap.yml registry-deployment.yaml registry-namespace.yml
namespace "registry" created
configmap "auth" created
deployment "registry" created
service "registry" created

{% endhighlight %}

Now that we have created registry, we should map DNS to web service ELB endpoint.
We can get the webservice ELB endpoint using the following command.

{% highlight bash %}

$kubectl -n registry get svc registry -o wide

NAME       CLUSTER-IP      EXTERNAL-IP                                                               PORT(S)         AGE       SELECTOR
registry   100.71.250.56   abcghccf8540698e8bff782799ca8h04-1234567890.us-east-2.elb.amazonaws.com   443:30494/TCP   1h       app=registry

{% endhighlight %}

We will point DNS to this ELB endpoint with domain registry.myapp.com

Once we have registry running, now it's time to push the image to a registry.

First, pull the image or build the image locally to push.

On local machine run following commands:

{% highlight bash %}

$docker pull busybox
latest: Pulling from busybox
f9ea5e501ad7: Pull complete
ac3f08b78d4e: Pull complete
Digest: sha256:da268b65d710e5ca91271f161d0ff078dc63930bbd6baac88d21b20d23b427ec
Status: Downloaded newer image for busybox:latest

{% endhighlight %}

Now login to our registry using the following commands.

{% highlight bash %}

$ sudo docker login registry.myapp.com

Username: admin

Password:

Login Succeeded

{% endhighlight %}

Now tag the image to point it to our registry using `docker tag` command

```
$ sudo docker tag busybox registry.myapp.com/my-app:latest

```
Once the image is tagged we are good to push.

Using the `docker push` command let's push the image.

{% highlight bash %}

$ sudo docker push docker.gocloudlogistics.com/my-app:latest

The push refers to a repository [registry.myapp.com/my-app]

05732a3f47b5: Pushed
30de36c4bd15: Pushed
5237590c0d08: Pushed
latest: digest: sha256:f112e608b2639b21498bd4dbca9076d378cc216a80d52287f7f0f6ea6ad739ab size: 205

{% endhighlight %}

We are successfully able to push image to registry running on kunbernetes and stored on S3.
Let's verify if it exists on S3.

Navigate to our s3 bucket and we can see the docker registry repository
`busybox` has been created.

```
$ s3cmd ls s3://myregistry/docker/registry/v2/repositories/
DIR   s3://myregistry/docker/registry/v2/repositories/busybox/

```
All our image related files are stored on S3.

In this way, we self-host container registry on kubernetes backed by s3 storage.
