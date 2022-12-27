# 第三章 通过 Service 网络连接 Pods

Pods 是 Kubernetes 中运行的应用程序的基本构建块。大多数应用是跨多组件分布式运行的，你通过在 kubernetes 中模式化pods 来对应每个组件。例如，你可能有一个 web 站点 Pod 以及一个 API Pod，或者你有一打微服务架构的 Pods。它们之间需要通信，kubernetes 支持标准的网络协议，tcp 以及 UDP，它们都通过 IP 地址来路由流量，但是当 Pods 被替换之后 IP 就变了，所以 Kubernetes 通过 Services 对象来提供网络地址发现机制。
 
Services 是支持 Pods 之间路由流量的灵活资源，可实现路由集群外的流量到 Pods 中，以及 Pods 到外部系统的流量。在本章，您将了解 Kubernetes 为将系统粘合在一起提供的所有不同 Service 配置, 你将会明白它们是如何透明地为你的应用程序工作的。

## 3.1 Kubernetes 如何路由网络流量

在前一章，你学到了两个关于 Pods 的重点：一个 Pod 一个拥有 Kubernetes 指定的 IP 地址的虚拟环境，以及 Pods 的生命周期是可以被其它类型资源自由控制的。如果一个 Pod 想要和其它 Pod 通信，可以使用 Ip 地址。然而这么干是有问题的，两个原因：第一，当 Pod 被替换时 Ip 会改变，第二，查找 Pod 的IP地址没有简单的方法，只能通过 Kubernetes API。

<b>现在就试试</b> 如果你部署了两个 Pods，你可以由其中一个 Ping 另外一个，前提是需要知道 IP 地址。

```
# 启动你的实验环境— 运行 Docker Desktop ，然后进入本章的源码目录:
cd ch03
# 创建两个 Deployments, 它们每一个都运行了一个 Pod:
kubectl apply -f sleep/sleep1.yaml -f sleep/sleep2.yaml
# 等待 Pod 到达 ready 状态:
kubectl wait --for=condition=Ready pod -l app=sleep-2
# 检查第二个 Pod 的 Ip 地址:
kubectl get pod -l app=sleep-2 --output 
    jsonpath='{.items[0].status.podIP}'
# 使用返回的 ip 地址，在 第一个 Pod ping 该 ip 地址:
kubectl exec deploy/sleep-1 -- ping -c 2 $(kubectl get pod -l app=sleep-2 
    --output jsonpath='{.items[0].status.podIP}')
```

我的输出如图 3.1 所示. 容器中的 Ping 工作的很好, 第一个 Pod 成功 ping 通第二个 Pod, 前提是我必须使用 kubectl 获取ip地址，并作为 ping命令的入参。

![图3.1  使用 IP 地址实现 Pod 网络通信—你可以使用 kubernetes API 获取 ip 地址.](./images/Figure3.1.png)

Kubernetes 中的虚拟网络覆盖整个集群，因此 Pods 即使在不同的节点上运行，也可以通过IP地址进行通信。此示例在单节点 K3s 集群和 100 节点AKS集群上的工作方式相同。这是一个有用的练习，可以帮助您了解Kubernetes没有任何特殊的网络魔力；它只是使用你的应用程序已经使用的标准协议。但是您通常不会这样做，因为IP地址是特定于一个Pod的，并且当Pod被替换时，替换将具有新的IP地址。

<b>现在就试试</b> 这些 Pods 由 Deployment 控制器管理。如果您删除了第二个Pod，它的控制器将开始使用新的IP地址进行 POD 替换。

```
# 检查当前 Pod 的 IP 地址:
kubectl get pod -l app=sleep-2 --output 
    jsonpath='{.items[0].status.podIP}'
# 删除 pod，deployment 将替换创建新的 pod:
kubectl delete pods -l app=sleep-2
# 检查替换产生的 POD ip:
kubectl get pod -l app=sleep-2 --output 
    jsonpath='{.items[0].status.podIP}'
```

在图 3.2, 我的输出显示了新的 POD 拥有了新的 IP 地址，如果你 ping 旧的地址，将会失败。

![图3.2 Pod IP 地址并不是其配置的一部分; 替换的 Pod 拥有新的地址.](./images/Figure3.2.png)

