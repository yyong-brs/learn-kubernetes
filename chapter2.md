# 第二章 Pods & Deployment 在 Kubernetes 中的应用

Kubernetes 通过容器来运行应用的工作负载，但是容器并不是在 Kubernetes 场景下你需要工作处理的对象。每个容器都属于某个叫做 Pod 的对象，它用于管理一个或多个容器，然后另外一个方面 Pods 被其它类型的资源所管理。这些高层级的对象抽象了具体的容器，它们提供了自我修复的程序并且提供了目标状态的工作流程：你告诉 Kubernetes 你想要实现的内容，然后 Kubernetes 来决定如何实现。
 
在本章中，我们将从 Kubernetes 基础的构建块开始：Pods 用于运行容器，以及 Deployments 用于管理 Pods。我们将使用一个简单的 Web app 进行练习，让你亲自动手使用 Kubernetes 命令行工具来管理应用并且使用 Kubernetes YAML 配置文件来定义应用程序。

## 2.1 Kubernetes 如何运行并管理容器

一个容器通常情况下作为虚拟环境来运行单个应用程序的组件。Kubernetes 将容器包装在另一个虚拟环境中： Pod。Pod 是一个计算单元，它在集群中的单个节点上运行。Pod 拥有受 Kubernetes 管理的的虚拟 IP 地址，并且就算Pods 运行在不同节点，它们之间也可以通过 Kubernetes 的虚拟网络进行通信。
 
正常情况下你只会在一个 Pod 中运行一个容器，但是你可以在一个 Pod 中运行多个容器，这将会开启一些有趣的部署选项。一个 Pod 中的所有容器将拥有相同的虚拟环境，所以它们共享相同的网络地址并且可以通过 localhost 通信。图 2.1 显示了容器与 Pods 之间的关系。

![图2.1](./images/Figure2.1.png)
<center>图2.1 容器在 Pod 内运行。你管理 Pods, Pods 管理容器 </center>

多容器 Pods 的业务在早期介绍有点多，但如果我掩盖它，只谈论单容器 Pods，你会理所当然地问：为什么 Kubernetes 使用 Pods 而不是容器。让我们运行一个Pod，然后看看使用容器上的这种抽象是什么样子的。

<b>现在就试试</b> 你可以通过 Kubernetes 命令行运行一个简单的 Pod(无需编写 YAML 文件)。语法类似于 Docker 运行容器: 你需要说明你要使用的容器的镜像以及任何用于配置 Pod 行为的其他参数。

```
# 运行一个单容器 Pod; restart 参数代表如果退出，不重启:
kubectl run hello-kiamol --image=kiamol/ch02-hello-kiamol --restart=Never
# 等待 Pod 就绪:
kubectl wait --for=condition=Ready pod hello-kiamol
# 查询所有的 Pod 清单:
kubectl get pods
# 查看 Pod 详细信息:
kubectl describe pod hello-kiamol
```

你可以看到图 2.2 是我执行时的输出，当你自己运行它时，你将在其中看到很多看起来晦涩难懂的信息，比如节点选择符和容错。他们都是 Pod 配置规范，Kubernetes 已将缺省值应用于我们没有在 Run 命令中指定的配置。

![图2.2](./images/Figure2.2.png)
<center>图2.2 运行最简单的 Pods 并且使用 Kubectl 检查它的状态 </center>

现在你的集群中拥有一个应用容器，它运行在一个 Pod 内。如果你之前接触过 Docker，这是一个熟悉的工作流程，事实证明 Pods 并不像看起来那么复杂。你的大多数 Pods 都会运行单个容器(直到您开始探索更高级的选项)，因此你可以有效地将 Pod 视为 Kubernetes 用来运行容器的一种机制。
 
Kubernetes 并不真正运行容器，它把这个责任交给节点上的容器运行时，可能会是 Docker 或者其他之类的。这就是为什么是抽象的：它是 Kubernetes 管理的资源，而容器则是 Kubernetes 之外的某个东西管理。你可以通过 Kubectl 来获取有关 Pod 的信息。

