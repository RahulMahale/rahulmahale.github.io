---
layout: post
title: Deploying Rails applications on Kubernetes cluster with Zero downtime
published: true
categories: [Kubernetes]
description: Deploying Rails applications on Kubernetes cluster with Zero downtime
author_github: rahulmahale
---

This post assumes that you have basic understanding of
[Kubernetes](http://kubernetes.io/)
terms like
[pods](http://kubernetes.io/docs/user-guide/pods/)
and
[deployments](http://kubernetes.io/docs/user-guide/deployments/).

### Problem

We deploy Rails applications on Kubernetes frequently
and
we need to ensure that
deployments do not cause any downtime.
When we used Capistrano to manage deployments
it was much easier since
it has provision to restart services in the rolling fashion.

Kubernetes restarts pods directly
and
any process already running on the pod is terminated.
So on rolling deployments we face downtime
until the new pod is up and running.

### Solution

In Kubernetes we have
[readiness probes and liveness probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/).
Liveness probes take care of keeping pod live
while readiness probe is responsible for keeping pods ready.

This is what Kubernetes documentation has to say about
when to use readiness probes.

>Sometimes, applications are temporarily unable to serve traffic. For example, an application might need to load large data or configuration files during startup. In such cases, you don’t want to kill the application, but you don’t want to send it requests either. Kubernetes provides readiness probes to detect and mitigate these situations. A pod with containers reporting that they are not ready does not receive traffic through Kubernetes Services.

It means
new traffic should not be routed to
those pods which are currently running but
are not ready yet.

### Using readiness probes in deployment flow

Here is what we are going to do.

* We will use readiness probes to deploy our Rails app.
* Readiness probes definition has to be specified in pod  `spec` of deployment.
* Readiness probe uses health check to detect the pod readiness.
* We will create a simple file on our pod with name `health_check` returning status `200`.
* This health check runs on arbitrary port 81.
* We will expose this port in nginx config running on a pod.
* When our application is up on nginx this health_check returns `200`.
* We will use above fields to configure health check in pod's spec of deployment.

Now let's build test deployment manifest.

{% highlight yaml %}
---
apiVersion: v1
kind: Deployment
metadata:
  name: test-staging
  labels:
    app: test-staging
  namespace: test
spec:
  template:
    metadata:
      labels:
        app: test-staging
    spec:
      containers:
      - image: <your-repo>/<your-image-name>:latest
        name: test-staging
        imagePullPolicy: Always
       env:
        - name: POSTGRES_HOST
          value: test-staging-postgres
        - name: APP_ENV
          value: staging
        - name: CLIENT
          value: test
        ports:
        - containerPort: 80
      imagePullSecrets:
        - name: registrykey

{% endhighlight %}

This is a simple deployment template which will terminate pod on the rolling deployment.
Application may suffer a downtime until the pod is in running state.

Next we will use readiness probe to define that pod is ready to accept the application traffic.
We will add the following block in deployment manifest.

{% highlight yaml %}
  readinessProbe:
    httpGet:
      path: /health_check
      port: 81
    periodSeconds: 5
    successThreshold: 3
    failureThreshold: 2
{% endhighlight %}

In above rediness probe definition `httpGet` checks the health check.

Health-check queries application on the file `health_check` printing `200` when accessed over port `81`.
We will poll it for each 5 seconds with the field `periodSeconds`.

We will mark pod as ready only if we get a successful health_check count for 3 times.
Similarly, we will mark it as a failure if we get failureThreshold twice.
This can be adjusted as per application need.
This helps deployment to determine if the pod is in ready status or not.
With readiness probes for rolling updates, we will use `maxUnavailable` and `maxSurge` in deployment strategy.

As per Kubernetes documentation.

>**`maxUnavailable`** is a field that specifies the maximum number of Pods
that can be unavailable during the update process.
The value can be an absolute number (e.g. 5) or a percentage of desired Pods (e.g. 10%).
The absolute number is calculated from percentage by rounding down.
This can not be 0.

and

>**`maxSurge`**  is field that specifies
The maximum number of Pods
that can be created above the desired number of Pods.
Value can be an absolute number (e.g. 5) or
a percentage of desired Pods (e.g. 10%).
This cannot be 0 if MaxUnavailable is 0.
The absolute number is calculated from percentage by rounding up.
By default, a value of 25% is used.

Now we will update our deployment manifests with
two replicas and the rolling update strategy by specifying the following parameters.

{% highlight yaml %}
  replicas: 2
  minReadySeconds: 50
  revisionHistoryLimit: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 50%
      maxSurge: 1
{% endhighlight %}

This makes sure that on deployment one of our pods is always running
and
at most 1 more pod can be created while deployment.

We can read more about rolling-deployments
[here](https://kubernetes.io/docs/concepts/workloads/controllers/deployment).

We can add this configuration in original deployment manifest.

{% highlight yaml %}

apiVersion: v1
kind: Deployment
metadata:
  name: test-staging
  labels:
    app: test-staging
  namespace: test
spec:
  replicas: 2
  minReadySeconds: 50
  revisionHistoryLimit: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 50%
      maxSurge: 1
  template:
    metadata:
      labels:
        app: test-staging
    spec:
      containers:
      - image: <your-repo>/<your-image-name>:latest
        name: test-staging
        imagePullPolicy: Always
       env:
        - name: POSTGREs_HOST
          value: test-staging-postgres
        - name: APP_ENV
          value: staging
        - name: CLIENT
          value: test
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /health_check
            port: 81
          periodSeconds: 5
          successThreshold: 3
          failureThreshold: 2
      imagePullSecrets:
        - name: registrykey

{% endhighlight %}


Let's launch this deployment using the command given below and monitor the rolling deployment.

{% highlight bash %}

$ kubectl apply -f test-deployment.yml
deployment "test-staging-web" configured

{% endhighlight %}

After the deployment is configured we can check the pods and how they are restarted.

We can also access the application to check if we face any down time.

{% highlight bash %}

$ kubectl  get pods
    NAME                                  READY      STATUS  RESTARTS    AGE
test-staging-web-372228001-t85d4           1/1       Running   0          1d
test-staging-web-372424609-1fpqg           0/1       Running   0          50s

{% endhighlight %}

We can see above that only one pod is re-created at the time
and
one of the old pod is serving the application traffic.
Also, new pod is running but not ready as it has not yet passed the readiness probe condition.

After sometime when the new pod is in ready state,
old pod is re-created and traffic is served by the new pod.
In this way, our application does not suffer any down-time and
we can confidently do deployments even at peak hours.