需要一个可以更改的资源的永久地址的问题是一个老问题，互联网使用DNS（域名系统）解决了这个问题，将友好名称映射到IP地址，Kubernetes使用相同的系统。Kubernetes集群内置了DNS服务器，它将服务名称映射到IP地址。图3.3显示了域名查找如何用于Pod-to-Pod通信。

![图3.3  Services 允许 Pods 使用固定的域名通信.](./images/Figure3.3.png)

这种类型的 Service 是对Pod及其网络地址的抽象，就像 Deployment 是对Pod及其容器的抽象一样。Service 有自己的IP地址，它是静态的。当消费者向该地址发出网络请求时，Kubernetes将其路由到Pod的实际IP地址。Service 和它的Pod之间的链接是用标签选择器设置的，就像Deployments和Pods之间的链接一样。

 清单 3.1 显示了 Service 的最小YAML规范，使用 app 标签标识Pod，Pod是网络流量的最终目标。

> 清单 3.1 sleep2-service.yaml, 最简单的 Service 定义
```
apiVersion: v1 
kind: Service
metadata:
  name: sleep-2 # Service 的名称用作DNS域名
# 该spec 配置需要一个选择器和一个端口列表。
spec:
  selector:
    app: sleep-2 # 匹配所有 app 标签值为 sleep-2 的 pods
  ports:
    - port: 80 # 监听端口 80 并发送到 Pod 上的端口80
```

此 Service 定义适用于我们在上一练习中运行的一个 Deployments。当您部署它时，Kubernetes会创建一个名为sleep-2的DNS条目，将流量路由到sleep-2 Deployment 创建的Pod中。其他Pod可以使用 Service 名称作为域名向该Pod发送流量。

<b>现在就试试</b> 使用 YAML 文件和通常的kubectl apply命令部署 Service，并验证网络流量是否路由到Pod。

```
# 部署清单 3.1 中的 Service :
kubectl apply -f sleep/sleep2-service.yaml
# 查看 service 基本信息:
kubectl get svc sleep-2
# 运行 ping 命令检查连通性—这将会失败:
kubectl exec deploy/sleep-1 -- ping -c 1 sleep-2
```
我的输出如图 3.4 所示, 你可以看到名称查找正常, ping命令没有按预期工作，因为ping使用的是Kubernetes Services不支持的网络协议。

![图3.4 部署一个 Service 以创建一个 DNS 入口, 为 Service 名称提供固定的IP地址.](./images/Figure3.4.png)

这是Kubernetes中服务发现背后的基本概念：部署Service 资源，并使用 Service 名称作为组件通信的域名。

不同类型的 Service 支持不同的网络模式，但您可以以相同的方式使用它们。接下来，我们将通过一个简单的分布式应用程序的工作示例，更深入地研究Pod到Pod的网络。

## 3.2 在 Pods 间路由流量

Kubernetes 中的默认 Service 类型称为ClusterIP。它创建了一个集群范围的IP地址，任何节点上的Pods都可以访问该地址。IP地址仅在集群内工作，因此ClusterIP Service 仅用于Pod之间的通信。这正是您想要的分布式系统，其中一些组件是内部的，不应该在集群外部访问。我们将使用一个使用内部API组件的简单网站来演示这一点。

<b>现在就试试</b> 运行两个Deployments，一个用于web应用程序，另一个用于API。此应用程序还没有 Service ，无法正常工作，因为 Web 网站找不到API。

```
# 部署两个分类的 deployment : 
kubectl apply -f numbers/api.yaml -f numbers/web.yaml
# 等待 Pod ready:
kubectl wait --for=condition=Ready pod -l app=numbers-web
# 转发端口到 web app:
kubectl port-forward deploy/numbers-web 8080:80
# 访问网站 http://localhost:8080 然后点击 Go 按钮
# —你将会看到错误消息
# 退出端口转发:
ctrl-c
```

您可以从图3.5所示的输出中看到，应用程序失败，并显示一条消息，说明API不可用。

![图3.5 web应用程序无法正常运行，因为对API的网络调用失败.](./images/Figure3.5.png)