<b>现在就试试</b> Kubectl 从 get pod 命令返回基本的信息,你可以通过指定输出相关的参数来请求返回更多的内容。你可以说出要在输出参数中看到的各个字段,可以使用 JSONPath 查询语言或者 Go 模板来得到复杂的输出。

```
# 获取 Pod 基本信息:
kubectl get pod hello-kiamol
# 指定输出的自定义列, 选择了网络相关信息:
kubectl get pod hello-kiamol --output custom-
    columns=NAME:metadata.name,NODE_IP:status.hostIP,POD_IP:status.podIP 
# 指定查询 JSONPath ,
# 选择 Pod 中第一个容器的 ID:
kubectl get pod hello-kiamol -o 
    jsonpath='{.status.containerStatuses[0].containerID}'
```

我的输出如图 2.3 所示。我在 windows 上的 Docker Desktop 运行了一个单节点的 Kubernetes 集群，第二条命令中的节点 IP 是我的 Linux 虚机的 Ip 地址，然后 Pod IP 是集群中的 Pod 虚拟地址。第三条命令输出的 container ID 以容器运行时的名字作为前缀，我的是 Docker。

![图2.3](./images/Figure2.3.png)
<center>图2.3 Kubectl 有很多选项可以自定义各种对象包括 Pods 的输出 </center>

这可能会让人觉得很枯燥，但它有两个重要的要点。首先，kubectl 是一个非常强大的工具，作为你与 Kubernetes 的主要联系点，你将花费大量时间使用它，非常值得去理解它能做什么。查询命令的输出是查看您关心的信息的有用方法，因为您可以访问所有资源的详细信息，这对自动化也很有用。第二个要点是提醒 Kubernetes不运行容器，Pod 中的容器 ID 是引用自另一个运行容器的系统。

Pods 在创建时被分配给一个节点，该节点负责管理 Pod 及其容器，它通过使用容器运行时中的称为容器运行时接口（CRI）的已知API 来实现此操作。CRI 让节点以相同的方式为所有不同的容器运行时管理容器。它使用一个标准API来创建和删除容器并查询它们的状态。当 Pod 运行时，节点与容器运行时一起工作，以确保 Pod 有它需要的所有容器。

<b>现在就试试</b> 所有 Kubernetes 环境都使用相同的 CRI 机制来管理容器，但不是所有的容器运行时都允许您访问 Kubernetes 之外的容器。本练习向您展示Kubernetes 节点如何确保其 Pod 容器的运行，这里使用的是 Docker 容器运行时进行演示。

```
# 查询 Pod 中的容器:
docker container ls -q --filter 
    label=io.kubernetes.container.name=hello-kiamol
# 现在，删除容器:
docker container rm -f $(docker container ls -q --filter 
    label=io.kubernetes.container.name=hello-kiamol)
# 检查 Pod 状态:
kubectl get pod hello-kiamol
# 可以再次看到容器:
docker container ls -q --filter 
    label=io.kubernetes.container.name=hello-kiamol
```

从图 2.4 中可以看到，当我删除 Docker 容器时，Kubernetes做出了反应，一时间 Pod 没有了容器，但 Kubernetes 立即创建了一个替代容器来修复 Pod 并将其恢复到正确的状态。

![图2.4](./images/Figure2.4.png)
<center>图2.4 Kubernetes 确保 Pods 可以始终拥有它所需的容器 </center>

从容器到 Pods 的抽象可以让 Kubernetes 修复此类问题。一个失败的容器可能是临时的故障；Pod 仍然存在，并且 Pod 可以用一个新的容器使其符合目标。这只是 Kubernetes 提供的自我修复的一个级别，Kubernetes 在 Pods 之上提供了进一步的抽象，使你的应用程序更具弹性。

其中一个抽象是 Deployment，我们将在下一节中讨论。在我们继续之前，让我们看看Pod中到底在运行什么。这是一个 web 应用程序，但你还不能访问它，因为我们还没有配置 Kubernetes 来路由网络到 Pod 的流量。我们可以使用 kubectl 的另一个特性来解决这个问题。

