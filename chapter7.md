# 第七章 使用多容器 Pods 扩展应用程序

We met Pods in chapter 2, when you learned that you can run many containers in one Pod, but you didn’t actually do it. In this chapter, you’re going to see how it works and understand the patterns it enables. This is the first of the more advanced topics in this part of the book, but it’s not a complicated subject—it just helps to have all the background knowledge from the previous chapters. Conceptually, it’s quite simple: one Pod runs many containers, which is typically your app container plus some helper containers. It’s what you can do with those helpers that makes this feature so interesting.

Containers in a Pod share the same virtual environment, so when one container takes an action, other containers can see it and react to it. They can even modify the intended action without the original container knowing. This behavior lets you model your application so that the app container is very simple—it just focuses on its work, and it has helpers that take care of integrating the app with other components and with the Kubernetes platform. It’s a great way to add a consistent management API to all your apps, whether new or legacy.

## 7.1 Pod 中多个容器如何通信

The Pod is a virtual environment that creates a shared networking and filesystem space for one or more containers. The containers are isolated units; they have their own processes and environment variables, and they can use different images with different technology stacks. The Pod is a single unit, so when it is allocated to run on a node, all the Pod containers run on the same node. You can have one container running Python and another running Java, but you can’t have some Linux and some Windows containers in the same Pod (yet), because Linux containers need to run on a Linux node and Windows containers on a Windows node.

Containers in a Pod share the network, so each container has the same IP address—the IP address of the Pod. Multiple containers can receive external traffic, but they need to listen on different ports, and containers within the Pod can communicate using the localhost address. Each container has its own filesystem, but it can mount volumes from the Pod, so containers can exchange information by sharing the same mounts. Figure 7.1 shows the layout of a Pod with two containers.

![Figure7.1](./images/Figure7.1.png)
**Figure 7.1 The Pod is a shared network and storage environment for many containers.**

That’s all the theory we need for now, and as we go through the chapter, you’ll be surprised at some of the smart things you can do just with shared networking and disk. We’ll start with some simple exercises in this section to explore the Pod environment. Listing 7.1 shows the multicontainer Pod spec for a Deployment. Two containers are defined that happen to use the same image, and they both mount an EmptyDir volume, which is defined in the Pod.

**Listing 7.1 sleep-with-file-reader.yaml, a simple multicontainer Pod spec**

```
spec:
  containers: # The containers field is an array.
    - name: sleep
      image: kiamol/ch03-sleep
      volumeMounts:
        - name: data
          mountPath: /data-rw # Mounts a volume as writable
    - name: file-reader # Containers need different names.
      image: kiamol/ch03-sleep # But containers can use the same or
                               # different images.
      volumeMounts:
        - name: data
          mountPath: /data-ro
          readOnly: true # Mounts the same volume as read-only
  volumes:
    - name: data # Volumes can be mounted by many
      containers.
        emptyDir: {}
```

This is a single Pod spec that runs two containers. When you deploy it, you’ll see that there are some differences in how you work with multicontainer Pods.

**TRY IT NOW** Deploy listing 7.1, and run a Pod with two containers.

```
# switch to the chapter folder:
cd ch07
# deploy the Pod spec:
kubectl apply -f sleep/sleep-with-file-reader.yaml
# get the detailed Pod information:
kubectl get pod -l app=sleep -o wide
# show the container names:
kubectl get pod -l app=sleep -o jsonpath='{.items[0].status.containerStatuses[*].name}'
# check the Pod logs—this will fail:
kubectl logs -l app=sleep
```

My output, which appears in figure 7.2, shows the Pod has two containers with a single IP address, which both run on the same node. You can see the details of the Pod as a single unit, but you can’t print the logs at a Pod level; you need to specify a container from which to fetch the logs.

Both of the containers in that exercise use the sleep image, so they’re not doing anything, but the containers keep running, and the Pod stays available to work with. The containers both mount the EmptyDir volume from the Pod, so that’s a shared part of the filesystem, and you can use it in both containers.

![Figure7.2](./images/Figure7.2.png)
**Figure 7.2 You always work with a Pod as a single unit, except when you need to specify a container.**

**TRY IT NOW** One container mounts the volume as read-write and the other one as read-only. You can write files in one container and read them in the other.