错误页面还显示了站点希望在其中查找API http://numbers-api 的域名。这不是一个完全限定的域名（比如blog.sixeed.com）；这是一个应该由本地网络解析的地址，但Kubernetes中的DNS服务器无法解析它，因为没有名称为numbersapi的Service。清单3.2中的配置显示了一个名称正确的Service，以及一个与API Pod匹配的标签选择器。

> 清单 3.2  api-service.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: numbers-api # Service 使用域名 numbers-api.
spec:
  ports:
    - port: 80
  selector:
    app: numbers-api # 流量被路由到带有此标签的Pods.
  type: ClusterIP # 此Services 仅适用于其他Pod 通信.
```

此 Service 与清单3.1中的 Service 类似，只是名称已更改，并且明确说明了ClusterIP的服务类型。这可以省略，因为它是默认的服务类型，但我认为如果包含它，它会使配置更清晰，Service 将在web Pod和API Pod之间路由流量，在不更改 Deployment 或 Pod的情况下修复应用程序。

<b>现在就试试</b> 为 API 创建一个 Service，以便域查找可以正常工作，并将流量从web Pod发送到API Pod。

```
# 部署清单 3.2 中的 Service:
kubectl apply -f numbers/api-service.yaml
# 查看Service 信息:
kubectl get svc numbers-api
# 转发端口到 web app:
kubectl port-forward deploy/numbers-web 8080:80
# 访问 http://localhost:8080 然后点击 Go 按钮
# 退出端口转发:
ctrl-c
```
我的输出如图3.6所示，显示了应用程序正常工作，网站显示了API生成的随机数。

![图3.6 部署 Service 可修复web应用程序和API之间的断开链接.](./images/Figure3.6.png)

除了 Services、Deployments 和Pods之外，这里的重要教训是，YAML 规范描述了Kubernetes中的整个应用程序，包括所有组件和它们之间的网络。Kubernetes不会对应用程序架构进行假设；您需要在YAML中指定它。这个简单的web应用程序需要定义三个Kubernetes资源，以便在当前状态下工作—两个Deployments和一个Service—但拥有所有这些移动部件的优势是增加了弹性。

<b>现在就试试</b> API Pod由 Deployment 控制器管理，因此您可以删除Pod并创建替换。该替换也与API服务中的标签选择器相匹配，因此流量被路由到新的Pod，应用程序继续工作。

```
# 检查 API Pod 的名字和 ip 地址:
kubectl get pod -l app=numbers-api -o custom-columns=NAME:metadata.name,POD_IP:status.podIP
# 删除 Pod:
kubectl delete pod -l app=numbers-api
# 检查替换的 Pod:
kubectl get pod -l app=numbers-api -o custom-columns=NAME:metadata.name,POD_IP:status.podIP 
# 转发端口到 web app:
kubectl port-forward deploy/numbers-web 8080:80
# 访问 http://localhost:8080 然后点击 Go 按钮
# 退出端口转发:
ctrl-c
```

图3.7显示了 Deployment 控制器创建的替换Pod。它是相同的API Pod规范，但在具有新IP地址的新Pod中运行。不过，API Service的IP地址没有改变，web Pod可以在相同的网络地址到达新的API Pod。

![图3.7  该 Service 将web Pod与API Pod隔离，因此API Pod是否更改无关紧要.](./images/Figure3.7.png)

我们在这些练习中手动删除Pod，以触发控制器创建替换，但在Kubernetes应用程序的正常生命周期中，Pod替换总是发生。每当您更新应用程序的组件以添加功能、修复错误或发布对依赖项的更新时，您都将替换Pods。每当一个节点宕机时，它的Pod就会在其他节点上被替换。Service 抽象以通过这些替换保持应用程序的通信。

这个演示应用程序还没有完成，因为它没有任何配置来从集群外部接收流量并将其发送到web Pod。到目前为止，我们已经使用了端口转发，但这确实是一个调试技巧。真正的解决方案是为web Pod部署 Service。

## 3.3 路由外部流量到 Pods

您有几个选项来配置 Kubernetes 以监听进入集群的流量并将其转发到Pod。我们将从一种简单而灵活的方法开始。这是一种叫做LoadBalancer 类型的 Service，它解决了将流量发送到Pod的问题，Pod可能运行在与接收流量的节点不同的节点上；图3.8显示了它的工作方式。

![图3.8 LoadBalancer 类型 Service 将外部流量从任何节点路由到匹配的Pod.](./images/Figure3.8.png)

这看起来是一个棘手的问题，特别是因为您可能有许多Pod与 Service 的标签选择器匹配，所以集群需要选择一个节点来发送流量，然后在该节点上选择一个Pod。Kubernetes为您提供了世界级的编排，因此您需要做的就是部署LoadBalancer Service。清单3.3显示了web应用程序的 Service 配置。

> 清单 3.3 web-service.yaml, 一个 LoadBalancer Service 用于外部流量
```
apiVersion: v1
kind: Service
metadata:
  name: numbers-web