<b>现在就试试</b> Kubectl 可以将流量从节点转发到Pod，这是一个从集群外部与 Pod 通信的快速方式。你可以监听在计算机上的特定端口，该端口是集群中的单个节点上的——并将流量转发到 Pod 中运行的应用程序。

```
# 监听你机器的 8080 端口，并将流量发送到 Pod 的 80 端口
kubectl port-forward pod/hello-kiamol 8080:80
# 现在访问 http://localhost:8080
# 当你结束之后，可以通过 ctrl-c 结束端口转发
```

我的输出如图 2.5 所示，你可以发现它是一个非常基本的 web 网站。这个 Web服务器和所有的内容都已打包到 Docker Hub 上的容器镜像中，该镜像是公开可用的。所有CRI-兼容的容器运行时可以提取镜像并从中运行容器，因此当您运行应用程序是，它对你的工作方式与对我的工作方式是相同的。
 
![图2.5](./images/Figure2.5.png)
<center>图 2.5 这个应用并没有配置接收网络流量，但是 Kubectl 可以转发网络流量 </center>

现在我们很好地了解了Pod，它是Kubernetes 中最小的计算单元。你需要了解这一切是如何运作的，但Pod是一个原始的资源，在正常使用中，您永远不会直接运行Pod；您总是创建一个控制器对象来管理 Pod。

## 2.2 通过控制器运行 Pods

It’s only the second section of the second chapter, and we’re already on to a new
Kubernetes object, which is an abstraction over other objects. Kubernetes does get
complicated quickly, but that complexity is a necessary part of such a powerful and
configurable system. The learning curve is the entrance fee for access to a world-class
container platform. 
 
 Pods are too simple to be useful on their own; they are isolated instances of an
application, and each Pod is allocated to one node. If that node goes offline, the Pod
is lost, and Kubernetes does not replace it. You could try to get high availability by run-
ning several Pods, but there’s no guarantee Kubernetes won’t run them all on the
same node. Even if you do get Pods spread across several nodes, you need to man-
age them yourself. Why do that when you have an orchestrator that can manage
them for you?
 
 That’s where controllers come in. A controller is a Kubernetes resource that man-
ages other resources. It works with the Kubernetes API to watch the current state of
the system, compares that to the desired state of its resources, and makes any changes
necessary. Kubernetes has many controllers, but the main one for managing Pods is
the Deployment, which solves the problems I’ve just described. If a node goes offline
and you lose a Pod, the Deployment creates a replacement Pod on another node; if
you want to scale your Deployment, you can specify how many Pods you want, and the
Deployment controller runs them across many nodes. Figure 2.6 shows the relation-
ship between Deployments, Pods, and containers.
 
![图2.6](./images/Figure2.6.png)
<center>图2.6 Deployment controllers manage Pods, and Pods manage containers</center>

You can create Deployment resources with kubectl, specifying the container image
you want to run and any other configuration for the Pod. Kubernetes creates the
Deployment, and the Deployment creates the Pod. 

TRY IT NOW Create another instance of the web application, this time using a
Deployment. The only required parameters are the name for the Deployment
and the image to run.
```
# create a Deployment called "hello-kiamol-2", running the same web app:
kubectl create deployment hello-kiamol-2 --image=kiamol/ch02-hello-kiamol
# list all the Pods:
kubectl get pods
```

You can see my output in figure 2.7. Now you have two Pods in your cluster: the origi-
nal one you created with the kubectl run command, and the new one created by the
Deployment. The Deployment-managed Pod has a name generated by Kubernetes,
which is the name of the Deployment followed by a random suffix.
 
![图2.7](./images/Figure2.7.png)
<center>图2.7 Create a controller resource, and it creates its own resources—Deployments create Pods</center>

One important thing to realize from this exercise: you created the Deployment, but
you did not directly create the Pod. The Deployment specification described the
Pod you wanted, and the Deployment created that Pod. The Deployment is a control-
ler that checks with the Kubernetes API to see which resources are running, realizes
the Pod it should be managing doesn’t exist, and uses the Kubernetes API to create it.
The exact mechanism doesn’t really matter; you can just work with the Deployment
and rely on it to create your Pod.
 
 How the deployment keeps track of its resources does matter, though, because it’s
