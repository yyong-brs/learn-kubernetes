# 第六章 通过 controllers 在多个 Pod 之间扩展应用

扩展应用程序的基本思想很简单:运行更多的pod。Kubernetes 将网络和存储从计算层中抽象出来，所以你可以运行许多 pod，它们是同一个应用程序的副本，只需将它们插入相同的抽象中即可。Kubernetes称这些pod为副本，在多节点集群中，它们将分布在许多节点上。这为您提供了规模的所有好处:更大的负载处理能力和故障时的高可用性——所有这些都在一个可以在几秒钟内扩大和缩小的平台中。

Kubernetes 还提供了一些可替代的扩展选项来满足不同的应用程序需求，我们将在本章中详细介绍这些选项。您最常使用的是 Deployment 控制器，它实际上是最简单的，但我们也将花时间讨论其他控制器，以便您了解如何在集群中扩展不同类型的应用程序。

## 6.1 Kubernetes 如何大规模运行应用程序

Pod 是 Kubernetes 中的计算单元，你在第二章中了解到你通常不会直接运行Pod;相反，您可以定义另一个资源来为您管理它们。该资源是一个控制器，从那时起我们就一直使用 Deployment 控制器。控制器 spec 配置包括一个 Pod Template，用于创建和替换Pod。它可以使用相同的 Template 创建Pod的多个副本。

Deployments 可能是您在Kubernetes中使用最多的资源，并且您已经有了很多使用它们的经验。现在是时候深入挖掘并了解 Deployment 实际上并不直接管理Pods——这是由另一个称为 ReplicaSet 的资源完成的。图6.1显示了Deployment、ReplicaSet和Pods之间的关系。

![图6.1](./images/Figure6.1.png)
<center>图 6.1 每个软件问题都可以通过添加另一个抽象层来解决</center>

在大多数情况下，你会使用 Deployment 来描述你的应用; Deployment 是一个管理 ReplicaSet 的控制器，ReplicaSet 是一个管理 Pods 的控制器。您可以直接创建 ReplicaSet，而不是使用Deployment，我们将在前几个练习中这样做，只是为了了解如何进行伸缩。ReplicaSet 的 YAML 与 Deployment 的 YAML 几乎相同;它需要一个选择器来查找它拥有的资源，需要一个 Pod 模板来创建资源。清单 6.1显 示了一个简短的 spec 配置。

> 清单 6.1 whoami.yaml, 一个不包含 Deployment 的 ReplicaSet

```
apiVersion: apps/v1
kind: ReplicaSet # Spec 配置与 deployment 基本相同
metadata:
  name: whoami-web
spec:
  replicas: 1
  selector: # selector 用于匹配 Pods
    matchLabels:
      app: whoami-web
  template: # 后面就是常规的 pod temlate.
```

在这个 spec 中，与我们使用的与 deployment 定义唯一不同的地方是对象类型 ReplicaSet 和 replicas字段，它声明要运行多少个pod。该 spec 使用单个副本，这意味着Kubernetes将运行单个Pod。

<b>现在就试试</b> 部署 ReplicaSet 和 LoadBalancer Service，它使用与 ReplicaSet 相同的标签选择器将流量发送到Pods。

```
# 进入本章练习目录:
cd ch06
# 部署 ReplicaSet 和 Service:
kubectl apply -f whoami/
# 检查资源:
kubectl get replicaset whoami-web
# 向 service 发起 http 请求:
curl $(kubectl get svc whoami-web -o jsonpath='http://{.status.loadBalancer.ingress[0].*}:8088')
# 删除所有的 Pods:
kubectl delete pods -l app=whoami-web
# 发起请求:
curl $(kubectl get svc whoami-web -o jsonpath='http://{.status.loadBalancer.ingress[0].*}:8088')
# 查看 ReplicaSet 明细:
kubectl describe rs whoami-web
```