spec:
  ports:
    - port: 8080 # Service 监听的端口
      targetPort: 80 # 发送到的目标 POD 流量端口
  selector:
    app: numbers-web
  type: LoadBalancer # LoadBalancer 类型.
```

该 Service 在端口8080上侦听，并在端口 80 上向web Pod发送流量。当您部署它时，您将能够使用web应用程序，而无需在kubectl中设置端口转发，但如何访问该应用程序的确切细节将取决于您运行Kubernetes的方式。

<b>现在就试试</b> 部署 Service，然后使用 kubectl 查找 Service 的地址

```
# 为网站部署LoadBalancer Service:
kubectl apply -f numbers/web-service.yaml
# 查看 Service 信息:
kubectl get svc numbers-web
# 使用格式设置从 EXTERNAL-IP 字段获取应用程序URL:
kubectl get svc numbers-web -o
    jsonpath=’http://{.status.loadBalancer.ingress[0].*}:8080’
```
图3.9显示了我在Docker Desktop Kubernetes集群上运行该练习的结果，在这里我可以浏览到地址为 http://localhost:8080.

![图3.9 Kubernetes从其运行的平台请求LoadBalancer Services的IP地址](./images/Figure3.9.png)

使用K3s或云中的托管Kubernetes集群，输出是不同的，其中Service部署为负载平衡器创建了一个专用的外部IP地址。图3.10显示了在我的LinuxVM上使用K3s集群的相同练习（使用相同的YAML规范）的输出，网站位于http://172.28.132.127:8080.

![图3.10 Different Kubernetes platforms use different addresses for LoadBalancer Services](./images/Figure3.10.png)

对于相同的应用程序清单，结果如何不同？我在第1章中说过，您可以以不同的方式部署Kubernetes，这都是相同的Kubernete（我的重点），但这并不是绝对正确的。Kubernetes包含许多扩展点，发行版在如何实现某些特性方面具有灵活性。LoadBalancer Services是一个很好的例子，说明了实现的不同之处，适合于发行版的目标。

- Docker Desktop是一个本地开发环境。它在一台机器上运行，并与网络堆栈集成，因此LoadBalancer Service 在本地主机地址可用。每个LoadBalancer Service 都发布到localhost，因此如果部署许多 LoadBalancer Service，则需要使用不同的端口。
- K3s支持带有自定义组件的LoadBalancer Services，该组件在您的计算机上设置路由表。每个LoadBalancer Service 都发布到您的计算机（或VM）的IP地址，因此您可以使用本地主机或从网络上的远程计算机访问服务。像Docker Desktop一样，您需要为每个 load balancer 使用不同的端口。
- 像AKS和EKS这样的云Kubernetes平台是高度可用的多节点集群。部署Kubernetes LoadBalancer 服务会在云中创建一个实际的负载均衡器，它跨越集群中的所有节点。云负载均衡器将传入流量发送到其中一个节点，然后Kubernete将其路由到Pod。您将为每个LoadBalancer服务获得不同的IP地址，并且它将是一个公共地址，可从internet访问。

这是我们将在其他Kubernetes特性中再次看到的模式，其中发行版具有不同的可用资源和不同的目标。最终，YAML清单是相同的，最终结果是一致的，但Kubernetes允许发行版在到达目的地的方式上有所不同。

Back in the world of standard Kubernetes, there’s another Service type you can use that listens for network traffic coming into the cluster and directs it to a Pod—the NodePort. NodePort Services don’t require an external load balancer—every node in the cluster listens on the port specified in the Service and sends traffic to the target port on the Pod. Figure 3.11 shows how it works.、
回到标准Kubernetes的世界，您可以使用另一种 NodePort Service 类型来监听进入集群的网络流量，并将其引导到 Pod。NodePort 类型 Service 不需要外部负载均衡器，集群中的每个节点都侦听 Service 中指定的端口，并将流量发送到Pod上的目标端口。图3.11显示了它的工作原理。

![图3.11 NodePort 类型 Service 还将外部流量路由到Pod，但它们不需要负载均衡器.](./images/Figure3.11.png)

NodePort Services don’t have the flexibility of LoadBalancer Services because you need a different port for each Service, your nodes need to be publicly accessible,and you don’t achieve load-balancing across a multinode cluster. NodePort Services also have different levels of support in the distributions, so they work as expected in K3s and Docker Desktop but not so well in Kind. Listing 3.4 shows a NodePort spec for reference.
NodePort Services 没有 LoadBalancer Services 的灵活性，因为您需要为每个 Service 提供不同的端口，您的节点需要可公开访问，并且无法跨多节点集群实现负载均衡。NodePort Services在发行版中也有不同级别的支持，因此它们在K3s和Docker Desktop中的工作效果与预期一致，但在Kind中却不太好。清单3.4显示了一个NodePort规范以供参考。

> 清单 3.4 web-service-nodePort.yaml, NodePort 类型 Service 配置
```
apiVersion: v1
kind: Service
metadata:
  name: numbers-web-node