a pattern that Kubernetes uses a lot. Any Kubernetes resource can have labels applied
that are simple key-value pairs. You can add labels to record your own data. For example,
you might add a label to a Deployment with the name release and the value 20.04 to
indicate this Deployment is from the 20.04 release cycle. Kubernetes also uses labels to
loosely couple resources, mapping the relationship between objects like a Deploy-
ment and its Pods.

TRY IT NOW The Deployment adds labels to Pods it manages. Use kubectl as
follows to print the labels the Deployment adds, and then list the Pods that
match that label:
```
# print the labels that the Deployment adds to the Pod:
kubectl get deploy hello-kiamol-2 -o 
    jsonpath='{.spec.template.metadata.labels}'
# list the Pods that have that matching label:
kubectl get pods -l app=hello-kiamol-2
```

My output is shown in figure 2.8, where you can see some internals of how the
resources are configured. Deployments use a template to create Pods, and part of that
template is a metadata field, which includes the labels for the Pod(s). In this case, the
Deployment adds a label called app with the value hello-kiamol-2 to the Pod. Querying
Pods that have a matching label returns the single Pod managed by the Deployment.
 
![图2.8](./images/Figure2.8.png)
<center>图2.8 Deployments add labels when they create Pods, and you can use those labels as filters.</center>

Using labels to identify the relationship between resources is such a core pattern in
Kubernetes that it’s worth showing a diagram to make sure it’s clear. Resources can
have labels applied at creation and then added, removed, or edited during their life-
time. Controllers use a label selector to identify the resources they manage. That can
be a simple query matching resources with a particular label, as shown in figure 2.9.
 
![图2.9](./images/Figure2.9.png)
<center>图2.9 Controllers identify the resources they manage by using labels and selectors.</center>

This process is flexible because it means controllers don’t need to maintain a list of all
the resources they manage; the label selector is part of the controller specification,
and controllers can find matching resources at any time by querying the Kubernetes
API. It’s also something you need to be careful with, because you can edit the labels
for a resource and end up breaking the relationship between it and its controller.

TRY IT NOW The Deployment doesn’t have a direct relationship with the Pod
it created; it only knows there needs to be one Pod with labels that match its
label selector. If you edit the labels on the Pod, the Deployment no longer
recognizes it.
```
# list all Pods, showing the Pod name and labels:
kubectl get pods -o custom-
    columns=NAME:metadata.name,LABELS:metadata.labels
# update the "app" label for the Deployment’s Pod:
kubectl label pods -l app=hello-kiamol-2 --overwrite app=hello-kiamol-x
# fetch Pods again:
kubectl get pods -o custom-
    columns=NAME:metadata.name,LABELS:metadata.labels
```

What did you expect to happen? You can see from the output shown in figure 2.10
that changing the Pod label effectively removes the Pod from the Deployment. At that
point, the Deployment sees that no Pods that match its label selector exist, so it cre-
ates a new one. The Deployment has done its job, but by editing the Pod directly, you
now have an unmanaged Pod.
 
![图2.10](./images/Figure2.10.png)
<center>图2.10 If you meddle with the labels on a Pod, you can remove it from the control of the Deployment.</center>

This can be a useful technique in debugging—removing a Pod from a controller so
you can connect and investigate a problem, while the controller starts a replacement
Pod, which keeps your app running at the desired scale. You can also do the opposite:
editing the labels on a Pod to fool a controller into acquiring that Pod as part of the
set it manages.
TRY IT NOW Return the original Pod to the control of the Deployment by set-
ting its app label back so it matches the label selector.
```
# list all Pods with a label called "app," showing the Pod name and
# labels:
kubectl get pods -l app -o custom-
    columns=NAME:metadata.name,LABELS:metadata.labels
# update the "app" label for the the unmanaged Pod:
kubectl label pods -l app=hello-kiamol-x --overwrite app=hello-kiamol-2
# fetch the Pods again:
kubectl get pods -l app -o custom-
    columns=NAME:metadata.name,LABELS:metadata.labels
```
This exercise effectively reverses the previous exercise, setting the app label back to
hello-kiamol-2 for the original Pod in the Deployment. Now when the Deployment
controller checks with the API, it sees two Pods that match its label selector. It’s sup-
posed to manage only a single Pod, however, so it deletes one (using a set of deletion
rules to decide which one). You can see in figure 2.11 that the Deployment removed
the second Pod and retained the original.
now have an unmanaged Pod.
 