```
# write a file to the shared volume using one container:
kubectl exec deploy/sleep -c sleep -- sh -c 'echo ${HOSTNAME} > /data-rw/hostname.txt'
# read the file using the same container:
kubectl exec deploy/sleep -c sleep -- cat /data-rw/hostname.txt
# read the file using the other container:
kubectl exec deploy/sleep -c file-reader -- cat /data-ro/hostname.txt
# try to add to the file to the read-only container—this will fail:
kubectl exec deploy/sleep -c file-reader -- sh -c 'echo more >> /data-ro/hostname.txt'
```

You’ll see when you run this exercise that the first container can write data into the shared volume, and the second container can read it, but it can’t write data itself. That’s because the volume mount is defined as read-only for the second container in this Pod spec. It’s not a generic Pod limitation; mounts can be defined as writable for multiple containers if you need that. Figure 7.3 shows my output.

![Figure7.3](./images/Figure7.3.png)
**Figure 7.3 Containers can mount the same Pod volume to share data but with different access levels.**

A good old empty directory volume shows its worth again here; it’s a simple scratch pad that all the Pod containers can access. Volumes are defined at the Pod level and mounted at the container level, which means you can use any type of volume or PVC and make it available for many containers to use. Decoupling the volume definition from the volume mount also allows selective sharing, so one container may be able to see Secrets whereas the others can’t.

The other shared space is the network, where containers can listen on different ports and provide independent pieces of functionality. This is useful if your app con- tainer is doing some background work but doesn’t have any features to report on progress. Another container in the same Pod can provide a REST API, which reports on what the app container is doing.
Listing 7.2 shows a simplified version of this process. This is an update to the sleep deployment that replaces the file-sharing container with a new container spec that runs a simple HTTP server.

**Listing 7.2 sleep-with-server.yaml, running a web server in a second container**

```
spec:
  containers:
    - name: sleep
      image: kiamol/ch03-sleep # The same container spec as listing 7.1
    - name: server
      image: kiamol/ch03-sleep # The second container is different.
      command: ['sh', '-c', "while true; do echo -e 'HTTP/1.1 ..."]
    ports:
      - containerPort: 8080 # Including the port just documents
                            # which port the application uses.
```

Now the Pod will run with the original app container—the sleep container, which isn’t really doing anything—and a server container, which provides an HTTP endpoint on port 8080. The two containers share the same network space, so the sleep container can access the server using the localhost address.

**TRY IT NOW** Update the sleep Deployment using the file from listing 7.2, and confirm that the server container is accessible.

```
# deploy the update:
kubectl apply -f sleep/sleep-with-server.yaml
# check the Pod status:
kubectl get pods -l app=sleep
# list the container names in the new Pod:
kubectl get pod -l app=sleep -o jsonpath='{.items[0].status.containerStatuses[*].name}'
# make a network call between the containers:
kubectl exec deploy/sleep -c sleep -- wget -q -O - localhost:8080
# check the server container logs:
kubectl logs -l app=sleep -c server
```

You can see my output in figure 7.4. Although these are separate containers, at the network level they function as though they were different processes running on the same machine, using the local address for communication.

It’s not just within the Pod that the network is shared. The Pod has an IP address on the cluster, and if any containers in the Pod are listening on ports, then other Pods can access them. You can create a Service that routes traffic to the Pod on a specific port, and whichever container is listening on that port will receive the request.

**TRY IT NOW** Use a kubectl command to expose the Pod port—this is a quick way to create a Service without writing YAML—and then test that the HTTP server is accessible externally.

```
# create a Service targeting the server container port:
kubectl expose -f sleep/sleep-with-server.yaml --type LoadBalancer --port 8020 --target-port 8080
# get the URL for your service:
kubectl get svc sleep -o jsonpath='http://{.status.loadBalancer.ingress[0].*}:8020'
# open the URL in your browser
# check the server container logs:
kubectl logs -l app=sleep -c server
```

![Figure7.4](./images/Figure7.4.png)
**Figure 7.4 Network communication between containers in the same Pod is over localhost.**

Figure 7.5 shows my output. From the outside world, it’s just network traffic going to a Service, which gets routed to a Pod. The Pod is running multiple containers, but that’s a detail that is hidden from the consumer.