可以在图6.2中看到我的输出。这里没有什么新东西;ReplicaSet拥有一个Pod，当你删除那个Pod时，ReplicaSet会替换它。我在最后的命令中删除了kubectl描述输出，但是如果运行它，您将看到它以一个事件列表结束，ReplicaSet在其中写入关于它如何创建Pods的活动日志。

![图6.2](./images/Figure6.2.png)
<center>图 6.2 使用 ReplicaSet 就像使用 Deployment: 它创建并管理Pods</center>

ReplicaSet 替换删除的 Pods 是因为它不断地运行一个控制循环，检查它拥有的对象数量是否与它应该拥有的副本数量相匹配。当您扩展应用程序时，您使用相同的机制—您更新ReplicaSet规范以设置新的副本数量，然后控制循环看到它需要更多副本，并从相同的 Pod 模板创建它们。

<b>现在就试试</b> 通过部署指定三个副本的更新 ReplicaSet 定义来扩展应用程序。

```
# 部署更新后的应用:
kubectl apply -f whoami/update/whoami-replicas-3.yaml
# 检查 Pods:
kubectl get pods -l app=whoami-web
# 删除所有 Pods:
kubectl delete pods -l app=whoami-web
# 再次检查:
kubectl get pods -l app=whoami-web
# 多发起几次 http 请求:
curl $(kubectl get svc whoami-web -o jsonpath='http://{.status.loadBalancer.ingress[0].*}:8088')
```

如图6.3所示，我的输出提出了几个问题:Kubernetes是如何快速扩展应用程序的，以及HTTP响应是如何来自不同的pod的?

![图6.3](./images/Figure6.3.png)
<center>图 6.3 扩展 ReplicaSets 是快速的，并且在规模上，一个 Service 可以将请求分发到许多pod</center>

第一个问题很简单:这是一个单节点集群，所以每个 Pod 都将运行在同一个节点上，并且该节点已经为应用程序拉取了 Docker 镜像。当你在生产集群中扩展时，很可能会安排新的 Pod 运行在本地没有镜像的节点上，它们需要在运行 Pod 之前拉取镜像。你可以缩放的速度受限于你的镜像可以被拉取的速度，这就是为什么你需要投入时间来优化你的镜像。

至于我们如何向同一个 Kubernetes 服务发出 HTTP 请求，并从不同的 pod 获得响应，这都取决于服务和 Pods 之间的松散耦合。当你扩大 ReplicaSet 时，突然有多个 pod 与服务的标签选择器匹配，当这种情况发生时，Kubernetes 在 pod 之间平衡负载请求。图 6.4 显示了相同的标签选择器如何维护 ReplicaSet 和 Pods 之间以及 Service 和 Pods 之间的关系。

![图6.4](./images/Figure6.4.png)
<center>图 6.4 具有与 ReplicaSet 相同标签选择器的 Service 将使用其所有 pod</center>

网络和计算之间的抽象使得 Kubernetes 的扩展如此容易。您现在可能正在经历一种温暖的感觉——突然，所有的复杂性都开始适应，您会看到资源之间的分离是如何促成一些非常强大的功能的。这是扩展的核心:你运行尽可能多的 pod，它们都位于一个 Service 后面。当消费者访问服务时，Kubernetes在Pods之间分配负载。

负载均衡是 Kubernetes 中所有服务类型的特性。在这些练习中，我们已经部署了一个 LoadBalancer Service，它接收到集群中的流量并将其发送到Pods。它还创建了一个 ClusterIP 供其他 pod 使用，当 pod 在集群内通信时，它们也受益于负载平衡。

<b>现在就试试</b> 部署一个新的 Pod，并使用它在内部调用 who-am-I 服务，使用 ClusterIP, Kubernetes从服务名称解析ClusterIP。

```
# 运行一个 sleep Pod:
kubectl apply -f sleep.yaml
# 检查 who-am-i Service 信息:
kubectl get svc whoami-web
# 在 sleep Pod 中为服务运行DNS查找:
kubectl exec deploy/sleep -- sh -c 'nslookup whoami-web | grep "^[^*]"'
# 发起一些 http 请求:
kubectl exec deploy/sleep -- sh -c 'for i in 1 2 3; do curl -w \\n -s http://whoami-web:8088; done;'
```