![图2.11](./images/Figure2.11.png)
<center>图2.11 More label meddling—you can force a Deployment to adopt a Pod if the labels match.</center>

Pods run your application containers, but just like containers, Pods are meant to be
short-lived. You will usually use a higher-level resource like a Deployment to manage
Pods for you. Doing so gives Kubernetes a better chance of keeping your app running
if there are issues with containers or nodes, but ultimately the Pods are running the
same containers you would run yourself, and the end-user experience for your apps
will be the same.

TRY IT NOW Kubectl’s port-forward command sends traffic to a Pod, but you
don’t have to find the random Pod name for a Deployment. You can config-
ure the port forward on the Deployment resource, and the Deployment
selects one of its Pods as the target.
```
# run a port forward from your local machine to the Deployment:
kubectl port-forward deploy/hello-kiamol-2 8080:80
# browse to http://localhost:8080
# when you’re done, exit with ctrl-c
```

You can see my output, shown in figure 2.12, of the same app running in a container
from the same Docker image, but this time, in a Pod managed by a Deployment.
 
![图2.12](./images/Figure2.12.png)
<center>图2.12 Pods and Deployments are layers on top of containers, but the app still runs in a container.</center>

Pods and Deployments are the only resources we’ll cover in this chapter. You can
deploy very simple apps by using the kubectl run and create commands, but more
complex apps need lots more configuration, and those commands won’t do. It’s time
to enter the world of Kubernetes YAML.

## 2.3 在清单文件中定义 Deployments

Application manifests are one of the most attractive aspects of Kubernetes, but also
one of the most frustrating. When you’re wading through hundreds of lines of YAML
trying to find the small misconfiguration that has broken your app, it can seem like
the API was deliberately written to confuse and irritate you. At those times, remember
that Kubernetes manifests are a complete description of your app, which can be ver-
sioned and tracked in source control, and result in the same deployment on any
Kubernetes cluster.
 
 Manifests can be written in JSON or YAML; JSON is the native language of the
Kubernetes API, but YAML is preferred for manifests because it’s easier to read, lets you
define multiple resources in a single file, and, most important, can record comments in
the specification. Listing 2.1 is the simplest app manifest you can write. It defines a sin-
gle Pod using the same container image we’ve already used in this chapter.

> Listing 2.1 pod.yaml, a single Pod to run a single container
```
# Manifests always specify the version of the Kubernetes API
# and the type of resource.
apiVersion: v1
kind: Pod
# Metadata for the resource includes the name (mandatory) 
# and labels (optional).
metadata:
  name: hello-kiamol-3
# The spec is the actual specification for the resource.
# For a Pod the minimum is the container(s) to run, 
# with the container name and image.
spec:
  containers:
   - name: web
     image: kiamol/ch02-hello-kiamol
 ```

That’s a lot more information than you need for a kubectl run command, but the big
advantage of the application manifest is that it’s declarative. Kubectl run and create
are imperative operations—it’s you telling Kubernetes to do something. Manifests are
declarative—you tell Kubernetes what you want the end result to be, and it goes off
and decides what it needs to do to make that happen.

TRY IT NOW You still use kubectl to deploy apps from manifest files, but you
use the apply command, which tells Kubernetes to apply the configuration in
the file to the cluster. Run another pod for this chapter’s sample app using a
YAML file with the same contents as listing 2.1.
```
# switch from the root of the kiamol repository to the chapter 2 folder:
cd ch02
# deploy the application from the manifest file:
kubectl apply -f pod.yaml
# list running Pods:
kubectl get pods
```