spec:
  ports:
    - port: 8080 # Service 侦听端口
      targetPort: 80 # 目标 Pod 端口
      nodePort: 30080 # 外部访问使用端口
  selector:
    app: numbers-web
  type: NodePort # Node 节点的 IP 可直接用于访问.
```

没有部署这个 NodePort Service 的练习（尽管如果您想尝试，YAML文件在章节的文件夹中）。这部分是因为它在每个发行版上的工作方式不同，因此本节将以许多if 分支结尾;你需要试着弄明白。但有一个更重要的原因是，您通常不会在生产中使用NodePort，而且最好在不同的环境中保持清单尽可能一致。坚持使用LoadBalancer 服务意味着从开发到生产都有相同的规范，这意味着需要维护和保持同步的YAML文件更少。

我们将通过深入了解 Service 在幕后的工作方式来完成本章，但在此之前，我们将再看一种使用服务的方式，即从Pods到集群外部组件的通信。

## 3.4 将流量路由到 Kubernetes 外面

您可以在 Kubernetes 中运行几乎任何服务器软件，但这并不意味着您应该这样做。像数据库这样的存储组件是在Kubernetes之外运行的典型候选组件，特别是如果您部署到云，并且可以使用托管数据库服务。或者您可能正在数据中心运行，需要与不会迁移到Kubernetes的现有系统集成。无论您使用的是什么架构，您仍然可以使用Kubernetes Services对集群外部的组件进行域名解析。

第一个选项是使用 ExternalName 类型 Service，它就像一个域到另一个域的别名。ExternalName Services允许您在应用程序Pod中使用本地名称，当Pod发出查找请求时，Kubernetes中的DNS服务器将本地名称解析为完全限定的外部名称。图3.12显示了如何工作，Pod使用解析为外部系统地址的本地名称。

![图3.12 使用ExternalName Service 可以为远程组件使用本地群集地址.](./images/Figure3.12.png)

本章的演示应用程序希望使用本地 API 生成随机数，但只需部署ExternalName服务，即可切换为从GitHub上的文本文件读取静态数。

<b>现在就试试</b> 在Kubernetes的每个版本中，您不能将服务从一种类型切换到另一种类型，因此在部署ExternalName服务之前，您需要删除API的原始ClusterIP服务。

```
# 删除当前的 API Service:
kubectl delete svc numbers-api
# 部署一个新的 ExternalName 类型 Service:
kubectl apply -f numbers-services/api-service-externalName.yaml
# 检查 Service 配置:
kubectl get svc numbers-api
# 刷新网站，通过 Go 按钮进行测试
```

我的输出如图3.13所示。您可以看到该应用程序以相同的方式工作，并且它使用相同的API URL。然而，如果您刷新页面，您会发现它总是返回相同的数字，因为它不再使用随机数API。

![图3.13 ExternalName Services可以用作重定向，以在集群外部发送请求.](./images/Figure3.13.png)

ExternalName Services是处理应用程序配置中无法解决的环境之间差异的有用方法。也许您有一个应用程序组件，它使用硬编码字符串作为数据库服务器的名称。在开发环境中，您可以使用预期的域名创建ClusterIP服务，该域名解析为在Pod中运行的测试数据库；在生产环境中，可以使用解析为数据库服务器的真实域名的ExternalName Service。清单3.5显示了API外部名称的YAML规范。

> 清单 3.5 api-service-externalName.yaml, 一个 ExternalName 类型 Service
```
apiVersion: v1
kind: Service
metadata:
  name: numbers-api # 集群中 Service 的本地域名