如图6.5所示，Pod 使用内部服务的行为与外部消费者的行为相同，并且请求在Pod之间是负载平衡的。当您运行这个练习时，您可能会看到请求完全均匀地分布，或者您可能会看到一些pod响应不止一次，这取决于网络的变化。

![图6.5](./images/Figure6.5.png)
<center>图 6.5 集群内部的世界:Pod-to-Pod网络还受益于服务负载均衡。</center>

在第3章中，我们讨论了服务，以及ClusterIP地址是如何从Pod的IP地址抽象出来的，因此当Pod被替换时，应用程序仍然可以使用相同的服务地址访问。现在您可以看到，服务可以是跨许多Pod的抽象，将流量路由到任何节点上的Pod的同一网络层可以跨多个Pod进行负载平衡。

## 6.2 使用 Deployments 和 ReplicaSets 来扩展负载

ReplicaSets make it incredibly easy to scale your app: you can scale up or down in seconds just by changing the number of replicas in the spec. It’s perfect for stateless components that run in small,lean containers, and that’s why applications built for Kubernetes typically use a distributed architecture, breaking down functionality across many pieces, which can be individually updated and scaled.

Deployments add a useful management layer on top of ReplicaSets. Now that we know how they work, we won’t be using ReplicaSets directly anymore—Deployments should be your first choice for defining applications. We won’t explore all the features of Deployments until we get to application upgrades and rollbacks in chapter 9,but it’s useful to understand exactly what the extra abstraction gives you. Figure 6.6 shows this.

![图6.6](./images/Figure6.6.png)

<center>图 6.6 Zero is a valid number of desired replicas; Deployments scale down old ReplicaSets to zero.</center>

A Deployment is a controller for ReplicaSets, and to run at scale, you include the same replicas field in the Deployment spec, and that is passed to the ReplicaSet. Listing 6.2 shows the abbreviated YAML for the Pi web application, which explicitly sets two replicas.

**Listing 6.2 web.yaml, a Deployment to run multiple replicas**

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pi-web
spec:
  replicas: 2 # The replicas field is optional; it defaults to 1.
selector:
  matchLabels:
    app: pi-web
  template: # The Pod spec follows.