You should be getting a feel for how powerful running multiple containers in a Pod is, and in the rest of the chapter, we’ll put the ideas to work in real-world scenarios. There’s one thing that needs to be stressed, though: a Pod is not a replacement for a VM, so don’t think you can run all the components of an app in one Pod. You might be tempted to model an app like that, with a web server container and an API container running in the same Pod—don’t. A Pod is a single unit, and it should be used for a single component of your app. Additional containers can be used to support the app container, but you shouldn’t be running different apps in the same Pod. Doing so ruins your ability to update, scale, and manage those components independently.

![Figure7.5](./images/Figure7.5.png)
**Figure 7.5 Services can route network requests to any Pod containers that have published ports.**

## 7.2 使用 init 容器设置应用程序

So far we’ve run Pods with multiple containers where all the containers run in parallel: they start together, and the Pod isn’t considered to be ready until all the containers are ready. You’ll hear that referred to as the sidecar pattern, which reinforces the idea that additional containers (the sidecars) play a supporting role to the application container (the motorcycle). There’s another pattern that Kubernetes supports when you need a container to run before the app container to set up part of the environment. This is called an init container.

Init containers work differently from sidecars. You can have multiple init containers defined for the Pod, and they run in sequence, in the order in which they’re written in the Pod spec. Each init container needs to complete successfully before the next one starts, and all must complete successfully before the Pod containers are started. Figure 7.6 shows the startup sequence for a Pod with init containers.

All containers can access volumes defined in the Pod, so the major use case is for an init container to write data that prepares the environment for the application container. Listing 7.3 shows a simple extension to the HTTP server in the sleep Pod from the previous exercise. An init container runs and generates an HTML file, which it writes in a mount for an EmptyDir volume. The server container responds to HTTP requests by sending the contents of that file.

![Figure7.6](./images/Figure7.6.png)
**Figure 7.6 Init containers are useful for startup tasks to prepare the Pod for the app containers.**

**Listing 7.3 sleep-with-html-server.yaml, an init container in the Pod spec**

```
spec: # Pod spec in the Deployment template
  initContainers: # Init containers have their own array,
    - name: init-html # and they run in sequence.
      image: kiamol/ch03-sleep
      command: ['sh', '-c', "echo '<!DOCTYPE html...' > /data/index.html"]
      volumeMounts:
    - name: data
      mountPath: /data # Init containers can mount Pod volumes.
```

This example uses the same sleep image for the init container, but it can be any image. You might use an init container to set up the application environment using tools that you don’t want to be present in the running application. An init container can use a Docker image with the Git command line installed and clone a repository into the shared filesystem. The app container can access to the files without you having to set up the Git client in your app image.

**TRY IT NOW** Deploy the update from listing 7.3, and see how init containers work.

```
# apply the updated spec with the init container:
kubectl apply -f sleep/sleep-with-html-server.yaml
# check the Pod containers:
kubectl get pod -l app=sleep -o jsonpath='{.items[0].status.containerStatuses[*].name}'
# check the init containers:
kubectl get pod -l app=sleep -o jsonpath='{.items[0].status.initContainerStatuses[*].name}'
# check logs from the init container—there are none:
kubectl logs -l app=sleep -c init-html
# check that the file is available in the sidecar:
kubectl exec deploy/sleep -c server -- ls -l /data-ro
```

You’ll pick up a few things from this exercise. App containers are guaranteed not to run until init containers complete successfully, so your app can safely make assumptions about the environment that the init container prepares. In this case, the HTML file is sure to exist before the server container starts. Init containers are a different part of the Pod spec, but some management features work in the same way as app containers—you can read the logs from init containers even after they have exited. My output appears in figure 7.7.

![Figure7.7](./images/Figure7.7.png)
**Figure 7.7 Init containers are useful for preparing the Pod environment for app and sidecar containers.**

That still isn’t a very real-world example, though, so let’s do something better. We covered app configuration in chapter 4 and saw how to use environment variables, ConfigMaps, and Secrets to build up a hierarchy of configuration settings. That’s great if your app supports it, but many older apps don’t have that flexibility; they expect to find a single config file in one place, and they don’t go looking anywhere else. Let’s look at an app like that.

**TRY IT NOW** This chapter has a new demo app, because if I’m getting bored with looking at Pi, then you must be, too. This one isn’t much more fun, but at least it’s different. It just writes a timestamp to a log file every few seconds. It has an old-style configuration framework, so we can’t use any of the configuration techniques we’ve learned so far.