spec:
  type: ExternalName
  externalName: raw.githubusercontent.com # 要解析的域名
```
Kubernetes使用DNS规范名称（CNAME）的标准特性实现ExternalName Services。当web Pod对 numbers-api 域名进行DNS查找时，Kubernetes DNS服务器返回CNAME，即raw.githubusercontent.com。然后，DNS解析将继续使用节点上配置的DNS服务器，因此它将连接到互联网以查找IP-即GitHub服务器的地址。

<b>现在就试试</b> Services 是集群范围Kubernetes Pod网络的一部分，因此任何Pod都可以使用 Service。本章第一个练习中的sleep Pods在容器镜像中有一个DNS查找命令，您可以使用它来检查API Service。

```
# 运行DNS查找工具以解析 Service 名称:
kubectl exec deploy/sleep-1 -- sh -c ’nslookup numbers-api | tail -n 5’
```

当你尝试这样做时，你可能会得到看起来像错误的混乱结果，因为Nslokup工具会返回很多信息，而且每次运行它的顺序都不一样。不过你想要的数据就在那里。我重复了该命令几次，以获得如图3.14所示的适合打印输出的效果。

![图3.14 在Kubernetes中，应用程序默认不会被隔离，因此任何Pod都可以查找任何 Service.](./images/Figure3.14.png)

关于ExternalName Services，有一件重要的事情需要了解，您可以从本练习中看到：它们最终只是为应用程序提供一个使用地址，但实际上并不会改变应用程序发出的请求。这对于通过TCP通信的数据库等组件来说很好，但对于HTTP服务来说就不那么简单了。HTTP请求在头字段中包含目标主机名，这与ExternalName响应中的实际域不匹配，因此客户端调用可能会失败。本章中的随机数应用程序有一些黑客代码来解决这个问题，手动设置主机标头，但这种方法最适合非HTTP服务。

还有一个选项用于将集群中的本地域名路由到外部系统。它不能解决HTTP头问题，但当您希望路由到IP地址而不是域名时，它允许您使用与ExternalName Services类似的方法。他们是 headless 类型 Service，它们被定义为ClusterIP Service 类型，但没有设置标签选择器，因此它们永远不会匹配任何Pod。相反，Service 部署有一个端点资源，该资源明确列出了 Service 应该解析的IP地址。

清单3.6显示了一个 headless Service 以及拥有一个 ip地址的 endpoint 。它还显示了YAML的新用法，定义了多个资源，用三个破折号分隔。

> 清单 3.6 api-service-headless.yaml, 具有显式地址的 Service
```
apiVersion: v1
kind: Service
metadata:
  name: numbers-api
spec:
  type: ClusterIP # 没有选择器字段使其成为 headless Service.(另外可以指定 ClusterIP: None,来提供 headless service,可以具备 selector)
  ports:
    - port: 80
---
kind: Endpoints # Endpoints 是新的资源类型.
apiVersion: v1
metadata:
  name: numbers-api
subsets:
  - addresses: # 静态 ip 地址集合
    - ip: 192.168.123.234
    ports:
      - port: 80 # 以及它们监听的 端口.
