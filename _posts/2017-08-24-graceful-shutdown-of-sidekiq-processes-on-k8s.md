---
layout: post
title: Graceful shutdown of Sidekiq processes on Kubernetes
published: true
categories: [Kubernetes]
description: Blog post on how to handle graceful shutdown of processes like Sidekiq on Kubernetes
author_github: rahulmahale
---

In our last [blog](http://blog.bigbinary.com/2017/07/25/deploying-rails-applications-using-kubernetes-with-zero-downtime.html), we explained how to handle
rolling deployments of Rails applications with no downtime.

In this article we will walk you through
how to handle graceful shutdown of processes in Kubernetes.

This post assumes that you have basic understanding of
[Kubernetes](http://kubernetes.io/)
terms like
[pods](http://kubernetes.io/docs/user-guide/pods/)
and
[deployments](http://kubernetes.io/docs/user-guide/deployments/).

### Problem

When we deploy Rails applications on kubernetes
it stops existing pods and spins up new ones.
When old pod is terminated by Replicaset,
then active Sidekiq processes are also terminated.
We run our batch jobs using sidekiq and it is possible that
sidekiq jobs might be running when deployment is being performed.
Terminating old pod during deployment can kill the already running jobs.

### Solution #1

As per default
[pod termination](https://kubernetes.io/docs/concepts/workloads/pods/pod/#termination-of-pods)
policy of kubernetes, kubernetes sends command to delete pod with a default grace period of 30 seconds.
At this time kubernetes sends TERM signal.
When the grace period expires, any processes still running in the Pod are killed with SIGKILL.

We can adjust the `terminationGracePeriodSeconds` timeout as per our need and can change it from
30 seconds to 2 minutes.


However there might be cases where we are not
sure how much time a process takes to gracefully shutdown.
In such cases we should consider using
 `PreStop` hook which is our next solution.

### Solution #2

Kubernetes provides many
[Container lifecycle hooks](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/).

`PreStop` hook is called immediately before a container is terminated.
It is a blocking call. It means it is synchronous.
It also means that this hook must be completed before
the container is terminated.

Note that unlike solution1 this solution is not time bound.
Kubernetes will wait as long as it takes for `PreStop` process
to finish. It is never a good idea to have a process which takes more
than a minute to shutdown but in real world there are cases
where more time is needed. Use `PreStop` for such cases.

We decided to use `preStop` hook to stop Sidekiq because we had some really long running processes.

### Using PreStop hooks in Sidekiq deployment

This is a simple deployment template which terminates
[Sidekiq process](https://github.com/mperham/sidekiq/wiki/Signals)
when pod is terminated during deployment.

{% highlight yaml %}

apiVersion: v1
kind: Deployment
metadata:
  name: test-staging-sidekiq
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
        - name: REDIS_HOST
          value: test-staging-redis
        - name: APP_ENV
          value: staging
        - name: CLIENT
          value: test
        volumeMounts:
            - mountPath: /etc/sidekiq/config
              name: test-staging-sidekiq
        ports:
        - containerPort: 80
      volumes:
        - name: test-staging-sidekiq
          configMap:
             name: test-staging-sidekiq
             items:
              - key: config
                path: sidekiq.yml
      imagePullSecrets:
        - name: registrykey

{% endhighlight %}

Next we will use `PreStop` lifecycle hook to stop
Sidekiq safely before pod termination.

We will add the following block in deployment manifest.

{% highlight yaml %}
lifecycle:
     preStop:
        exec:
          command: ["/bin/bash", "-l", "-c", "cd /opt/myapp/current; for f in tmp/pids/sidekiq*.pid; do bundle exec sidekiqctl stop $f; done"]
{% endhighlight %}

`PreStop` hook stops all the
Sidekiq processes and does graceful shutdown of Sidekiq
before terminating the pod.

We can add this configuration in original deployment manifest.

{% highlight yaml %}

apiVersion: v1
kind: Deployment
metadata:
  name: test-staging-sidekiq
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
        lifecycle:
          preStop:
            exec:
              command: ["/bin/bash", "-l", "-c", "cd /opt/myapp/current; for f in tmp/pids/sidekiq*.pid; do bundle exec sidekiqctl stop $f; done"]
        env:
        - name: REDIS_HOST
          value: test-staging-redis
        - name: APP_ENV
          value: staging
        - name: CLIENT
          value: test
        volumeMounts:
            - mountPath: /etc/sidekiq/config
              name: test-staging-sidekiq
        ports:
        - containerPort: 80
      volumes:
        - name: test-staging-sidekiq
          configMap:
             name: test-staging-sidekiq
             items:
              - key: config
                path: sidekiq.yml

      imagePullSecrets:
        - name: registrykey

{% endhighlight %}


Let's launch this deployment and monitor the rolling deployment.

{% highlight bash %}

$ kubectl apply -f test-deployment.yml
deployment "test-staging-sidekiq" configured

{% endhighlight %}

We can confirm that existing Sidekiq jobs are completed
before termination of old pod during the deployment process.
In this way we handle a graceful shutdown of
Sidekiq process. We can apply this technique to other processes
as well.