The new Pod works in the same way as a Pod created with the kubectl run command:
it’s allocated to a node, and it runs a container. The output in figure 2.13 shows that
when I applied the manifest, Kubernetes decided it needed to create a Pod to get the
current state of the cluster up to my desired state. That’s because the manifest speci-
fies a Pod named hello-kiamol-3, and no such Pod existed.
 
![图2.13](./images/Figure2.13.png)
<center>图2.13 Applying a manifest sends the YAML file to the Kubernetes API, which  applies changes.</center>

Now that the Pod is running, you can manage it in the same way with kubectl: by list-
ing the details of the Pod and running a port forward to send traffic to the Pod. The
big difference is that the manifest is easy to share, and manifest-based Deployment is
repeatable. I can run the same kubectl apply command with the same manifest any
number of times, and the result will always be the same: a Pod named hello-kiamol-3
running my web container.

TRY IT NOW Kubectl doesn’t even need a local copy of a manifest file. It can
read the contents from any public URL. Deploy the same Pod definition
direct from the file on GitHub.
```
# deploy the application from the manifest file:
kubectl apply -f https://raw.githubusercontent.com/sixeyed/kiamol/
    master/ch02/pod.yaml
```

Figure 2.14 shows the output. The resource definition matches the Pod running in
the cluster, so Kubernetes doesn’t need to do anything, and kubectl shows that the
matching resource is unchanged.
 
![图2.14](./images/Figure2.14.png)
<center>图2.14 Kubectl can download manifest files from  a web server and send them  to the Kubernetes API.</center>
 
 Application manifests start to get more interesting when you work with higher-level
resources. When you define a Deployment in a YAML file, one of the required fields is
the specification of the Pod that the Deployment should run. That Pod specification is
the same API for defining a Pod on its own, so the Deployment definition is a compos-
ite that includes the Pod spec. Listing 2.2 shows the minimal definition for a Deploy-
ment resource, running yet another version of the same web app.

> Listing 2.2 deployment.yaml, a Deployment and Pod specification

```
# Deployments are part of the apps version 1 API spec.
apiVersion: apps/v1
kind: Deployment
# The Deployment needs a name.
metadata:
  name: hello-kiamol-4
# The spec includes the label selector the Deployment uses 
# to find its own managed resources—I’m using the app label,
# but this could be any combination of key-value pairs.
spec:
  selector:
    matchLabels:
      app: hello-kiamol-4
    # The template is used when the Deployment creates a Pod template.
    # Pods in a Deployment don’t have a name, 
    # but they need to specify labels that match the selector
    # metadata.
      labels:
        app: hello-kiamol-4
  # The Pod spec lists the container name and image spec.
  containers:
    - name: web
      image: kiamol/ch02-hello-kiamol
```

This manifest is for a completely different resource (which just happens to run the
same application), but all Kubernetes manifests are deployed in the same way using
kubectl apply. That gives you a nice layer of consistency across all your apps—no mat-
ter how complex they are, you’ll define them in one or more YAML files and deploy
them using the same kubectl command.

TRY IT NOW Apply the Deployment manifest to create a new Deployment,
which in turn will create a new Pod.
```
# run the app using the Deployment manifest:
kubectl apply -f deployment.yaml
# find Pods managed by the new Deployment:
kubectl get pods -l app=hello-kiamol-4
```

The output in figure 2.15 shows the same end result as creating a Deployment with
kubectl create, but my whole app specification is clearly defined in a single YAML file.
 
![图2.15](./images/Figure2.15.png)
<center>图2.15 Applying a manifest creates the Deployment because no matching resource existed.</center>

As the app grows in complexity, I need to specify how many replicas I want, what CPU
and memory limits should apply, how Kubernetes can check whether the app is healthy,
and where the application configuration settings come from and where it writes data—
I can do all that just by adding to the YAML. 

## 2.4 应用在 Pods 中运行

## 2.5 了解 Kubernetes 资源管理

## 2.6 实验室