```

该端点规范中的 IP地址是假的，但Kubernetes不会验证该地址是否可访问，因此该代码将无错误地部署。

<b>现在就试试</b>  使用此无头服务替换ExternalName服务。这将导致应用程序失败，因为API域名现在解析为无法访问的IP地址。
```
# 删除之前的 Service:
kubectl delete svc numbers-api
# 部署 headless Service:
kubectl apply -f numbers-services/api-service-headless.yaml
# 检查 Service:
kubectl get svc numbers-api
# 检查 endpoint:
kubectl get endpoints numbers-api
# 检查 DNS lookup:
kubectl exec deploy/sleep-1 -- sh -c ’nslookup numbers-api | grep
    "^[^*]"’
# 访问网站—当你尝试获取数字时返回失败
```

我的输出（如图3.15所示）证实了Kubernetes将很乐意让您部署一个破坏应用程序的服务变更。域名解析内部群集IP地址，但对该地址的任何网络调用都会失败，因为它们被路由到端点中不存在的实际IP地址。
![图3.15 Service 中的错误配置可能会破坏您的应用程序，即使不部署应用程序更改.](./images/Figure3.15.png)

该练习的输出提出了几个有趣的问题：DNS查找如何返回集群IP地址而不是端点地址？为什么域名以.default.svc.cluster.local结尾？使用Kubernetes服务不需要网络工程背景，但如果您了解服务解决方案的实际工作方式，这将有助于您跟踪问题，这就是我们将如何完成本章的内容。

## 3.5 理解 Kubernetes Service 解析

Kubernetes支持您的应用程序可能需要的所有网络配置，这些配置使用基于已建立的网络技术的服务。应用程序组件在Pod中运行，并使用标准传输协议和DNS名称与其他Pod进行通信以进行发现。您不需要任何特殊的代码或库；您的应用程序在Kubernetes中的工作方式与您在物理服务器或虚拟机上部署的方式相同。

在本章中，我们已经介绍了所有 Service 类型及其典型用例，因此现在您已经很好地了解了可以使用的模式。如果您觉得这里有太多的细节，请放心，大多数时候您都会部署 ClusterIP 类型服务，这几乎不需要配置。它们大部分都是无缝工作的，但深入一层来理解堆栈是很有用的。图3.16显示了下一层次的细节。

![图3.16 Kubernetes运行DNS服务器和代理，并将它们与标准网络工具一起使用。](./images/Figure3.16.png)

关键是 ClusterIP 是网络上不存在的虚拟IP地址。Pods通过运行在节点上的 kube-proxy 访问网络，并使用数据包过滤将虚拟IP发送到真实端点。Kubernetes services  只要它们存在，就保留它们的IP地址，并且 Service 可以独立于应用程序的任何其他部分而存在。Service 有一个控制器，每当Pods发生更改时，该控制器会更新端点列表，因此客户端始终使用静态虚拟IP地址，kube-proxy 始终拥有最新的端点列表。

<b>现在就试试</b> 您可以看到Kubernetes如何在Pod更改时通过列出Pod更改之间 Service 的端点来保持端点列表的即时更新。端点使用与 Service 相同的名称，您可以使用kubectl查看端点详细信息。
```
# 查看 sleep-2 Service 端点信息:
kubectl get endpoints sleep-2
# 删除 pod:
kubectl delete pods -l app=sleep-2
# 检查 端点使用了新的 POD ip 进行了更新:
kubectl get endpoints sleep-2
# 删除整个 Deployment:
kubectl delete deploy sleep-2
# 检查 endpoint 仍然存在, 没有 IP 地址清单:
kubectl get endpoints sleep-2
```

您可以在图3.17中看到我的输出，这是第一个问题的答案Kubernetes DNS返回Cluster IP地址，而不是端点，因为端点地址发生了变化。

![图3.17 Service 的Cluster IP地址不会更改，但端点列表始终在更新。.](./images/Figure3.17.png)

使用静态虚拟IP意味着客户端可以无限期地缓存DNS查找响应（许多应用程序这样做是错误的性能节省），并且无论一段时间内发生多少Pod替换，IP地址都将继续工作。关于域名后缀的第二个问题需要通过横向步骤来回答，以查看Kubernetes命名空间。

每个Kubernetes资源都位于一个命名空间中，这是一个可以用来对其他资源进行分组的资源。命名空间是对Kubernetes集群进行逻辑分区的一种方式，您可以为每个产品、每个团队或单个共享命名空间。我们暂时还不会使用名称空间，但我在这里介绍它们，因为它们在DNS解析中发挥作用。图3.18显示了命名空间在服务名称中的位置。

![图3.18 命名空间对集群进行逻辑分区，但 Service 可以跨命名空间访问.](./images/Figure3.18.png)

您的集群中已经有多个命名空间，到目前为止我们部署的所有资源都是在默认名称空间中创建的（称为 default；这就是为什么我们不需要在YAML文件中指定命名空间）。内部Kubernetes组件（如DNS服务器和KubernetesAPI）也在kube-system 命名空间的Pods中运行。

<b>现在就试试</b> Kurectl支持命名空间，您可以使用命名空间标志处理默认命名空间之外的资源。
```
# 在 default 命名空间检查 Service:
kubectl get svc --namespace default
# 检查 system 命名空间 Service :
kubectl get svc -n kube-system
# 尝试DNS查找完全限定的服务名称:
kubectl exec deploy/sleep-1 -- sh -c 'nslookup numbers-
  api.default.svc.cluster.local | grep "^[^*]"'