```

The label selector for the Deployment needs to match the labels defined in the Pod template, and those labels are used to express the chain of ownership from Pod to ReplicaSet to Deployment. When you scale a Deployment, it updates the existing ReplicaSet to set the new number of replicas, but if you change the Pod spec in the Deployment, it replaces the ReplicaSet and scales the previous one down to zero. That gives the Deployment a lot of control over how it manages the update and how it deals with any problems.

**TRY IT NOW** Create a Deployment and Service for the Pi web application, and make some updates to see how the ReplicaSets are managed.

```
# deploy the Pi app:
kubectl apply -f pi/web/
# check the ReplicaSet:
kubectl get rs -l app=pi-web
# scale up to more replicas:
kubectl apply -f pi/web/update/web-replicas-3.yaml
# check the RS:
kubectl get rs -l app=pi-web
# deploy a changed Pod spec with enhanced logging:
kubectl apply -f pi/web/update/web-logging-level.yaml
# check ReplicaSets again:
kubectl get rs -l app=pi-web
```

This exercise shows that the ReplicaSet is still the scale mechanism: when you increase or decrease the number of replicas in your Deployment, it just updates the ReplicaSet. The Deployment is the, well, deployment mechanism, and it manages application updates through multiple ReplicaSets. My output, which appears in figure 6.7, shows how the Deployment waits for the new ReplicaSet to be fully operational before completely scaling down the old one.

You can use the kubectl scale command as a shortcut for scaling controllers. You should use it sparingly because it’s an imperative way to work, and it’s much better to use declarative YAML files, so that the state of your apps in production always exactly matches the spec stored in source control. But if your app is underperforming and the automated deployment takes 90 seconds, it’s a quick way to scale—as long as you remember to update the YAML file, too.

**TRY IT NOW** Scale up the Pi application using kubectl directly, and then see what happens with the ReplicaSets when another full deployment happens.

```
# we need to scale the Pi app fast:
kubectl scale --replicas=4 deploy/pi-web
# check which ReplicaSet makes the change:
kubectl get rs -l app=pi-web
# now we can revert back to the original logging level:
kubectl apply -f pi/web/update/web-replicas-3.yaml
# but that will undo the scale we set manually:
kubectl get rs -l app=pi-web
# check the Pods:
kubectl get pods -l app=pi-web
```

![图6.7](./images/Figure6.7.png)

<center>图 6.7 Deployments manage ReplicaSets to keep the desired number of Pods available during updates.**

You’ll see two things when you apply the updated YAML: the app scales back down to three replicas, and the Deployment does that by scaling the new ReplicaSet down to zero Pods and scaling the old ReplicaSet back up to three Pods. Figure 6.8 shows that the updated Deployment results in three new Pods being created.

It shouldn’t be a surprise that the Deployment update overwrote the manual scale level; the YAML definition is the desired state, and Kubernetes does not attempt to retain any part of the current spec if the two differ. It might be more of a surprise that the Deployment reused the old ReplicaSet instead of creating a new one, but that’s a more efficient way for Kubernetes to work, and it’s possible because of more labels.

Pods created from Deployments have a generated name that looks random but actually isn’t. The Pod name contains a hash of the template in the Pod spec for the Deployment, so if you make a change to the spec that matches a previous Deployment, then it will have the same template hash as a scaled-down ReplicaSet, and the Deployment can find that ReplicaSet and scale it up again to effect the change. The Pod template hash is stored in a label.

**TRY IT NOW** Check out the labels for the Pi Pods and ReplicaSets to see the template hash.

![图6.8](./images/Figure6.8.png)

<center>图 6.8 Deployments know the spec for their ReplicaSets and can roll back by scaling an old ReplicaSet.</center>

Pods created from Deployments have a generated name that looks random but actually isn’t. The Pod name contains a hash of the template in the Pod spec for the Deployment, so if you make a change to the spec that matches a previous Deployment, then it will have the same template hash as a scaled-down ReplicaSet, and the Deployment can find that ReplicaSet and scale it up again to effect the change. The Pod template hash is stored in a label.

**TRY IT NOW** Check out the labels for the Pi Pods and ReplicaSets to see the template hash.

```
# list ReplicaSets with labels:
kubectl get rs -l app=pi-web --show-labels
# list Pods with labels:
kubectl get po -l app=pi-web --show-labels
```

Figure 6.9 shows that the template hash is included in the object name, but this is just for convenience—Kubernetes uses the labels for management.

![图6.9](./images/Figure6.9.png)

<center>图 6.9 Object names generated by Kubernetes aren’t just random—they include the template hash.</center>

Knowing the internals of how a Deployment is related to its Pods will help you understand how changes are rolled out and clear up any confusion when you see lots of ReplicaSets with desired Pod counts of zero. But the interaction between the compute layer in the Pods and the network layer in the Services works in the same way.

In a typical distributed application, you’ll have different scale requirements for each component, and you’ll make use of Services to achieve multiple layers of load balancing between them. The Pi application we’ve deployed so far has only a ClusterIP Service—it’s not a public-facing component. The public component is a proxy (actually, it’s a reverse proxy because it handles incoming traffic rather than outgoing traffic), and that uses a LoadBalancer Service. We can run both the web component and the proxy at scale and achieve load balancing from the client to the proxy Pods and from the proxy to the application Pods.

**TRY IT NOW** Create the proxy Deployment, which runs with two replicas, along with a Service and ConfigMap, which sets up the integration with the Pi web app.

```
# deploy the proxy resources:
kubectl apply -f pi/proxy/
# get the URL to the proxied app:
kubectl get svc whoami-web -o jsonpath='http://{.status.loadBalancer.ingress[0].*}:8080/?dp=10000'
# browse to the app, and try a few different values for 'dp' in the URL
```

If you open the developer tools in your browser and look at the network requests, you can find the response headers sent by the proxy. These include the hostname of the proxy server—which is actually the Pod name—and the web page itself includes the name of the web application Pod that generated the response. My output, which appears in figure 6.10, shows a response that came from the proxy cache.

![图6.10](./images/Figure6.10.png)

<center>图 6.10 The Pi responses include the name of the Pod that sent them, so you can see the load balancing at work.</center>

This configuration is a simple one, which makes it easy to scale. The Pod spec for the proxy uses two volumes: a ConfigMap to load the proxy configuration file and an EmptyDir to store the cached responses. ConfigMaps are read-only, so one ConfigMap can be shared by all the proxy Pods. EmptyDir volumes are writable, but they’re unique to the Pod, so each proxy gets its own volume to use for cache files. Figure 6.11 shows the setup.

![图6.11](./images/Figure6.11.png)

<center>图 6.11 Running Pods at scale—some types of volume can be shared, whereas others are unique to the Pod.</center>

This architecture presents a problem, which you’ll see if you request Pi to a high number of decimal places and keep refreshing the browser. The first request will be slow because it is computed by the web app; subsequent responses will be fast, because they come from the proxy cache, but soon your request will go to a different proxy Pod that doesn’t have that response in its cache, so the page will load slowly again.

It would be nice to fix this by using shared storage, so every proxy Pod had access to the same cache. Doing so will bring us back to the tricky area of distributed storage that we thought we’d left behind in chapter 5, but let’s start with a simple approach and see where it gets us.

**TRY IT NOW** Deploy an update to the proxy spec, which uses a HostPath volume for cache files instead of an EmptyDir. Multiple Pods on the same node will use the same volume, which means they’ll have a shared proxy cache.

```
# deploy the updated spec:
kubectl apply -f pi/proxy/update/nginx-hostPath.yaml
# check the Pods—the new spec adds a third replica:
kubectl get po -l app=pi-proxy
# browse back to the Pi app, and refresh it a few times
# check the proxy logs:
kubectl logs -l app=pi-proxy --tail 1
```

Now you should be able to refresh away to your heart’s content, and responses will always come from the cache, no matter which proxy Pod you are directed to. Figure 6.12 shows all my proxy Pods responding to requests, which are shared between them by the Service.
For most stateful applications, this approach wouldn’t work. Apps that write data tend to assume they have exclusive access to the files, and if another instance of the same app tries to use the same file location, you’d get unexpected but disappointing results—like the app crashing or the data being corrupted. The reverse proxy I’m using is called Nginx; it’s unusually lenient here, and it will happily share its cache directory with other instances of itself.

If your apps need scale and storage, you have a couple of other options for using different types of controller. In the rest of this chapter, we’ll look at the DaemonSet; the final type is the StatefulSet, which gets complicated quickly, and we’ll come to it in chapter 8 where it gets most of the chapter to itself. DaemonSets and StatefulSets are both Pod controllers, and although you’ll use them a lot less frequently than Deployments, you need to know what you can do with them because they enable some powerful patterns.

![图6.12](./images/Figure6.12.png)

<center>图 6.12 At scale, you can see all the Pod logs with kubectl, using a label selector.</center>

## 6.3 使用 DaemonSets 实现高可用性

The DaemonSet takes its name from the Linux daemon, which is usually a system process that runs constantly as a single instance in the background (the equivalent of a Windows Service in the Windows world). In Kubernetes, the DaemonSet runs a single replica of a Pod on every node in the cluster, or on a subset of nodes, if you add a selector in the spec.
DaemonSets are common for infrastructure-level concerns, where you might want to grab information from every node and send it on to a central collector. A Pod runs on each node, grabbing just the data for that node. You don’t need to worry about any resource conflicts, because there will be only one Pod on the node. We’ll use DaemonSets later in this book to collect logs from Pods, and metrics about the node’s activity.

You can also use them in your own designs when you want high availability without the load requirements for many replicas on each node. A reverse proxy is a good example: a single Nginx Pod can handle many thousands of concurrent connections, so you don’t necessarily need a lot of them, but you may want to be sure there’s one running on every node, so a local Pod can respond wherever the traffic lands. Listing 6.3 shows the abbreviated YAML for a DaemonSet—it looks much like the other controllers but without the replica count.

**Listing 6.3 nginx-ds.yaml, a DaemonSet for the proxy component**

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: pi-proxy
spec:
  selector:
    matchLabels: # DaemonSets use the same label selector mechanism.
      app: pi-proxy # Finds the Pods that the set owns
template:
  metadata:
    labels:
      app: pi-proxy # Labels applied to the Pods must match the selector.
spec:
# Pod spec follows
```