```
# run the app, which uses a single config file:
kubectl apply -f timecheck/timecheck.yaml
# check the container logs—there won’t be any:
kubectl logs -l app=timecheck
# check the log file inside the container:
kubectl exec deploy/timecheck -- cat /logs/timecheck.log
# check the config setup:
kubectl exec deploy/timecheck -- cat /config/appsettings.json
```

You can see my output in figure 7.8. A limited configuration framework isn’t the only reason this app isn’t a good citizen in a container platform—there are no logs in the Pod, either—but we can address all the problems with additional containers in the Pod.

![Figure7.8](./images/Figure7.8.png)
**Figure 7.8 Older apps that use a single configuration source can’t benefit from a configuration hierarchy.**

An init container is a perfect tool to bring this app into line with the configuration approach we want to use for all our apps. We can store the settings in ConfigMaps, Secrets, and environment variables, and use an init container to read from all the different inputs, merge the contents, and write the output to the single file location that the app uses. Listing 7.4 shows the init container in the Pod spec.

**Listing 7.4 timecheck-with-config.yaml, an init container that writes configuration**

```
spec:
  initContainers:
    - name: init-config
      image: kiamol/ch03-sleep # This image has the jq tool.
      command: ['sh', '-c', "cat /config-in/appsettings.json | jq --arg
    APP_ENV \"$APP_ENVIRONMENT\" '.Application.Environment=$APP_ENV' >
    /config-out/appsettings.json"]
     env:
       - name: APP_ENVIRONMENT # All containers have their own environment
         value: TEST # variables—they're not shared in the Pod.
     volumeMounts:
       - name: config-map # Mounts a ConfigMap volume to read
         mountPath: /config-in
       - name: config-dir
         mountPath: /config-out # Mounts an EmptyDir volume to write to
```

There are a few things to note before we update the deployment:
- The init container uses the jq tool, which the app doesn’t need. The containers use different images, each with just the tools necessary to run that step.
- The command in the init container reads from a ConfigMap volume mount, merges the environment variable values, and writes out to an EmptyDir volume mount.
- The app container mounts the EmptyDir volume to the path where the config file needs to be. The file generated by the init container hides the default configuration in the app image.
- Containers don’t share environment variables. The settings are specified for the init container; the app container doesn’t see those.
- Containers map the volumes they need. Both containers mount the EmptyDir volume, which they share, but only the init container mounts the ConfigMap.
When we apply this update, the app’s behavior will change in line with the ConfigMap and environment variables, even though the app container doesn’t use them as configuration sources.

**TRY IT NOW** Update the timecheck app using listing 7.4 so the app container is configured from multiple sources.

```
# apply the ConfigMap and the new Deployment spec:
kubectl apply -f timecheck/timecheck-configMap.yaml -f timecheck/timecheck-with-config.yaml
# wait for the containers to start:
kubectl wait --for=condition=ContainersReady pod -l app=timecheck,version=v2
# check the log file in the new app container:
kubectl exec deploy/timecheck -- cat /logs/timecheck.log
# see the config file built by the init container:
kubectl exec deploy/timecheck -- cat /config/appsettings.json
```

You’ll see when you run this that the app works with the new configuration, and the only change for the application container spec is that the config directory is mounted from the EmptyDir volume. My output is shown in figure 7.9.

This approach works because the config file is loaded from a dedicated directory. Remember that a volume mount overwrites a directory from the image, if it already exists. If the app loaded a config file from the same directory as the app binaries, you couldn’t do this because the EmptyDir mount would overwrite the whole app folder. In that scenario, you would need an additional step in the app container startup to copy the config file from the mount into the application directory.

![Figure7.9](./images/Figure7.9.png)
**Figure 7.9 Init containers can change app behavior without changes to the app code or Docker image.**

Applying a standard configuration approach to nonstandard apps is a great use for init containers, but older apps still won’t play nicely in a modern platform, and that’s where sidecar containers can help.

## 7.3 通过 adapter 容器以应用一致性

