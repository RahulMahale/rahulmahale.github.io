---
layout: post
title: Using Kubernetes Configmap with configuration files for deploying Rails applications
published: true
categories: [Kubernetes]
description: Using Kubernetes Configmap with configuration files for deploying Rails applications
author_github: rahulmahale
---

This post assumes that you have basic understanding of
[Kubernetes](http://kubernetes.io/)
terms like
[pods](http://kubernetes.io/docs/user-guide/pods/)
and
[deployments](http://kubernetes.io/docs/user-guide/deployments/).

We deploy our Rails applications on Kubernetes and frequently do rolling deployments.

While performing application deployments on kubernetes cluster, sometimes we need to change the application configuration file.
Changing this application configuration file means we
need to change source code, commit the change and then go through the complete deployment process.

This gets cumbersome for simple changes.

Let's take the case of wanting to add queue in sidekiq configuration.

We should be able to change configuration and restart the pod instead of modifying the source-code, creating a new image and then performing a new deployment.

This is where Kubernetes's [ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configmap/) comes handy.
It allows us to handle configuration files much more efficiently.

Now we will walk you through the process of managing sidekiq configuration file using configmap.

## Starting with configmap

First we need to create a configmap.
We can either create it using `kubectl create configmap` command
or
we can use a yaml template.

We will be using yaml template `test-configmap.yml` which already has sidekiq configuration.

{% highlight yaml %}

apiVersion: v1
kind: ConfigMap
metadata:
  name: test-staging-sidekiq
  labels:
    name: test-staging-sidekiq
  namespace: test
data:
  config: |-
    ---
    :verbose: true
    :environment: staging
    :pidfile: tmp/pids/sidekiq.pid
    :logfile: log/sidekiq.log
    :concurrency: 20
    :queues:
      - [default, 1]
    :dynamic: true
    :timeout: 300

{% endhighlight %}

The above template creates configmap in the `test` namespace and is only accessible in that namespace.

Let's launch this configmap using following command.

{% highlight bash %}

$ kubectl create -f  test-configmap.yml
configmap "test-staging-sidekiq" created

{% endhighlight %}

After that let's use this configmap to create our `sidekiq.yml` configuration file in deployment template named `test-deployment.yml`.

{% highlight yaml %}
---
apiVersion: extensions/v1beta1
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
                path:  sidekiq.yml
      imagePullSecrets:
        - name: registrykey

{% endhighlight %}

Now let's create a deployment using above template.

{% highlight bash %}

$ kubectl create -f  test-deployment.yml
deployment "test-pv" created

{% endhighlight %}

Once the deployment is created, pod  running from that deployment will  start sidekiq using the `sidekiq.yml` mounted at `/etc/sidekiq/config/sidekiq.yml`.

Let's check this on the pod.

{% highlight bash %}

deployer@test-staging-2766611832-jst35:~$ cat /etc/sidekiq/config/sidekiq_1.yml
---
:verbose: true
:environment: staging
:pidfile: tmp/pids/sidekiq_1.pid
:logfile: log/sidekiq_1.log
:concurrency: 20
:timeout: 300
:dynamic: true
:queues:
  - [default, 1]

{% endhighlight %}

Our sidekiq process uses this configuration to start sidekiq.
Looks like configmap did its job.

Further if we want to add one new queue to sidekiq,
we can simply modify the configmap template and restart the pod.

For example if we want to add `mailer` queue we will modify template as shown below.

{% highlight yaml %}

apiVersion: v1
kind: ConfigMap
metadata:
  name: test-staging-sidekiq
  labels:
    name: test-staging-sidekiq
  namespace: test
data:
  config: |-
    ---
    :verbose: true
    :environment: staging
    :pidfile: tmp/pids/sidekiq_1.pid
    :logfile: log/sidekiq_1.log
    :concurrency: 20
    :queues:
      - [default, 1]
      - [mailer, 1]
    :dynamic: true
    :timeout: 300

{% endhighlight %}

Let's launch this configmap using following command.

{% highlight bash %}

$ kubectl apply -f  test-configmap.yml
configmap "test-staging-sidekiq" configured

{% endhighlight %}

Once the post is restarted, it will use new sidekiq configuration fetched from the configmap.

In this way, we keep our Rails application configuration files out of the source-code and tweak them as needed.