This spec for the proxy still uses a HostPath volume. That means each Pod will have its own proxy cache, so we don’t get ultimate performance from a shared cache. This approach would work for other stateful apps, which are fussier than Nginx, because there’s no issue with multiple instances using the same data files.

**TRY IT NOW** You can’t convert from one type of controller to another, but we can make the change from Deployment to DaemonSet without breaking the app.

```
# deploy the DaemonSet:
kubectl apply -f pi/proxy/daemonset/nginx-ds.yaml
# check the endpoints used in the proxy service:
kubectl get endpoints pi-proxy
# delete the Deployment:
kubectl delete deploy pi-proxy
# check the DaemonSet:
kubectl get daemonset pi-proxy
# check the Pods:
kubectl get po -l app=pi-proxy
# refresh your latest Pi calculation on the browser
```

Figure 6.13 shows my output. Creating the DaemonSet before removing the Deployment means there are always Pods available to receive requests from the Service. Deleting the Deployment first would make the app unavailable until the DaemonSet started. If you check the HTTP response headers, you should also see that your request came from the proxy cache, because the new DaemonSet Pod uses the same HostPath volume as the Deployment Pods.

![图6.13](./images/Figure6.13.png)

<center>图 6.13 You need to plan the order of the deployment for a big change to keep your app online.</center>