# 以及 kube-system 下 的 Service:
kubectl exec deploy/sleep-1 -- sh -c 'nslookup kube-dns.kube-
  system.svc.cluster.local | grep "^[^*]"'
```

我的输出如图3.19所示，回答了第二个问题：Service 的本地域名只是服务名称，但这是包含Kubernetes命名空间的完全限定域名的别名。

![图3.19 可以使用相同的kubectl命令查看不同命名空间中的资源.](./images/Figure3.19.png)

在Kubernetes之旅的早期了解命名空间很重要，因为它可以帮助您看到Kubernete的核心功能也可以作为Kubernets应用程序运行，但除非您明确设置了命名空间，否则在kubectl中看不到它们。命名空间是细分集群以提高利用率而不损害安全性的一种强大方式，我们将在第11章中再次讨论它们。

现在我们已经完成了命名空间和Service。在本章中，您已经了解到每个Pod都有自己的IP地址，Pod通信最终使用标准TCP和UDP协议的IP地址。您永远不会直接使用Pod IP地址，尽管您总是创建一个Service 资源，Kubernetes使用它通过DNS提供服务发现。服务支持多种网络模式，不同的服务类型配置Pod之间的网络流量，从外部世界进入Pod，以及从Pod到外部世界。您还了解到，服务有自己的生命周期，独立于Pod和Deployments，因此在我们继续之前，最后要做的就是清理。

<b>现在就试试</b> 删除 Deployment 也会删除其所有Pod，但 Service 没有级联删除。它们是需要单独删除的独立对象。
```
# 删除 Deployments:
kubectl delete deploy --all
# 删除 Services:
kubectl delete svc --all
# 检查还有什么在运行:
kubectl get all
```

现在集群又干净了，尽管如图3.20所示，您需要小心使用其中一些kubectl命令。

![图3.20 您需要显式删除创建的任何服务，但要注意all参数.](./images/Figure3.20.png)

## 3.6 实验室

This lab is going to give you some practice creating Services, but it’s also going to get you thinking about labels and selectors, which are powerful features of Kubernetes.The goal is to deploy Services for an updated version of the random-number app,which has had a UI makeover. Here are your hints:
这个实验室将为您提供一些创建 Service 的实践，但它也将让您思考标签和选择器，这是Kubernetes的强大功能。目标是为更新版本的随机数应用程序部署服务，该应用程序已经进行了UI改造。以下是您的提示：
- 本章的实验室文件夹包含deployments.yaml文件。通过它来使用kubectl部署应用程序。
- 检查Pods，有两个版本的web应用程序正在运行.
- 编写一个 Service，使API可用于域名 numbers-api w为其他Pod 服务.
- 在端口8088上编写一个 Service，使网站的版本2可以从外部访问。.
- 您需要仔细查看Pod标签以获得正确的结果.

这个实验室是本章练习的扩展，如果你想检查我的解决方案，它可以在GitHub上的仓库中找到：https://github.com/yyong-brs/learn-kubernetes/tree/master/kiamol/ch03/lab/README.md 。