Moving apps to Kubernetes is a great opportunity to add a layer of consistency across all your apps, so you deploy and manage them the same way using the same tools, no matter what the app is doing, or what technology stack it uses, or when it was developed. My fellow Docker Captain, Sune Keller, has talked about the service hotel (https://bit.ly/376rBcF) concept they use at Alm Brand. Their container platform offers a set of guarantees for “customers” (like high availability and security), provided they adhere to the rules (like pulling the configuration from the platform and writing logs out to it). 

Not all apps know about the rules, and some of them can’t be applied by the platform from the outside, but sidecar containers run alongside the app container so they have a privileged position. You can use them as adapters, which understand some aspect of how the app works and adapts it to how the platform wants it to work. Logging is a classic example.
Every app writes some output to log entries—or should; otherwise, it would be entirely unmanageable, and you should refuse to work with it. Modern app platforms like Node.js and .NET Core write to the standard output stream, which is where Docker fetches container logs and where Kubernetes gets the Pod logs. Older apps have different ideas about logging, and they may write to files or other targets that never get surfaced as container logs, so you never see any Pod logs (see Appendix D in the ebook to learn more about logging in Docker). That’s what the timecheck app does, and we can fix it with a very simple sidecar container. The spec appears in listing 7.5.

**Listing 7.5 timecheck-with-logging.yaml, using a sidecar container to expose logs**

```
containers:
  - name: timecheck
    image: kiamol/ch07-timecheck
    volumeMounts:
      - name: logs-dir # The app container writes the log file
        mountPath: /logs # to an EmptyDir volume mount.
      # Abbreviated—the full spec also includes the config mount.
  - name: logger
    image: kiamol/ch03-sleep # The sidecar just watches the log file.
    command: ['sh', '-c', 'tail -f /logs-ro/timecheck.log']
    volumeMounts:
      - name: logs-dir
        mountPath: /logs-ro # Uses the same volume as the app
        readOnly: true
```

All the sidecar does is mount the log volume (go EmptyDir!) and use the standard Linux tail command to read from the log file. The -f option means the command will follow the file; effectively, it just sits and watches for new writes, and when any lines are written to the file, they’re echoed to standard out. It’s a relay that adapts the app’s actual logging implementation to the expectations of Kubernetes.

**TRY IT NOW** Apply the update from listing 7.5, and check the app logs are available.

```
# add the sidecar logging container:
kubectl apply -f timecheck/timecheck-with-logging.yaml
# wait for the containers to start:
kubectl wait --for=condition=ContainersReady pod -l app=timecheck,version=v3
# check the Pods:
kubectl get pods -l app=timecheck
# check the containers in the Pod:
kubectl get pod -l app=timecheck -o jsonpath='{.items[0].status.containerStatuses[*].name}'
# now you can see the app logs in the Pod:
kubectl logs -l app=timecheck -c logger
```

There’s some inefficiency here because the app container writes logs to a file and then the logging container reads them back out again. There will be a small time lag and potentially a lot of wasted disk, but the Pod will be replaced in the next app update, and all the space used in the volume will be reclaimed. The benefit is that this Pod now behaves like every other Pod, making application logs available to Kubernetes but without any changes needed to the app itself, as is shown in figure 7.10.

Receiving configuration from the platform and writing logs to the platform are pretty much the fundamentals for any application, but as your platform matures, you’ll have more expectations for standard behavior. You’ll want to be able to test that the application inside the container is healthy, and you’ll also want to be able to pull metrics from the app to see what it’s doing and how hard it’s working.

Sidecars can help there, too, either by running custom containers, which provide information tailored to the app, or by having standard health and metrics container images, which you apply to all your Pod specs. We’ll round off the exercises using the timecheck app and add those features that make it a good citizen for Kubernetes. We’ll cheat, though, with some more static HTTP server containers, which you can see in listing 7.6.

![Figure7.10](./images/Figure7.10.png)
**Figure 7.10 Adapters bring a layer of consistency to Pods, making old apps behave like new apps.**

**Listing 7.6 timecheck-good-citizen.yaml, more sidecars to extend the app**

```
containers: # The previous app and logging containers are the same.
  - name: timecheck
# ...
  - name: logger
# ...
  - name: healthz # A new sidecar that exposes a healthcheck API
    image: kiamol/ch03-sleep # This is just a static response.
    command: ['sh', '-c', "while true; do echo -e 'HTTP/1.1 200 OK\nContent-
      Type: application/json\nContent-Length: 17\n\n{\"status\": \"OK\"}' | nc
      -l -p 8080; done"]
    ports:
      - containerPort: 8080 # Available at port 8080 in the Pod
  - name: metrics # Another sidecar, which adds a metrics API
    image: kiamol/ch03-sleep # The content is static again.
    command: ['sh', '-c', "while true; do echo -e 'HTTP/1.1 200 OK\nContent-
      Type: text/plain\nContent-Length: 104\n\n# HELP timechecks_total The
      total number timechecks.\n# TYPE timechecks_total
      counter\ntimechecks_total 6' | nc -l -p 8081; done"]
      ports:
        - containerPort: 8081 # The content is avaialable on a different
                              # port.
```

The full YAML file also includes a ClusterIP Service, which publishes on port 8080 for the health endpoint and port 8081 for the metrics endpoint. In a production cluster, these would be used by other components to collect monitoring stats. The Deployment is an extension of the previous releases, so the app uses an init container for configuration and has a logging sidecar along with the new sidecars.

**TRY IT NOW** Deploy the update, and check the new management endpoints for the health and performance of the app.

```
# apply the update:
kubectl apply -f timecheck/timecheck-good-citizen.yaml
# wait for all the containers to be ready:
kubectl wait --for=condition=ContainersReady pod -l app=timecheck,version=v4
# check the running containers:
kubectl get pod -l app=timecheck -o jsonpath='{.items[0].status.containerStatuses[*].name}'
# use the sleep container to check the timecheck app health:
kubectl exec deploy/sleep -c sleep -- wget -q -O - http://timecheck:8080
# check its metrics:
kubectl exec deploy/sleep -c sleep -- wget -q -O - http://timecheck:8081
```

When you run the exercise, you’ll see everything works as expected, as shown in figure 7.11. You may also see the updates weren’t as speedy as you’re used to, with the new Pod taking longer to start up and the old Pod taking longer to terminate. The additional startup time is from having the init container, the app container, and all the sidecars—they all need to be ready before the new Pod is considered ready. The additional termination time is because the replaced Pod also had multiple containers, which are each given a grace period for the container process to shut down.

![Figure7.11](./images/Figure7.11.png)
**Figure 7.11 Multiple adapter sidecars give the app a consistent management API.**

There is an overhead to running all these sidecar containers as adapters. You’ve seen that it increases deployment times, but it also increases the ongoing compute requirements of the app—even storage and basic sidecars, which just tail log files and serve simple HTTP responses, all use memory and compute cycles. But if you want to move existing apps to Kubernetes that don’t have those features, it’s an acceptable approach to get all your apps behaving in the same way, as shown in figure 7.12.

In the previous exercise, we used an old sleep Pod we had lying around to call the new HTTP endpoints for the timecheck app. Remember that Kubernetes has a flat networking model, where Pods can send traffic to any other Pods via a Service. You may want more control over the network communication in your app, and you can do that with sidecars, too, by running a proxy container that manages the outgoing traffic from your app container.

![Figure7.12](./images/Figure7.12.png)
**Figure 7.12 A consistent management API makes it easy to work with Pods—it doesn’t matter how the API is provided inside the Pod.**

## 7.4 通过 ambassador 容器抽象连接

The ambassador pattern lets you control and simplify outgoing connections from your application: your app makes network requests to the localhost address, which are picked up and performed by the ambassador. You can make use of a generic ambassador container, or one that is specific to your application components, in several situations. Figure 7.13 shows some examples. The logic in the ambassador might be geared to improving performance or increasing reliability or security.

Taking control of the network away from the application is hugely powerful. A proxy container can do service discovery, load balancing, retries, and even layer encryption onto an unencrypted channel. Perhaps you’ve heard of the service mesh architecture, using technologies like Linkerd and Istio—they’re all powered by proxy sidecar containers in a variation of the ambassador pattern.

![Figure7.13](./images/Figure7.13.png)
**Figure 7.13 The ambassador pattern has lots of potential, from simplifying app logic to increasing performance.**

We won’t use a service mesh architecture here because that would take us well past lunchtime and on into the night, but we’ll get a flavor of what it can do with a simplified example. The starting point is the random-number app we’ve used before. There’s a web app running in a Pod, which consumes an API running in another Pod. The API is the only component the web app uses, so ideally we would restrict network calls to any other address, but in the initial deployment that doesn’t happen.

**TRY IT NOW** Run the random-number app, and verify that the web app container can use any network address.

```
# deploy the app and Services:
kubectl apply -f numbers/
# find the URL for your app:
kubectl get svc numbers-web -o jsonpath='http://{.status.loadBalancer.ingress[0].*}:8090'
# browse and get yourself a nice random number
# check that the web app has access to other endpoints:
kubectl exec deploy/numbers-web -c web -- wget -q -O -http://timecheck:8080
```

The web Pod can reach the API using the ClusterIP Service and the domain name numbers-api, but it can also access any other address, which could be a URL on the public internet or another ClusterIP Service. Figure 7.14 shows the app can read the health endpoint of the timecheck app—that should be a private endpoint, and it might expose information that is useful to someone up to no good.

![Figure7.14](./images/Figure7.14.png)
**Figure 7.14 Kubernetes doesn’t have any default restrictions on outgoing connections from Pod containers.**

You have a lot of options for restricting network access besides using a proxy sidecar, but the ambassador pattern comes with some additional features that make it worth considering. Listing 7.7 shows an update to the web app spec, using a simple proxy container as an ambassador.

**Listing 7.7 web-with-proxy.yaml, using a proxy as an ambassador**

```
containers:
  - name: web
    image: kiamol/ch03-numbers-web
    env:
      - name: http_proxy # Sets the container to use the proxy
        value: http://localhost:1080 # so traffic goes to the ambassador
      - name: RngApi__Url
        value: http://localhost/api # Uses a localhost address for the API
  - name: proxy
    image: kiamol/ch07-simple-proxy # This is a basic HTTP proxy.
      env:
        - name: Proxy__Port # Routes network requests from the app
          value: "1080" # using the configured URI mapping
        - name: Proxy__Request__UriMap__Source
          value: http://localhost/api
        - name: Proxy__Request__UriMap__Target
          value: http://numbers-api/sixeyed/kiamol/master/ch03/numbers/rng
```

This example shows the major pieces of the ambassador pattern: the app container uses localhost addresses for any services it consumes, and it’s configured to route all network calls through the proxy container. The proxy is a custom app that logs network calls, maps localhost addresses to real addresses, and blocks any addresses that are not listed in the map. All that becomes functionality in the Pod, but it’s transparent to the application container.

**TRY IT NOW** Update the random-number app, and confirm the network is now locked down.

```
# apply the update from listing 7.5:
kubectl apply -f numbers/update/web-with-proxy.yaml
# refresh your browser, and get a new number
# check the proxy container logs:
kubectl logs -l app=numbers-web -c proxy
# try to read the health of the timecheck app:
kubectl exec deploy/numbers-web -c web -- wget -q -O -http://timecheck:8080
# check proxy logs again:
kubectl logs -l app=numbers-web -c proxy
```

Now the web app is decoupled even further from the API, because it doesn’t even know the URL of the API—that’s set in the ambassador, which can be configured independently of the app. The web app is also restricted to using a single address for outgoing requests, and all of those calls are logged by the proxy, as you see in figure 7.15.

The ambassador for this web app proxies HTTP calls outside of the Pod, but the ambassador pattern is wider than that. It plugs into the network at the transport layer,so it can work on any kind of traffic. A database ambassador can make some smart choices, like sending queries to a read-only database replica and using only the master database for writes. That’s going to improve performance and scale, while keeping complex logic out of the application.

![Figure7.15](./images/Figure7.15.png)
**Figure 7.15 All network access is via the ambassador, which can implement its own access rules.**

We’ll round out the chapter by taking a closer look at what it means to use the Pod as a shared environment for many containers.

## 7.5 理解 Pod 环境

The Pod is a boundary around one or more containers, just like the container is a boundary around one or more processes. Pods create layers of virtualization without adding overhead, so they’re flexible and efficient. The cost of that flexibility is—as always—complexity, and you need to be aware of some nuances to working with multicontainer Pods.
The main thing to understand is that the Pod is still the single unit of compute, even if lots of containers are running inside it. Pods aren’t ready until all the containers in the Pod are ready, and Services send traffic only to Pods that are ready. Adding sidecars and init containers adds to the failure modes for your application.

**TRY IT NOW** You can break your application if an init container fails. This update to the numbers app won’t be successful because the init container is misconfigured.

```
# apply the update:
kubectl apply -f numbers/update/web-v2-broken-init-container.yaml
# check the new Pod:
kubectl get po -l app=numbers-web,version=v2
# check the logs for the new init container:
kubectl logs -l app=numbers-web,version=v2 -c init-version
# check the status of the Deployment:
kubectl get deploy numbers-web
# check the status of the ReplicaSets:
kubectl get rs -l app=numbers-web
```

You can see in this exercise that the failed init container effectively prevents the application from updating. The new Pod never enters the running state and won’t receive traffic from the Service. The Deployment never scales down the old ReplicaSet because the new one doesn’t reach the required level of availability, but the basic details of the Deployment look like the update has worked, as shown in figure 7.16.

The same situation will happen if a sidecar container fails on startup—the Pod doesn’t have all of its containers running so the Pod itself isn’t ready. Any deployment checks you have in place need to be extended for multicontainer Pods to ensure all init containers run to completion and all Pod containers are running. You need to be aware of the following restart conditions, too:

- If a Pod with init containers is replaced, then the new Pod runs all the init containers again. You must ensure your init logic can be run repeatedly.
- If you deploy a change to the init container image(s) for a Pod, that restarts the Pod. Init containers all execute again, and app containers are replaced.
- If you deploy a Pod spec change to the app container image(s), the app containers are replaced, but the init containers are not executed again.
- If an application container exits, then the Pod re-creates it. Until the container is replaced, the Pod is not fully running and won’t receive Service traffic.

The Pod is a single compute environment, but when you add multiple moving parts inside that environment, you need to test all the failure scenarios and make sure your app behaves as you expect.

There’s one last part of the Pod environment that we haven’t covered: the compute layer. Pod containers have a shared network and can share parts of the filesystem, but they can’t access each other’s processes—the container boundary still provides compute isolation. That’s the default behavior, but in some cases, you want your sidecar to have access to the processes in the application container, either for interprocess communication or so the sidecar can fetch metrics about the app process.

![Figure7.16](./images/Figure7.16.png)
**igure 7.16 Adding more containers to your Pod spec adds more opportunities for the Pod to fail**

You can enable this access with a simple setting in the Pod spec: shareProcess-Namespace: true. That means every container in the Pod shares the same compute space and can see each other’s processes.

**TRY IT NOW** Deploy an update to the sleep Pod so the containers use a shared compute space and can access each other’s processes.

```
# check the processes in the current container:
kubectl exec deploy/sleep -c sleep -- ps
# apply the update:
kubectl apply -f sleep/sleep-with-server-shared.yaml
# wait for the new containers:
kubectl wait --for=condition=ContainersReady pod -l app=sleep,version=shared
# check the processes again:
kubectl exec deploy/sleep -c sleep -- ps
```

You can see my output in figure 7.17. The sleep container can see all the server container’s processes, and it could happily kill them all and leave the Pod in a confused state.

![Figure7.17](./images/Figure7.17.png)
**Figure 7.17 You can configure a Pod so all containers can see all processes—use with care.**

That’s all for multicontainer Pods. You’ve seen in this chapter that you can use init containers to prepare the environment for your application container and run sidecar containers to add features to your app, all without changing the app code or the Docker image. There are some caveats to using multiple containers, but it’s a pattern you’ll use often to extend your applications. Just remember that the Pod should be one logical component: I don’t want to see you running Nginx, WordPress, and MySQL in a single Pod just because you can. Let’s tidy up now and get ready for the lab.

**TRY IT NOW** Remove everything matching this chapter’s label.

```
kubectl delete all -l kiamol=ch07
```

## 7.6 实验室

It’s back to the Pi app for this lab. The Docker image kiamol/ch05-pi can actually be used in different ways, and to run it as a web app, you need to override the startup command in the container spec. We’ve done that in the YAML files in previous chapters, but now we’ve been asked to use a standard approach to setting up the pod. Here are the requirements and some hints:

- The app container needs to use a standard startup command that all Pods in our platform are using. It should run /init/startup.sh.
- The Pod should use port 80 for the app container.
- The Pod should also publish port 8080 for an HTTP server, which returns the version number of the app,
- The app container image doesn’t contain a startup script, so you’ll need to usesomething that can create that script and make it executable for the app container to run.
- The app doesn’t publish a version API on port 8080 (or anywhere else), so you’ll need something that can provide that (it can just be any static text).

The starting point is the YAML in ch07/lab/pi, which is broken at the moment. You’ll need to do some investigation into how the app ran in previous chapters and apply the techniques we’ve learned in this chapter. You have plenty of ways to approach this one, and you’ll find my sample solution in the usual place: https://github.com/sixeyed/kiamol/blob/master/ch07/lab/README.md.