I’m using a single-node cluster, so my DaemonSet runs a single Pod; with more nodes, I’d have one Pod on each node. The control loop watches for nodes joining the cluster, and any new nodes will be scheduled to start a replica Pod as soon as they join. The controller also watches the Pod status, so if a Pod is removed, then a replacement starts up.

**TRY IT NOW** Manually delete the proxy Pod. The DaemonSet will start a replacement.

```
# check the status of the DaemonSet:
kubectl get ds pi-proxy
# delete its Pod:
kubectl delete po -l app=pi-proxy
# check the Pods:
kubectl get po -l app=pi-proxy
```

If you refresh your browser while the Pod is being deleted, you’ll see it doesn’t respond until the DaemonSet has started a replacement. This is because you’re using a single-node lab cluster. Services send traffic only to running Pods, so in a multinode environment, the request would go to a node that still had a healthy Pod. Figure 6.14 shows my output.

![图6.14](./images/Figure6.14.png)

<center>图 6.14 DaemonSets watch nodes and Pods to ensure the desired replica count is always met.</center>

Situations where you need a DaemonSet are often a bit more nuanced than just wanting to run a Pod on every node. In this proxy example, your production cluster might have only a subset of nodes that can receive traffic from the internet, so you’d want to run proxy Pods only on those nodes. You can achieve that with labels, adding whatever arbitrary label you’d like to identify your nodes and then selecting that label in the Pod spec. Listing 6.4 shows this with a nodeSelector field.

**Listing 6.4 nginx-ds-nodeSelector.yaml, a DaemonSet with node selection**

```
# This is the Pod spec within the template field of the DaemonSet.
spec:
  containers:
    # ...
  volumes:
    # ...
  nodeSelector: # Pods will run only on certain nodes.
    kiamol: ch06 # Selected with the label kiamol=ch06
```

