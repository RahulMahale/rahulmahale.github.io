---
layout: post
title: Using Kubernetes Persistent volume to store persistent data
published: true
categories: [Kubernetes]
description: This post summarizes how to use the Kubernetes persistent volume in the deployment
author_github: rahulmahale
---


In one of our projects
we are running Rails application on
[Kubernetes](http://kubernetes.io/)
cluster.
It is proven tool for managing and deploying docker containers in production.

In kubernetes containers are managed using
[deployments](http://kubernetes.io/docs/user-guide/deployments/)
and they are termed as
[pods](http://kubernetes.io/docs/user-guide/pods/).
`deployment` holds the specification of pods.
It is responsible to run the pod with specified resources.
When `pod` is restarted or `deployment` is deleted then data is lost on pod.
We need to retain data out of pods lifecycle
when the  `pod` or `deployment` is destroyed.

We use docker-compose during development mode.
In docker-compose  linking between host directory and container directory works out of the box.
We wanted similar mechanism with kuberentes to link volumes.
In kubernetes we have various types of
[volumes](https://kubernetes.io/docs/user-guide/volumes/#types-of-volumes1)
to use.
We chose
[persistent volume](http://kubernetes.io/docs/user-guide/persistent-volumes/)
with
[AWS EBS](https://aws.amazon.com/ebs/)
storage.
We used persistent volume claim as per the need of application.

As per the
[Persistent Volume's definition](http://kubernetes.io/docs/user-guide/persistent-volumes/) (PV)
Cluster administrators must first create storage in order for Kubernetes to mount it.

Our Kubernetes cluster is hosted on AWS. We created AWS EBS
volumes which can be used to create persistent volume.

Let's create a sample volume using aws cli and
try to use it in the deployment.

{% highlight bash %}

aws ec2 create-volume --availability-zone us-east-1a --size 20 --volume-type gp2

{% endhighlight %}

This will create a volume in `us-east-1a` region.
We need to note `VolumeId` once the volume is created.

{% highlight bash %}

$ aws ec2 create-volume --availability-zone us-east-1a --size 20 --volume-type gp2
{
    "AvailabilityZone": "us-east-1a",
    "Encrypted": false,
    "VolumeType": "gp2",
    "VolumeId": "vol-123456we7890ilk12",
    "State": "creating",
    "Iops": 100,
    "SnapshotId": "",
    "CreateTime": "2017-01-04T03:53:00.298Z",
    "Size": 20
}


{% endhighlight %}

Now let's create a persistent volume template `test-pv` to create volume using this EBS storage.

{% highlight yaml %}

kind: PersistentVolume
apiVersion: v1
metadata:
  name: test-pv
  labels:
    type: amazonEBS
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  awsElasticBlockStore:
    volumeID: <your-volume-id>
    fsType: ext4

{% endhighlight %}

Once we had template to create persistent volume,
we used [kubectl](http://kubernetes.io/docs/user-guide/kubectl/) to launch it.
Kubectl is command line tool to interact with Kubernetes cluster.

{% highlight bash %}

$ kubectl create -f  test-pv.yml
persistentvolume "test-pv" created

{% endhighlight %}

Once persistent volume is created you can check using following command.

{% highlight bash %}

$ kubectl get pv
NAME       CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS      CLAIM               REASON    AGE
test-pv     10Gi        RWX           Retain          Available                                7s

{% endhighlight %}

Now that our persistent volume is in available state,
we can claim it by creating
[persistent volume claim policy](http://kubernetes.io/docs/user-guide/persistent-volumes/#persistentvolumeclaims).

We can define persistent volume claim using following template `test-pvc.yml`.

{% highlight yaml %}

kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-pvc
  labels:
    type: amazonEBS
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi

{% endhighlight %}

Let's create persistent volume claime using above template.

{% highlight bash %}
$ kubectl create -f  test-pvc.yml

persistentvolumeclaim "test-pvc" created

{% endhighlight %}

After creating
the persistent volume claim, our
persistent volume will change from `available` state to `bound` state.

{% highlight bash %}

$ kubectl get pv
NAME       CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS     CLAIM               REASON    AGE
test-pv    10Gi        RWX           Retain          Bound      default/test-pvc              2m

$kubectl get pvc
NAME        STATUS    VOLUME    CAPACITY   ACCESSMODES   AGE
test-pvc    Bound     test-pv   10Gi        RWX           1m

{% endhighlight %}

Now we have persistent volume claim available on our Kubernetes cluster,
Let's use it in deployment.

### Deploying Kubernetes application

We will use following deployment template as `test-pv-deployment.yml`.

{% highlight yaml %}

apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: test-pv
  labels:
    app: test-pv
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: test-pv
        tier: frontend
    spec:
      containers:
      - image: <your-repo>/<your-image-name>:latest
        name: test-pv
        imagePullPolicy: Always
        env:
        - name: APP_ENV
          value: staging
        - name: UNICORN_WORKER_PROCESSES
          value: "2"
        volumeMounts:
        - name: test-volume
          mountPath: "/<path-to-my-app>/shared/data"
        ports:
        - containerPort: 80
      imagePullSecrets:
        - name: registrypullsecret
      volumes:
      - name: test-volume
        persistentVolumeClaim:
          claimName: test-pvc


{% endhighlight %}

Now launch the deployment using following command.

{% highlight bash %}

$ kubectl create -f  test-pvc.yml
deployment "test-pv" created

{% endhighlight %}

Once the deployment is up and running all the contents on `shared` directory will
be stored on persistent volume claim.
Further when pod or deployment crashes for
any reason our data will be always retained
on the persistent volume.
We can use it to launch the application deployment.

This solved our goal of retaining data across deployments across `pod`
restarts.