The DaemonSet controller doesn’t just watch to see nodes joining the cluster; it looks at all nodes to see if they match the requirements in the Pod spec. When you deploy this change, you’re telling the DaemonSet to run on only nodes that have the label kiamol set to the value of ch06. There will be no matching nodes in your cluster, so the DaemonSet will scale down to zero.

**TRY IT NOW** Update the DaemonSet to include the node selector from listing 6.4. Now there are no nodes that match the requirements, so the existing Pod will be removed. Then label a node, and a new Pod will be scheduled.

```
# update the DaemonSet spec:
kubectl apply -f pi/proxy/daemonset/nginx-ds-nodeSelector.yaml
# check the DS:
kubectl get ds pi-proxy
# check the Pods:
kubectl get po -l app=pi-proxy
# now label a node in your cluster so it matches the selector:
kubectl label node $(kubectl get nodes -o jsonpath='{.items[0].metadata.name}') kiamol=ch06 --overwrite
# check the Pods again:
kubectl get ds pi-proxy
```

You can see the control loop for the DaemonSet in action in figure 6.15. When the node selector is applied, no nodes meet the selector, so the desired replica count for the DaemonSet drops to zero. The existing Pod is one too many for the desired count, so it is removed. Then, when the node is labeled, there’s a match for the selector, and the desired count increases to one, so a new Pod is created.

![图6.15](./images/Figure6.15.png)

<center>图 6.15 DaemonSets watch nodes and their labels, as well as the current Pod status.</center>

DaemonSets have a different control loop from ReplicaSets because their logic needs to watch node activity as well as Pod counts, but fundamentally, they are both controllers that manage Pods. All controllers are responsible for the life cycle of their managed objects, but the links can be broken. We’ll use the DaemonSet in one more exercise to show how Pods can be set free from their controllers.

**TRY IT NOW** Kubectl has a cascade option on the delete command, which you can use to delete a controller without deleting its managed objects. Doing so leaves orphaned Pods behind, which can be adopted by another controller if they are a match for their previous owner.

```
# delete the DaemonSet, but leave the Pod alone:
kubectl delete ds pi-proxy --cascade=false
# check the Pod:
kubectl get po -l app=pi-proxy
# recreate the DS:
kubectl apply -f pi/proxy/daemonset/nginx-ds-nodeSelector.yaml
# check the DS and Pod:
kubectl get ds pi-proxy
kubectl get po -l app=pi-proxy
# delete the DS again, without the cascade option:
kubectl delete ds pi-proxy
# check the Pods:
kubectl get po -l app=pi-proxy
```

Figure 6.16 shows the same Pod survives through the DaemonSet being deleted and recreated. The new DaemonSet requires a single Pod, and the existing Pod is a match for its template, so it becomes the manager of the Pod. When this DaemonSet is deleted, the Pod is removed too.

Putting a halt on cascading deletes is one of those features you’re going to use rarely, but you’ll be very glad you knew about it when you do need it. In this scenario, you might be happy with all your existing Pods but have some maintenance tasks coming up on the nodes. Rather than have the DaemonSet adding and removing Pods while you work on the nodes, you could delete it and reinstate it after the maintenance is done.

The example we’ve used here for DaemonSets is about high availability, but it’s limited to certain types of application—where you want multiple instances and it’s acceptable for each instance to have its own independent data store. Other applications where you need high availability might need to keep data synchronized between instances, and for those, you can use StatefulSets. Don’t skip on to chapter 8 yet, though, because you’ll learn some neat patterns in chapter 7 that help with stateful apps, too.

![图6.16](./images/Figure6.16.png)

<center>图 6.16 Orphaned Pods have lost their controller, so they're not part of a highly available set anymore.</center>

StatefulSets, DaemonSets, ReplicaSets, and Deployments are the tools you use to model your apps, and they should give you enough flexibility to run pretty much any- thing in Kubernetes. We’ll finish this chapter with a quick look at how Kubernetes actually manages objects that own other objects, and then we’ll review how far we’ve come in this first section of the book.

## 6.4 理解 Kubernetes 中的对象所有权

Controllers use a label selector to find objects that they manage, and the objects themselves keep a record of their owner in a metadata field. When you delete a controller, its managed objects still exist but not for long. Kubernetes runs a garbage collector process that looks for objects whose owner has been deleted, and it deletes them, too. Object ownership can model a hierarchy: Pods are owned by ReplicaSets, and ReplicaSets are owned by Deployments.

**TRY IT NOW** Look at the owner reference in the metadata fields for all Pods and ReplicaSets.

```
# check which objects own the Pods:
kubectl get po -o custom-columns=NAME:'{.metadata.name}', OWNER:'{.metadata.ownerReferences[0].name}',OWNER_KIND:'{.metadata.ownerReferences[0].kind}'
# check which objects own the ReplicaSets:
kubectl get rs -o custom-columns=NAME:'{.metadata.name}', OWNER:'{.metadata.ownerReferences[0].name}',OWNER_KIND:'{.metadata.ownerReferences[0].kind}'
```

Figure 6.17 shows my output, where all of my Pods are owned by some other object, and all but one of my ReplicaSets are owned by a Deployment.

![图6.17](./images/Figure6.17.png)
<center>图 6.17 Objects know who their owners are—you can find this in the object metadata.</center>

Kubernetes does a good job of managing relationships, but you need to remember that controllers track their dependents using the label selector alone, so if you fiddle with labels, you could break that relationship. The default delete behavior is what you want most of the time, but you can stop cascading deletes using kubectl and delete only the controller—that removes the owner reference in the metadata for the dependents, so they don’t get picked up by the garbage collector.

We’re going to finish up with a look at the architecture for the latest version of the Pi app, which we’ve deployed in this chapter. Figure 6.18 shows it in all its glory.

![图6.18](./images/Figure6.18.png)
<center>图 6.18 The Pi application: no annotations necessary—the diagram should be crystal clear.</center>

Quite a lot is going on in this diagram: it’s a simple app, but the deployment is complex because it uses lots of Kubernetes features to get high availability, scale, and flexibility. By now you should be comfortable with all those Kubernetes resources, and you should understand how they fit together and when to use them. Around 150 lines of YAML define the application, but those YAML files are all you need to run this app on your laptop or on a 50-node cluster in the cloud. When someone new joins the project, if they have solid Kubernetes experience—or if they’ve read the first six chapters of this book—they can be productive straight away.
That’s all for the first section. My apologies if you had to take a few extended lunchtimes this week, but now you have all the fundamentals of Kubernetes, with best practices built in. All we need to do is tidy up before you attempt the lab.

**TRY IT NOW** All the top-level objects in this chapter had a kiamol label applied. Now that you understand cascading deletes, you’ll know that when you delete all those objects, all their dependents get deleted, too.

```
# remove all the controllers and Services:
kubectl delete all -l kiamol=ch06
```

## 6.5 Lab
Kubernetes has changed a lot over the last few years. The controllers we’ve used in this chapter are the recommended ones, but there have been alternatives in the past. Your job in this lab is to take an app spec that uses some older approaches and update it to use the controllers you’ve learned about.

- Start by deploying the app in ch06/lab/numbers—it’s the random-number app from chapter 3 but with a strange configuration. And it’s broken.
- You need to update the web component to use a controller that supports high load. We’ll want to run dozens of these in production.
- The API needs to be updated, too. It needs to be replicated for high availability, but the app uses a hardware random-number generator attached to the server, which can be used by only one Pod at a time. Nodes with the right hardware have the label rng=hw (you’ll need to simulate that in your cluster).
- This isn’t a clean upgrade, so you need to plan your deployment to make sure there’s no downtime for the web app.

Sounds scary, but you shouldn’t find this too bad. My solution is on GitHub for you to check: https://github.com/sixeyed/kiamol/blob/master/ch06/lab/README.md.
