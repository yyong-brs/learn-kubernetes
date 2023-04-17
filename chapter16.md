# 第十六章 使用策略上下文和准入控制保护应用程序

Containers are a lightweight wrapper around application processes. They start quickly and add little overhead to your app because they use the operating system kernel of the machine on which they’re running. That makes them super efficient, but at the cost of strong isolation—containers can be compromised, and a compromised container could provide unrestricted access to the server and to all the other containers running on it. Kubernetes has many features to secure your applications, but none of them are enabled by default. In this chapter, you’ll learn how to use the security controls in Kubernetes and how to set up your cluster so those controls are required for all your workloads.

容器是围绕应用程序进程的轻量级包装器。它们启动迅速并且几乎不会增加应用程序的开销，因为它们使用运行它们的机器的操作系统内核。这使它们非常高效，但代价是强隔离——容器可能会受到损害，而受到损害的容器可能会提供对服务器及其上运行的所有其他容器的不受限制的访问。 Kubernetes 有许多功能可以保护您的应用程序，但默认情况下都没有启用。在本章中，您将学习如何使用 Kubernetes 中的安全控制以及如何设置您的集群，以便您的所有工作负载都需要这些控制。

Securing applications in Kubernetes is about limiting what containers can do, so if an attacker exploits an app vulnerability to run commands in the container, they can’t get beyond that container. We can do this by restricting network access to other containers and the Kubernetes API, restricting mounts of the host’s filesystem, and limiting the operating system features the container can use. We’ll cover the essential approaches, but the security space is large and evolving. This chapter is even longer than the others—you’re about to learn a lot, but it will be only the start of your journey to a secure Kubernetes environment.
保护 Kubernetes 中的应用程序就是限制容器可以执行的操作，因此如果攻击者利用应用程序漏洞在容器中运行命令，他们将无法越过该容器。我们可以通过限制对其他容器和 Kubernetes API 的网络访问、限制主机文件系统的挂载以及限制容器可以使用的操作系统功能来做到这一点。我们将介绍基本方法，但安全空间很大且不断发展。本章比其他章节更长——您将学到很多东西，但这只是您通往安全 Kubernetes 环境之旅的开始。

## 16.1 使用网络策略(network policies)保护通信

Restricting network access is one of the simplest ways to secure your applications. Kubernetes has a flat networking model, where every Pod can reach every other Pod by its IP address, and Services are accessible throughout the cluster. There’s no reason why the Pi web application should access the to-do list database, or why the Hello, World web app should use the Kubernetes API, but by default, they can. You learned in chapter 15 how you can use Ingress resources to control access to HTTP routes, but that applies only to external traffic coming into the cluster. You also need to control access within the cluster, and for that, Kubernetes offers network policies.

限制网络访问是保护应用程序的最简单方法之一。 Kubernetes 有一个扁平的网络模型，其中每个 Pod 都可以通过其 IP 地址访问每个其他 Pod，并且服务可以在整个集群中访问。 Pi web 应用程序没有理由访问待办事项列表数据库，或者 Hello, World web 应用程序为什么应该使用 Kubernetes API，但默认情况下，它们可以。您在第 15 章中学习了如何使用 Ingress 资源来控制对 HTTP 路由的访问，但这仅适用于进入集群的外部流量。您还需要控制集群内的访问，为此，Kubernetes 提供了网络策略。

Network policies work like firewall rules, blocking traffic to or from Pods at the port level. The rules are flexible and use label selectors to identify objects. You can deploy a blanket deny-all policy to stop outgoing traffic from all Pods, or you can deploy a policy that restricts incoming traffic to a Pod’s metrics port so it can be accessed only from Pods in the monitoring namespace. Figure 16.1 shows how that looks in the cluster.
网络策略像防火墙规则一样工作，在端口级别阻止进出 Pod 的流量。规则很灵活，使用标签选择器来识别对象。您可以部署一揽子拒绝所有策略来阻止所有 Pod 的传出流量，或者您可以部署一个策略来限制传入流量到 Pod 的指标端口，以便只能从监控命名空间中的 Pod 访问它。图 16.1 显示了它在集群中的样子。

![图16.1](./images/Figure16.1.png)
<center>图 16.1 Network policy rules are granular—you can apply clusterwide defaults with Pod overrides.网络策略规则是细粒度的——您可以应用集群范围内的默认设置和 Pod 覆盖。 </center>

NetworkPolicy objects are separate resources, which means they could be modeled outside of the application by a security team, or they could be built by the product team. Or, of course, each team might think the other team has it covered, and apps make it to production without any policies, which is a problem. We’ll deploy an app that has slipped through with no policies and look at the problems it has.

NetworkPolicy 对象是独立的资源，这意味着它们可以由安全团队在应用程序外部建模，也可以由产品团队构建。或者，当然，每个团队都可能认为另一个团队已经涵盖了它，并且应用程序在没有任何策略的情况下投入生产，这是一个问题。我们将部署一个没有策略的应用程序，并查看它存在的问题。

TRY IT NOW
Deploy the Astronomy Picture of the Day (APOD) app, and confirm that the app components can be accessed by any Pod.
现在就试试，部署 Astronomy Picture of the Day (APOD) 应用程序，并确认任何 Pod 都可以访问该应用程序组件。

   ```
   # switch to the chapter folder:
   cd ch16
   # deploy the APOD app:
   kubectl apply -f apod/
   # wait for it to be ready:
   kubectl wait --for=condition=ContainersReady pod -l app=apod-web
   # browse to the Service on port 8016 if you want to see today’s
   # picture
   # now run a sleep Pod:
   kubectl apply -f sleep.yaml
   # confirm that the sleep Pod can use the API:
   kubectl exec deploy/sleep -- curl -s http://apod-api/image
   # read the metrics from the access log:
   kubectl exec deploy/sleep -- sh -c 'curl -s http://apod-log/metrics |
   head -n 2'
   ```

You can clearly see the issue in this exercise—the whole of the cluster is wide open, so from the sleep Pod, you can access the APOD API and the metrics from the access log component Figure 16.2 shows my output. Let’s be clear that there’s nothing special about the sleep Pod; it’s just a simple way to demonstrate the problem. Any container in the cluster can do the same.
你可以清楚地看到这个练习中的问题——整个集群是完全开放的，所以你可以从睡眠 Pod 访问 APOD API 和来自访问日志组件的指标图 16.2 显示了我的输出。让我们明确一点，没有什么特别的关于睡眠舱；这只是演示问题的一种简单方法。集群中的任何容器都可以做同样的事情。

![图16.2](./images/Figure16.2.png)
<center>图 16.2 The downside of Kubernetes’s flat networking model is that every Pod is accessible.  Kubernetes 扁平网络模型的缺点是每个 Pod 都是可访问的。</center>



Pods should be isolated so they receive traffic only from the components that need to access them, and they send traffic only to components they need to access. Network policies model that with ingress rules (don’t confuse them with Ingress resources), which restrict incoming traffic, and egress rules, which restrict outgoing traffic. In the APOD app, the only component that should have access to the API is the web app. Listing 16.1 shows this as an ingress rule in a NetworkPolicy object.
Pod 应该是隔离的，以便它们只从需要访问它们的组件接收流量，并且它们只将流量发送到它们需要访问的组件。网络策略使用限制传入流量的入口规则（不要将它们与入口资源混淆）和限制传出流量的出口规则建模。在 APOD 应用程序中，唯一应该有权访问 API 的组件是 Web 应用程序。清单 16.1 将其显示为 NetworkPolicy 对象中的入口规则。

> Listing 16.1 networkpolicy-api.yaml, restricting access to Pods by their labels 清单 16.1 networkpolicy-api.yaml，通过标签限制对 Pod 的访问

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
name: apod-api
spec:
podSelector: # This is the Pod where the rule applies.
matchLabels:
app: apod-api
ingress: # Rules default to deny, so this rule
- from: # denies all ingress except where the
- podSelector: # source of the traffic is a Pod with
matchLabels: # the apod-web label.
app: apod-web
ports: # This restriction is by port.
- port: api # The port is named in the API Pod spec.
```

The NetworkPolicy spec is fairly straightforward, and rules can be deployed in advance of the application so it’s secure as soon as the Pods start. Ingress and egress rules follow the same pattern, and both can use namespace selectors as well as Pod selectors. You can create global rules and then override them with more fine-grained rules at the application level.

NetworkPolicy 规范相当简单，并且可以在应用程序之前部署规则，因此一旦 Pod 启动它就是安全的。入口和出口规则遵循相同的模式，并且都可以使用命名空间选择器和 Pod 选择器。您可以创建全局规则，然后在应用程序级别使用更细粒度的规则覆盖它们。

One big problem with network policy—when you deploy the rules, they probably won’t do anything. Just like Ingress objects need an ingress controller to act on them, NetworkPolicy objects rely on the network implementation in your cluster to enforce them. When you deploy this policy in the next exercise, you’ll probably be disappointed to find the APOD API is still not restricted to the web app.

网络策略的一个大问题——当你部署规则时，它们可能不会做任何事情。就像 Ingress 对象需要一个入口控制器来对它们进行操作一样，NetworkPolicy 对象依赖于集群中的网络实现来强制执行它们。当您在下一个练习中部署此策略时，您可能会失望地发现 APOD API 仍不限于 Web 应用程序。

TRY IT NOW
Apply the network policy, and see if your cluster actually enforces it.
现在就试试， 应用网络策略，看看您的集群是否真正执行了它。

   ```
   # create the policy:
   kubectl apply -f apod/update/networkpolicy-api.yaml
   # confirm it is there:
   kubectl get networkpolicy
   # try to access the API from the sleep Pod—this is not permitted
   # by the policy:
   kubectl exec deploy/sleep -- curl http://apod-api/image
   ```

You can see in figure 16.3 that the sleep Pod can access the API—the NetworkPolicy that limits ingress to the web Pods is completely ignored. I’m running this on Docker Desktop, but you’ll get the same results with a default setup in K3s, AKS, or EKS.

您可以在图 16.3 中看到睡眠 Pod 可以访问 API——限制进入 Web Pod 的 NetworkPolicy 被完全忽略。我在 Docker Desktop 上运行它，但你会在 K3s、AKS 或 EKS 中使用默认设置获得相同的结果。

![图16.3](./images/Figure16.3.png)
<center>图 16.3 The network setup in your Kubernetes cluster may not enforce network policies.  Kubernetes 集群中的网络设置可能不会强制执行网络策略 </center>

The networking layer in Kubernetes is pluggable, and not every network plugin supports NetworkPolicy enforcement. The simple networks in standard cluster deployments don’t have support, so you get into this tricky situation where you can deploy all your NetworkPolicy objects, but you don’t know whether they’re being enforced unless you test them. Cloud platforms have different levels of support here. You can specify a network policy option when you create an AKS cluster; with EKS, you need to manually install a different network plugin after you create the cluster.

Kubernetes 中的网络层是可插入的，并非每个网络插件都支持 NetworkPolicy 实施。标准集群部署中的简单网络不受支持，因此您会陷入这种棘手的情况，您可以部署所有 NetworkPolicy 对象，但除非您对其进行测试，否则您不知道它们是否被强制执行。云平台在这里有不同级别的支持。您可以在创建 AKS 群集时指定网络策略选项；使用 EKS，您需要在创建集群后手动安装不同的网络插件。

This is all very frustrating for you following along with these exercises (and me writing them), but it causes a much more dangerous disconnect for organizations
using Kubernetes in production. You should look to adopt security controls early in the build cycle, so NetworkPolicy rules are applied in your development and test envi-
ronments to run the app with a close-to-production configuration. A misconfigured network policy can easily break your app, but you won’t know that if your nonproduction environments don’t enforce policy.

这对你进行这些练习（以及我编写它们）来说非常令人沮丧，但它会给组织带来更危险的脱节在生产中使用 Kubernetes。您应该在构建周期的早期采用安全控制，以便在您的开发和测试环境中应用 NetworkPolicy 规则使用接近生产的配置运行应用程序的环境。错误配置的网络策略很容易破坏您的应用程序，但如果您的非生产环境不执行策略，您将不会知道这一点。

If you want to see NetworkPolicy in action, the next exercise creates a custom cluster using Kind with Calico, an open source network plugin that enforces policy. You’ll need Docker and the Kind command line installed for this. Be warned: This exercise alters the Linux configuration for Docker and will make your original cluster unusable. Docker Desktop users can fix everything with the Reset Kubernetes button, and Kind users can replace their old cluster with a new one, but other setups might not be so lucky. It’s fine to skip these exercises and just read through my output; we’ll switch back to your normal cluster in the next section.

如果您想了解 NetworkPolicy 的实际应用，下一个练习将使用 Kind 和 Calico 创建一个自定义集群，Calico 是一个实施策略的开源网络插件。为此，您需要安装 Docker 和 Kind 命令行。请注意：此练习会更改 Docker 的 Linux 配置，并会使您的原始集群无法使用。 Docker Desktop 用户可以使用 Reset Kubernetes 按钮修复所有问题，Kind 用户可以用新集群替换旧集群，但其他设置可能就没那么幸运了。跳过这些练习并通读我的输出就可以了；我们将在下一节中切换回您的普通集群。

TRY IT NOW
Create a new cluster with Kind, and deploy a custom network plugin.
现在就试试，使用 Kind 创建一个新集群，并部署一个自定义网络插件。
   ```
   # install the Kind command line using instructions at
   # https://kind.sigs.k8s.io/docs/user/quick-start/
   # create a new cluster with a custom Kind configuration:
   kind create cluster --image kindest/node:v1.18.4 --name kiamol-ch16
   --config kind/kind-calico.yaml
   # install the Calico network plugin:
   kubectl apply -f kind/calico.yaml
   # wait for Calico to spin up:
   kubectl wait --for=condition=ContainersReady pod -l k8s-app=calico-
   node -n kube-system
   # confirm your new cluster is ready:
   kubectl get nodes
   ```

My output in figure 16.4 is abbreviated; you’ll see many more objects being created in the Calico deployment. At the end, I have a new cluster that enforces network policy. Unfortunately, the only way to know if your cluster uses a network plugin that does enforce policy is to set up your cluster with a network plugin that you know enforces policy.
我在图 16.4 中的输出是缩写的；您会看到在 Calico 部署中创建了更多对象。最后，我有一个执行网络策略的新集群。不幸的是，要知道您的集群是否使用执行策略的网络插件的唯一方法是使用您知道执行策略的网络插件设置您的集群。

Now we can try again. This cluster is completely new, with nothing running, but, of course, Kubernetes manifests are portable, so we can quickly deploy the APOD app again and try it out. (Kind supports multiple clusters running different Kubernetes versions with different configurations, so it’s a great option for test environments, but it’s not as developer-friendly as Docker Desktop or K3s).
现在我们可以再试一次。这个集群是全新的，没有任何运行，但是，当然，Kubernetes 清单是可移植的，所以我们可以快速再次部署 APOD 应用程序并进行试用。 （Kind 支持运行具有不同配置的不同 Kubernetes 版本的多个集群，因此它是测试环境的绝佳选择，但它不像 Docker Desktop 或 K3s 那样对开发人员友好）。

TRY IT NOW
Repeat the APOD and sleep deployments, and confirm that the network policy blocks unauthorized traffic.
现在就试试，重复 APOD 和睡眠部署，并确认网络策略阻止未经授权的流量。

   ```
   # deploy the APOD app to the new cluster:
   kubectl apply -f apod/
   # wait for it to spin up:
   kubectl wait --for=condition=ContainersReady pod -l app=apod-web
   # deploy the sleep Pod:
   kubectl apply -f sleep.yaml
   # confirm the sleep Pod has access to the APOD API:
   kubectl exec deploy/sleep -- curl -s http://apod-api/image
   # apply the network policy:
   kubectl apply -f apod/update/networkpolicy-api.yaml
   ```

![图16.4](./images/Figure16.4.png)
<center>图 16.4 Installing Calico gives you a cluster with network policy support—at the cost of your other cluster.  安装 Calico 为您提供了一个具有网络策略支持的集群——以您的其他集群为代价。</center>

```
      # confirm the sleep Pod can’t access the API:
   kubectl exec deploy/sleep -- curl -s http://apod-api/image
   # confirm the APOD web app still can:
   kubectl exec deploy/apod-web -- wget -O- -q http://apod-api/image
```

Figure 16.5 shows what we expected the first time around: only the APOD web app can access the API, and the sleep app times out when it tries to connect because the network plugin blocks the traffic.
图 16.5 显示了我们第一次的预期：只有 APOD 网络应用程序可以访问 API，并且睡眠应用程序在尝试连接时超时，因为网络插件阻止了流量。

Network policies are an important security control in Kubernetes, and they’re attractive to infrastructure teams who are used to firewalls and segregated networks. But you need to understand where policies fit in your developer workflow if you do choose to adopt them. If engineers run their own clusters without enforcement and you apply policy only later in the pipeline, your environments have very different configurations, and something will get broken.

网络策略是 Kubernetes 中重要的安全控制，它们对习惯于防火墙和隔离网络的基础设施团队很有吸引力。但是，如果您确实选择采用政策，则需要了解政策在哪些方面适合您的开发人员工作流程。如果工程师在没有强制执行的情况下运行他们自己的集群，而您只是在管道的后期应用策略，那么您的环境具有非常不同的配置，并且某些东西会被破坏。

![图16.5](./images/Figure16.5.png)
<center>图 16.5 Calico enforces policy so traffic to the API Pod is allowed only from the web Pod.   Calico 强制执行策略，因此只允许从 Web Pod 到 API Pod 的流量。</center>

I’ve covered only the basic details of the NetworkPolicy API here, because the complexity is more in the cluster configuration than in the policy resources. If you want to explore further, there’s a great GitHub repository full of network policy recipes published by Ahmet Alp Balkan, an engineer at Google: <https://github.com/ahmetb/kubernetes-network-policy-recipes>.
我在这里只介绍了 NetworkPolicy API 的基本细节，因为集群配置比策略资源更复杂。如果您想进一步探索，有一个很棒的 GitHub 存储库，其中包含由 Google 工程师 Ahmet Alp Balkan 发布的网络策略配方： https://github.com/ahmetb/kubernetes-network-policy-recipes 。

Now let’s clear up the new cluster and see if your old cluster still works.
现在让我们清理新集群，看看你的旧集群是否仍然有效。

TRY IT NOW
Remove the Calico cluster, and see if the old cluster is still accessible.
现在就试试，删除 Calico 集群，并查看旧集群是否仍可访问。

   ```
   # delete the new cluster:
   kind delete cluster --name kiamol-ch16
   # list your Kubernetes contexts:
   kubectl config get-contexts
   # switch back to your previous cluster:
   kubectl config set-context <your_old_cluster_name>
   # see if you can connect:
   kubectl get nodes
   ```

Your previous cluster is probably no longer accessible because of the network changes Calico made, even though Calico isn’t running now. Figure 16.6 shows me about to hit the Reset Kubernetes button in Docker Desktop; if you’re using Kind, you’ll need to delete and recreate your original cluster, and if you’re using something else and it doesn’t work . . . I did warn you.

由于 Calico 所做的网络更改，您之前的集群可能无法再访问，即使 Calico 现在没有运行。图 16.6 显示我即将点击 Docker Desktop 中的 Reset Kubernetes 按钮；如果您使用的是 Kind，则需要删除并重新创建您的原始集群，如果您使用的是其他集群并且它不起作用。 . .我确实警告过你。

![图16.6](./images/Figure16.6.png)
<center>图 16.6 Calico running in a container was able to reconfigure my network and break things. 在容器中运行的 Calico 能够重新配置我的网络并破坏一切。</center>

Now that we’re all back to normal (hopefully), we can move on to securing containers themselves, so applications don’t have privileges like reconfiguring the network stack.
现在我们都恢复正常了（希望如此），我们可以继续保护容器本身，这样应用程序就没有像重新配置网络堆栈这样的特权了。

## 16.2 使用安全上下文(security contets)限制容器功能
Container security is really about Linux security and the access model for the container user (Windows Server containers have a different user model that doesn’t have the same issues). Linux containers usually run as the root super-admin account, and unless you explicitly configure the user, root inside the container is root on the host, too. If an attacker can break out of a container running as root, they’re in charge of your server now. That’s a problem for all container runtimes, but Kubernetes adds a few more problems of its own.

容器安全性实际上是关于 Linux 安全性和容器用户的访问模型（Windows Server 容器具有不同的用户模型，但不存在相同的问题）。 Linux 容器通常以 root 超级管理员帐户运行，除非您明确配置用户，否则容器内的 root 也是主机上的 root。如果攻击者可以突破以 root 身份运行的容器，他们现在就可以控制您的服务器。这是所有容器运行时的问题，但 Kubernetes 本身又增加了一些问题。

In the next exercise, you’ll run the Pi web application with a basic Deployment configuration. That container image is packaged on top of the official .NET Core application run-time image from Microsoft. The Pod spec isn’t deliberately insecure, but you’ll see that the defaults aren’t encouraging.

在下一个练习中，您将使用基本部署配置运行 Pi Web 应用程序。该容器映像打包在 Microsoft 的官方 .NET Core 应用程序运行时映像之上。 Pod 规范并非故意不安全，但您会发现默认设置并不令人鼓舞。

TRY IT NOW
Run a simple application, and check the default security situation.
现在就试试，运行一个简单的应用程序，并检查默认的安全情况。

   ```
   # deploy the app:
   kubectl apply -f pi/
   # wait for the container to start:
   kubectl wait --for=condition=ContainersReady pod -l app=pi-web
   # print the name of the user in the Pod container:
   kubectl exec deploy/pi-web -- whoami
   # try to access the Kubernetes API server:
   kubectl exec deploy/pi-web -- sh -c 'curl -k -s
   https://kubernetes.default | grep message'
   # print the API access token:
   kubectl exec deploy/pi-web -- cat /run/secrets/kubernetes.io/
   serviceaccount/token
   ```

The behavior is scary: the app runs as root, it has access to the Kubernetes API server, and it even has a token set up so it can authenticate with Kubernetes. Figure 16.7 shows it all in action. Running as root magnifies any exploit an attacker can find in the application code or the runtime. Having access to the Kubernetes API means an attacker doesn’t even need to break out of the container—they can use the token to query the API and do interesting things, like fetch the contents of Secrets (depending on the access permissions for the Pod, which you’ll learn about in chapter 17).

行为很可怕：应用程序以 root 身份运行，它可以访问 Kubernetes API 服务器，它甚至设置了一个令牌，以便它可以通过 Kubernetes 进行身份验证。图 16.7 展示了这一切。以 root 身份运行会放大攻击者可以在应用程序代码或运行时中找到的任何漏洞。访问 Kubernetes API 意味着攻击者甚至不需要突破容器——他们可以使用令牌查询 API 并做一些有趣的事情，比如获取 Secrets 的内容（取决于 Pod 的访问权限） ，您将在第 17 章中了解）。

Kubernetes provides multiple security controls at the Pod and container levels, but they’re not enabled by default because they could break your app. You can run containers as a different user, but some apps work only if they’re running as root. You can drop Linux capabilities to restrict what the container can do, but then some app features could fail. This is where automated testing comes in, because you can increasingly tighten the security around your apps, running tests at each stage to confirm everything still works.

Kubernetes 在 Pod 和容器级别提供多种安全控制，但默认情况下未启用它们，因为它们可能会破坏您的应用程序。您可以以不同的用户身份运行容器，但某些应用程序只有在以根用户身份运行时才能运行。您可以删除 Linux 功能以限制容器可以执行的操作，但某些应用程序功能可能会失败。这就是自动化测试的用武之地，因为您可以越来越多地加强应用程序的安全性，在每个阶段运行测试以确认一切仍然有效。

![图16.7](./images/Figure16.7.png)
<center>图 16.7 If you’ve heard the phrase “secure by default,” it wasn’t said about Kubernetes. 如果您听说过“默认安全”这句话，那么它并不是关于 Kubernetes 的</center>

The main control you’ll use is the SecurityContext field, which applies security at the Pod and container levels. Listing 16.2 shows a Pod SecurityContext that explicitly sets the user and the Linux group (a collection of users), so all of the containers in the Pod run as the unknown user rather than root.
您将使用的主要控件是 SecurityContext 字段，它在 Pod 和容器级别应用安全性。清单 16.2 显示了一个明确设置用户和 Linux 组（用户集合）的 Pod SecurityContext，因此 Pod 中的所有容器都以未知用户而不是 root 身份运行。

> Listing 16.2 deployment-podsecuritycontext.yaml, running as a specific user 清单 16.2 deployment-podsecuritycontext.yaml，作为特定用户运行

```
spec: # This is the Pod spec in the Deployment.
securityContext: # These controls apply to all Pod containers.
runAsUser: 65534 # Runs as the “unknown” user
runAsGroup: 3000 # Runs with a nonexistent group
```

That’s simple enough, but moving away from root has repercussions, and the Pi spec needs a few more changes. The app listens on port 80 inside the container, and Linux requires elevated permissions to listen on that port. Root has the permission, but the new user doesn’t, so the app will fail to start. It needs some additional configuration in an environment variable to set the app to listen on port 5001 instead, which is valid for the new user. This is the sort of detail you need to drive out for each app or each class of application, and you’ll find the requirements only when the app stops working.
这很简单，但是离开 root 会产生影响，并且 Pi 规范需要做更多的改变。该应用程序在容器内侦听端口 80，而 Linux 需要提升权限才能侦听该端口。 Root有权限，新用户没有权限，所以应用会启动失败。它需要在环境变量中进行一些额外的配置，以将应用程序设置为侦听端口 5001，这对新用户有效。这是您需要为每个应用程序或每个类别的应用程序推出的那种细节，只有当应用程序停止工作时，您才会找到这些要求。


TRY IT NOW
Deploy the secured Pod spec. This uses a nonroot user and an unrestricted port, but the port mapping in the Service hides that detail from consumers.
现在就试试，部署受保护的 Pod 规范。这使用非 root 用户和不受限制的端口，但服务中的端口映射向消费者隐藏了该详细信息。

   ```
   # add the nonroot SecurityContext:
   kubectl apply -f pi/update/deployment-podsecuritycontext.yaml
   # wait for the new Pod:
   kubectl wait --for=condition=ContainersReady pod -l app=pi-web
   # confirm the user:
   kubectl exec deploy/pi-web -- whoami
   # list the API token files:
   kubectl exec deploy/pi-web -- ls -l /run/secrets/kubernetes.io/
   serviceaccount/token
   # print out the access token:
   kubectl exec deploy/pi-web -- cat /run/secrets/kubernetes.io/
   serviceaccount/token
   ```

Running as a nonroot user addresses the risk of an application exploit escalating into a full server takeover, but as shown in figure 16.8, it doesn’t solve all the problems. The Kubernetes API token is mounted with permissions for any account to read it, so an attacker could still use the API in this setup. What they can do with the API depends on how your cluster is configured—in early versions of Kubernetes, they’d be able to do everything. The identity to access the API server is different from the Linux user, and it might have administrator rights in the cluster, even though the container process is running as a least-privilege user.

以非 root 用户身份运行解决了应用程序漏洞升级为完全服务器接管的风险，但如图 16.8 所示，它并没有解决所有问题。 Kubernetes API 令牌的安装权限允许任何帐户读取它，因此攻击者仍然可以在此设置中使用该 API。他们可以用 API 做什么取决于你的集群是如何配置的——在 Kubernetes 的早期版本中，他们可以做任何事情。访问 API 服务器的身份与 Linux 用户不同，它可能在集群中具有管理员权限，即使容器进程以最低权限用户身份运行。

An option in the Pod spec stops Kubernetes from mounting the access token, which you should include for every app that doesn’t actually need to use the Kubernetes API—which will be pretty much everything, except workloads like ingress controllers, which need to find Service endpoints. It’s a safe option to set, but the next level of run-time control will need more testing and evaluation. The SecurityContext field in the container spec allows for more fine-grained control than at the Pod level. Listing 16.3 shows a set of options that work for the Pi app

Pod 规范中的一个选项阻止 Kubernetes 安装访问令牌，你应该为每个实际上不需要使用 Kubernetes API 的应用程序包含访问令牌——这几乎是所有的东西，除了像入口控制器这样的工作负载，它需要找到服务端点。这是一个安全的设置选项，但下一级运行时控制将需要更多的测试和评估。容器规范中的 SecurityContext 字段允许比 Pod 级别更细粒度的控制。清单 16.3 显示了一组适用于 Pi 应用程序的选项

![图16.8](./images/Figure16.8.png)
<center>图 16.8 You need an in-depth security approach to Kubernetes; one setting is not enough. 您需要一种深入的 Kubernetes 安全方法；一种设置是不够的。</center>

> Listing 16.3 deployment-no-serviceaccount-token.yaml, tighter security policies 清单 16.3 deployment-no-serviceaccount-token.yaml，更严格的安全策略

```
spec:
automountServiceAccountToken: false # Removes the API token
securityContext: # Applies to all containers
runAsUser: 65534
runAsGroup: 3000
containers:
- image: kiamol/ch05-pi
# …
securityContext: # Applies for this container
allowPrivilegeEscalation: false # The context settings block
capabilities: # the process from escalating to
drop: # higher privileges and drops
- all # all additional capabilities.
```

The capabilities field lets you explicitly add and remove Linux kernel capabilities. This app works happily with all capabilities dropped, but other apps will need some added back in. One feature this app doesn’t support is the readOnlyRootFilesystem option. That’s a powerful one to include if your app can work with a read-only filesystem, because it means attackers can’t write files, so they can’t download malicious scripts or binaries. How far you take this depends on the security profile of your organization. You can mandate that all apps need to run as nonroot, with all capabilities dropped and with a read-only filesystem, but that might mean you need to rewrite most of your apps.

capabilities 字段允许您显式添加和删除 Linux 内核功能。此应用程序可以在所有功能被删除的情况下正常运行，但其他应用程序将需要重新添加一些功能。此应用程序不支持的一项功能是 readOnlyRootFilesystem 选项。如果您的应用程序可以使用只读文件系统，那么这是一个强大的功能，因为这意味着攻击者无法写入文件，因此他们无法下载恶意脚本或二进制文件。你能走多远取决于你的组织的安全配置文件。您可以要求所有应用程序都需要以非 root 用户身份运行，放弃所有功能并使用只读文件系统，但这可能意味着您需要重写大部分应用程序。

A pragmatic approach is to secure your existing apps as tightly as you can at the container level and make sure you have security in depth around the rest of your policies and processes. The final spec for the Pi app is not perfectly secure, but it’s a big improvement on the defaults—and the application still works.

一种务实的方法是在容器级别尽可能严密地保护现有应用程序，并确保围绕其余策略和流程提供深入的安全保护。 Pi 应用程序的最终规范并不完全安全，但它是对默认设置的重大改进——应用程序仍然可以运行。

TRY IT NOW
Update the Pi app with the final security configuration.
现在就试试，使用最终的安全配置更新 Pi 应用程序。

   ```
   # update to the Pod spec in listing 16.3:
   kubectl apply -f pi/update/deployment-no-serviceaccount-token.yaml
   # confirm the API token doesn’t exist:
   kubectl exec deploy/pi-web -- cat /run/secrets/kubernetes.io/
   serviceaccount/token
   # confirm that the API server is still accessible:
   kubectl exec deploy/pi-web -- sh -c 'curl -k -s
   https://kubernetes.default | grep message'
   # get the URL, and check that the app still works:
   kubectl get svc pi-web -o jsonpath='http://{.status.loadBalancer
   .ingress[0].*}:8031'
   ```

As shown in figure 16.9, the app can still reach the Kubernetes API server, but it has no access token, so an attacker would need to do more work to send valid API requests. Applying a NetworkPolicy to deny ingress to the API server would remove that option altogether.

如图 16.9 所示，应用程序仍然可以访问 Kubernetes API 服务器，但它没有访问令牌，因此攻击者需要做更多的工作才能发送有效的 API 请求。应用 NetworkPolicy 拒绝进入 API 服务器将完全删除该选项。

You need to invest in adding security to your apps, but if you have a reasonably small range of application platforms, you can build up generic profiles: you might find that all your .NET apps can run as nonroot but need a writable filesystem, and all your Go apps can run with a read-only filesystem but need some Linux capabilities added. The challenge, then, is in making sure your profiles actually are applied, and Kubernetes has a nice feature for that: admission control.

您需要投资以增加应用程序的安全性，但如果您的应用程序平台范围相当小，则可以构建通用配置文件：您可能会发现所有 .NET 应用程序都可以非根用户运行但需要可写文件系统，并且您所有的 Go 应用程序都可以使用只读文件系统运行，但需要添加一些 Linux 功能。那么，挑战在于确保您的配置文件实际得到应用，而 Kubernetes 有一个很好的功能：准入控制。

## 16.3 使用 webhook 阻止和修改工作负载
Every object you create in Kubernetes goes through a process to check if it’s okay for the cluster to run that object. That process is admission control, and we saw an admission controller at work in chapter 12, trying to deploy a Pod spec that requested more resources than the namespace had available. The ResourceQuota admission controller is a built-in controller, which stops workloads from running if they exceed quotas, and Kubernetes has a plug-in system so you can add your own admission control rules.

您在 Kubernetes 中创建的每个对象都会经过一个过程来检查集群是否可以运行该对象。这个过程就是准入控制，我们在第 12 章中看到了一个准入控制器在工作，它试图部署一个 Pod 规范，该规范请求的资源多于命名空间可用的资源。 ResourceQuota 准入控制器是一个内置控制器，它会在工作负载超过配额时停止运行，并且 Kubernetes 有一个插件系统，因此您可以添加自己的准入控制规则。


Two other controllers add that extensibility: the ValidatingAdmissionWebhook, which works like ResourceQuota to allow or block object creation, and the MutatingAdmissionWebhook, which can actually edit object specs, so the object that is created is different from the request. Both controllers work in the same way: you create a configuration object specifying the object life cycles you want to control and a URL for a web server that applies the rules. Figure 16.10 shows how the pieces fit together.

其他两个控制器增加了这种可扩展性：ValidatingAdmissionWebhook，它像 ResourceQuota 一样工作以允许或阻止对象创建，以及 MutatingAdmissionWebhook，它实际上可以编辑对象规范，因此创建的对象与请求不同。两个控制器的工作方式相同：您创建一个配置对象指定要控制的对象生命周期和应用规则的 Web 服务器的 URL。图 16.10 显示了这些片段是如何组合在一起的。

![图16.9](./images/Figure16.9.png)
<center>图 16.9 The secured app is the same for users but much less fun for attackers. 受保护的应用程序对用户来说是一样的，但对攻击者来说乐趣却少得多。</center>



Admission webhooks are hugely powerful because Kubernetes calls into your own code, which can be running in any language you like and can apply whatever rules you need. In this section, we’ll apply some webhooks I’ve written in Node.js. You won’t need to edit any code, but you can see in listing 16.4 that the code isn’t particularly complicated.
Admission webhooks 非常强大，因为 Kubernetes 调用您自己的代码，这些代码可以以您喜欢的任何语言运行，并且可以应用您需要的任何规则。在本节中，我们将应用一些我用 Node.js 编写的 webhook。您不需要编辑任何代码，但您可以在清单 16.4 中看到代码并不是特别复杂。

## 16.4 validate.js, custom logic for a validating webhook validate.js，用于验证 webhook 的自定义逻辑

```
   # the incoming request has the object spec—this checks to see
   # if the service token mount property is set to false;
   # if not, the response stops the object from being created:
   if (object.spec.hasOwnProperty("automountServiceAccountToken")) {
   admissionResponse.allowed =
   (object.spec.automountServiceAccountToken == false);
   }
```
![图16.10](./images/Figure16.10.png)
<center>图 16.10 Admission webhooks let you apply your own rules when objects are created. Admission webhooks 允许您在创建对象时应用自己的规则。</center>

Webhook servers can run anywhere—inside or outside of the cluster—but they must be served on HTTPS. The only complication comes if you want to run webhooks inside your cluster signed by your own certificate authority (CA), because the webhook configuration needs a way to trust the CA. This is a common scenario, so we’ll walk through that complexity in the next exercises.
Webhook 服务器可以在任何地方运行——集群内部或外部——但它们必须在 HTTPS 上提供服务。如果您想在集群内运行由您自己的证书颁发机构 (CA) 签名的 webhook，唯一的麻烦就来了，因为 webhook 配置需要一种信任 CA 的方法。这是一个常见的场景，所以我们将在接下来的练习中逐步介绍这种复杂性。

TRY IT NOW
Start by creating a certificate and deploying the webhook server to use that certificate.
现在就试试，首先创建证书并部署 webhook 服务器以使用该证书。

   ```
   # run the Pod to generate a certificate:
   kubectl apply -f ./cert-generator.yaml
   # when the container is ready, the certificate is done:
   kubectl wait --for=condition=ContainersReady pod -l app=cert-generator
   # the Pod has deployed the cert as a TLS Secret:
   kubectl get secret -l kiamol=ch16
   # deploy the webhook server, using the TLS Secret:
   kubectl apply -f admission-webhook/
   # print the CA certificate:
   kubectl exec -it deploy/cert-generator -- cat ca.base64
   ```

The last command in that exercise will fill your screen with Base64-encoded text, which you’ll need in the next exercise (don’t worry about writing it down, though; we’ll automate all the steps). You now have the webhook server running, secured by a TLS certificate issued by a custom CA. My output appears in figure 16.11.
该练习中的最后一个命令将用 Base64 编码的文本填充您的屏幕，您将在下一个练习中需要它（不过不要担心写下来；我们将自动执行所有步骤）。您现在已经运行了 webhook 服务器，它由自定义 CA 颁发的 TLS 证书保护。我的输出如图 16.11 所示。

![图16.11](./images/Figure16.11.png)
<center>图 16.11 Webhooks are potentially dangerous, so they need to be secured with HTTPS. Webhooks 具有潜在危险，因此需要使用 HTTPS 对其进行保护。</center>

The Node.js app is running and has two endpoints: a validating webhook, which checks that all Pod specs have the automountServiceAccountToken field set to false, and a mutating webhook, which applies a container SecurityContext set with the runAsNonRoot flag. Those two policies are intended to work together to ensure a base level of security for all applications. Listing 16.5 shows the spec for the ValidatingWebhookConfiguration object.
Node.js 应用程序正在运行并具有两个端点：一个验证 webhook，它检查所有 Pod 规范是否将 automountServiceAccountToken 字段设置为 false，以及一个变异 webhook，它应用设置了 runAsNonRoot 标志的容器 SecurityContext。这两个策略旨在协同工作以确保所有应用程序的基本安全级别。清单 16.5 显示了 ValidatingWebhookConfiguration 对象的规范。

> Listing 16.5 validatingWebhookConfiguration.yaml, applying a webhook 清单 16.5 validatingWebhookConfiguration.yaml，应用 webhook

```
apiVersion: admissionregistration.k8s.io/v1beta1
kind: ValidatingWebhookConfiguration
metadata:
name: servicetokenpolicy
webhooks:
- name: servicetokenpolicy.kiamol.net
rules: # These are the object
- operations: [ "CREATE", "UPDATE" ] # types and operations
apiGroups: [""] # that invoke the
apiVersions: ["v1"] # webhook—all Pods.
resources: ["pods"]
clientConfig:
service:
name: admission-webhook # The webhook Service to call
namespace: default
path: "/validate" # URL for the webhook
caBundle: {{ .Values.caBundle }} # The CA certificate
```

Webhook configurations are flexible: you can set the types of operation and the types of object on which the webhook operates. You can have multiple webhooks configured for the same object—validating webhooks are all called in parallel, and any one of them can block the operation. This YAML file is part of a Helm chart that I’m using just for this config object, as an easy way to inject the CA certificate. A more advanced Helm chart would include a job to generate the certificate and deploy the webhook server along with the configurations—but then you wouldn’t see how it all fits together.
Webhook配置灵活：可以设置操作的类型和webhook操作的对象类型。您可以为同一个对象配置多个 webhooks——验证 webhooks 都是并行调用的，其中任何一个都可以阻止操作。这个 YAML 文件是我为这个配置对象使用的 Helm 图表的一部分，作为注入 CA 证书的简单方法。更高级的 Helm chart 将包括生​​成证书和部署 webhook 服务器以及配置的作业——但是你不会看到它们是如何组合在一起的。


TRY IT NOW
Deploy the webhook configuration, passing the CA cert from the generator Pod as a value to the local Helm chart. Then try to deploy an app, which fails the policy.
现在就试试，部署 webhook 配置，将 CA 证书作为值从生成器 Pod 传递到本地 Helm 图表。然后尝试部署一个应用程序，但该策略未通过该策略。

   ```
   # install the configuration object:
   helm install validating-webhook admission-webhook/helm/validating-
   webhook/ --set caBundle=$(kubectl exec -it deploy/cert-generator
   -- cat ca.base64)
   # confirm it’s been created:
   kubectl get validatingwebhookconfiguration
   # try to deploy an app:
   kubectl apply -f vweb/v1.yaml
   # check the webhook logs:
   kubectl logs -l app=admission-webhook --tail 3
   # show the ReplicaSet status for the app:
   kubectl get rs -l app=vweb-v1
   # show the details:
   kubectl describe rs -l app=vweb-v1
   ```

In this exercise, you can see the strength and the limitation of validating webhooks. The webhook operates at the Pod level, and it stops Pods from being created if they don’t match the service token rule. But it’s the ReplicaSet and the Deployment that try to create the Pod, and they don’t get blocked by the admission controller, so you have to dig a bit deeper to find why the app isn’t running. My output is shown in figure 16.12, where the describe command is abridged to show just the error line.
在本练习中，您可以看到验证 webhook 的优势和局限性。 webhook 在 Pod 级别运行，如果 Pod 与服务令牌规则不匹配，它会停止创建 Pod。但尝试创建 Pod 的是 ReplicaSet 和 Deployment，它们不会被准入控制器阻止，因此您必须更深入地挖掘才能找到应用程序未运行的原因。我的输出如图 16.12 所示，其中 describe 命令被删节以仅显示错误行。

You need to think carefully about the objects and operations you want your webhook to act on. This validation could happen at the Deployment level instead, which would give a better user experience, but it would miss Pods created directly or by other types of controllers. It’s also important to return a clear message in the webhook response, so users know how to fix the issue. The ReplicaSet will just keep trying to create the Pod and failing (it’s tried 18 times on my cluster while I’ve been writing this), but the failure message tells me what to do, and this one is easy to fix.
您需要仔细考虑您希望 webhook 执行的对象和操作。这种验证可以在 Deployment 级别进行，这将提供更好的用户体验，但它会错过直接创建的 Pod 或由其他类型的控制器创建的 Pod。在 webhook 响应中返回一条明确的消息也很重要，这样用户就知道如何解决问题。 ReplicaSet 将继续尝试创建 Pod 并失败（在我写这篇文章时它在我的集群上尝试了 18 次），但失败消息告诉我该怎么做，而且这个很容易修复。

![图16.12](./images/Figure16.12.png)
<center>图 16.2 Validating webhooks can block object creation, whether it was instigated by a user or a controller. 验证 webhook 可以阻止对象的创建，无论它是由用户还是控制器发起的。</center>



One of the problems with admission webhooks is that they score very low on discoverability. You can use kubectl to check if there are any validating webhooks configured, but that doesn’t tell you anything about the actual rules, so you need to have that documented outside of the cluster. The situation gets even more confusing with mutating webhooks, because if they work as expected, they give users a different object from the one they tried to create. In the next exercise, you’ll see that a wellintentioned mutating webhook can break applications.
admission webhooks 的问题之一是它们在可发现性方面的得分非常低。您可以使用 kubectl 检查是否配置了任何验证 webhook，但这并没有告诉您任何关于实际规则的信息，因此您需要在集群外部记录这些内容。变异的 webhook 使情况变得更加混乱，因为如果它们按预期工作，它们会为用户提供与他们试图创建的对象不同的对象。在下一个练习中，您将看到一个善意的变异 webhook 可以破坏应用程序。

TRY IT NOW
Configure a mutating webhook using the same webhook server but a different URL path. This webhook adds security settings to Pod specs. Deploy another app, and you’ll see the changes from the webhook stop the app from running.
现在就试试，使用相同的 webhook 服务器但不同的 URL 路径配置可变 webhook。此 webhook 将安全设置添加到 Pod 规范。部署另一个应用程序，您会看到来自 webhook 的更改使应用程序停止运行。

   ```
   # deploy the webhook configuration:
   helm install mutating-webhook admission-webhook/helm/mutating-webhook/
   --set caBundle=$(kubectl exec -it deploy/cert-generator -- cat
   ca.base64)
   # confirm it’s been created:
   kubectl get mutatingwebhookconfiguration
   # deploy a new web app:
   kubectl apply -f vweb/v2.yaml
   # print the webhook server logs:
   kubectl logs -l app=admission-webhook --tail 5
   # show the status of the ReplicaSet:
   kubectl get rs -l app=vweb-v2
   # show the details:
   kubectl describe pod -l app=vweb-v2
   ```

Oh dear. The mutating webhook adds a SecurityContext to the Pod spec with the runAsNonRoot field set to true. That flag tells Kubernetes not to run any containers in the Pod if they’re configured to run as root—which this app is, because it’s based on the official Nginx image, which does use root. As you can see in figure 16.13, describing the Pod tells you what the problem is, but it doesn’t state that the spec has been mutated. Users will be highly confused when they check their YAML again and find no runAsNonRoot field.
哦亲爱的。可变 webhook 将 SecurityContext 添加到 Pod 规范，其中 runAsNonRoot 字段设置为 true。该标志告诉 Kubernetes 不要在 Pod 中运行任何容器，如果它们被配置为以 root 运行——这个应用程序就是这样，因为它基于官方 Nginx 图像，它确实使用 root。正如你在图 16.13 中看到的，描述 Pod 告诉你问题是什么，但它并没有说明规范已经改变。当用户再次检查他们的 YAML 并发现没有 runAsNonRoot 字段时，他们会非常困惑。

The logic inside a mutating webhook is entirely up to you—you can accidentally change objects to set an invalid spec they will never deploy. It’s a good idea to have a more restrictive object selector for your webhook configurations. Listing 16.5 applies to every Pod, but you can add namespace and label selectors to narrow the scope. This webhook has been built with sensible rules, and if the Pod spec already contains a runAsNonRoot value, the webhook leaves it alone, so apps can be modeled to explicitly require the root user.
变异 webhook 中的逻辑完全取决于您——您可能会不小心更改对象以设置它们永远不会部署的无效规范。为您的 webhook 配置设置一个限制性更强的对象选择器是个好主意。清单 16.5 适用于每个 Pod，但您可以添加名称空间和标签选择器来缩小范围。这个 webhook 是用合理的规则构建的，如果 Pod 规范已经包含一个 runAsNonRoot 值，webhook 就不会管它，所以应用程序可以被建模为明确需要 root 用户。

Admission controller webhooks are a useful tool to know about, and they let you do some cool things. You can add sidecar containers to Pods with mutating webhooks, so you could use a label to identify all the apps that write log files and have a webhook automatically add a logging sidecar to those Pods. Webhooks can be dangerous, which is something you can mitigate with good testing and selective rules in your config objects, but they will always be invisible because the logic is hidden inside the webhook server.
Admission Controller webhooks 是一个有用的工具，可以让你做一些很酷的事情。您可以使用可变 webhook 将 sidecar 容器添加到 Pod，这样您就可以使用标签来识别所有写入日志文件的应用程序，并让 webhook 自动将日志记录 sidecar 添加到这些 Pod。 Webhooks 可能很危险，您可以通过配置对象中的良好测试和选择性规则来缓解这种情况，但它们将始终不可见，因为逻辑隐藏在 webhook 服务器内部。

![图16.13](./images/Figure16.13.png)
<center>图 16.13 Mutating webhooks can cause application failures, which are difficult to debug. 变异的 webhooks 会导致应用程序故障，这很难调试。</center>



In the next section, we’ll look at an alternative approach that uses validating webhooks under the hood but wraps them in a management layer. Open Policy Agent (OPA) lets you define your rules in Kubernetes objects, which are discoverable in the cluster and don’t require custom code.
在下一节中，我们将研究另一种方法，该方法在底层使用验证 webhook，但将它们包装在管理层中。 Open Policy Agent (OPA) 允许您在 Kubernetes 对象中定义规则，这些对象在集群中是可发现的并且不需要自定义代码。

## 16.4 使用 Open Policy Agent 控制准入
OPA is a unified approach to writing and implementing policies. The goal is to provide a standard language for describing all kinds of policy and integrations to apply policies in different platforms. You can describe data access policies and deploy them in SQL databases, and you can describe admission control policies for Kubernetes objects. OPA is another CNCF project that provides a much cleaner alternative to custom validating webhooks with OPA Gatekeeper.
OPA 是编写和实施策略的统一方法。目标是提供一种标准语言来描述各种策略和集成，以在不同平台上应用策略。你可以描述数据访问策略并将它们部署在 SQL 数据库中，你可以描述 Kubernetes 对象的准入控制策略。 OPA 是另一个 CNCF 项目，它为使用 OPA Gatekeeper 的自定义验证 webhooks 提供了更简洁的替代方案。

OPA Gatekeeper features three parts: you deploy the Gatekeeper components in your cluster, which include a webhook server and a generic ValidatingWebhookConfiguration; then you create a constraint template, which describes the admission control policy; and then you create a specific constraint based on the template. It’s a flexible approach where you can build a template for the policy “all Pods must have the expected labels” and then deploy a constraint to say which labels are needed in which namespace.
OPA Gatekeeper 具有三个部分：您在集群中部署 Gatekeeper 组件，其中包括一个 webhook 服务器和一个通用的 ValidatingWebhookConfiguration；然后创建一个约束模板，它描述了准入控制策略；然后根据模板创建特定约束。这是一种灵活的方法，您可以为“所有 Pod 必须具有预期的标签”策略构建一个模板，然后部署一个约束来说明在哪个命名空间中需要哪些标签。

We’ll start by removing the custom webhooks we added and deploying OPA Gatekeeper, ready to apply some admission policies.
我们将首先删除我们添加的自定义 webhook 并部署 OPA Gatekeeper，准备应用一些准入策略。

TRY IT NOW
Uninstall the webhook components, and deploy Gatekeeper.
现在就试试，卸载 webhook 组件，并部署 Gatekeeper。


   ```
   # remove the webhook configurations created with Helm:
   helm uninstall mutating-webhook
   helm uninstall validating-webhook
   # remove the Node.js webhook server:
   kubectl delete -f admission-webhook/
   # deploy Gatekeeper:
   kubectl apply -f opa/
   ```

I’ve abbreviated my output in figure 16.14—when you run the exercise, you’ll see the OPA Gatekeeper deployment installs many more objects, including things we haven’t come across yet called CustomResourceDefinitions (CRDs). We’ll cover those in more detail in chapter 20 when we look at extending Kubernetes, but for now, it’s enough to know that CRDs let you define new types of object that Kubernetes stores and manages for you.
我在图 16.14 中简化了我的输出——当您运行该练习时，您会看到 OPA Gatekeeper 部署安装了更多的对象，包括我们尚未遇到的称为 CustomResourceDefinitions (CRD) 的对象。当我们着眼于扩展 Kubernetes 时，我们将在第 20 章中更详细地介绍这些内容，但就目前而言，知道 CRD 可以让您定义 Kubernetes 为您存储和管理的新对象类型就足够了。

Gatekeeper uses CRDs so you can create templates and constraints as ordinary Kubernetes objects, defined in YAML and deployed with kubectl. The template contains the generic policy definition in a language called Rego (pronounced “ray-go”). It’s an expressive language that lets you evaluate the properties of some input object to check if they meet your requirements. It’s another thing to learn, but Rego has some big advantages: policies are fairly easy to read, and they live in your YAML files, so they’re not hidden in the code of a custom webhook; and there are lots of sample Rego policies to enforce the kind of rules we’ve looked at in this chapter. Listing 16.6 shows a Rego policy that requires objects to have labels.
Gatekeeper 使用 CRD，因此您可以创建模板和约束作为普通 Kubernetes 对象，在 YAML 中定义并使用 kubectl 部署。该模板包含使用称为 Rego（发音为“ray-go”）的语言的通用策略定义。它是一种表达性语言，可让您评估某些输入对象的属性以检查它们是否满足您的要求。学习是另一回事，但 Rego 有一些很大的优势：策略相当容易阅读，并且它们存在于您的 YAML 文件中，因此它们不会隐藏在自定义 webhook 的代码中；并且有很多示例 Rego 策略来执行我们在本章中看到的那种规则。清单 16.6 显示了要求对象具有标签的 Rego 策略。

![图16.14](./images/Figure16.14.png)
<center>图 16.14 OPA Gatekeeper takes care of all the tricky parts of running a webhook server. OPA Gatekeeper 负责处理运行 webhook 服务器的所有棘手部分。</center>

> Listing 16.6 requiredLabels-template.yaml, a basic Rego policy 清单 16.6 requiredLabels-template.yaml，一个基本的 Rego 策略

```
# This fetches all the labels on the object and all the
# required labels from the constraint; if required labels
# are missing, that’s a violation that blocks object creation.
violation[{"msg": msg, "details": {"missing_labels": missing}}] {
provided := {label | input.review.object.metadata.labels[label]}
required := {label | label := input.parameters.labels[_]}
missing := required - provided
count(missing) > 0
msg := sprintf("you must provide labels: %v", [missing])
}
```

You deploy that policy with Gatekeeper as a constraint template, and then you deploy a constraint object that enforces the template. In this case, the template, called RequiredLabels, uses parameters to define the labels that are required. Listing 16.7 shows a specific constraint for all Pods to have app and version labels.
您使用 Gatekeeper 部署该策略作为约束模板，然后部署强制执行该模板的约束对象。在这种情况下，名为 RequiredLabels 的模板使用参数来定义所需的标签。清单 16.7 显示了所有 Pod 具有应用和版本标签的特定约束。

> Listing 16.7 requiredLabels.yaml, a constraint from a Gatekeeper template 清单 16.7 requiredLabels.yaml，来自 Gatekeeper 模板的约束

```
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: RequiredLabels # The API and Kind identify this as
metadata: # a Gatekeeper constraint from the
name: requiredlabels-app # RequiredLabels template.
spec:
match:
kinds:
- apiGroups: [""]
kinds: ["Pod"] # The constraint applies to all Pods.
parameters:
labels: ["app", "version"] # Requires two labels to be set
```

This is much easier to read, and you can deploy many constraints from the same template. The OPA approach lets you build a standard policy library, which users can apply in their application specs without needing to dig into the Rego. In the next exercise, you’ll deploy the constraint from listing 16.7 with another constraint that requires all Deployments, Services, and ConfigMaps to have a kiamol label. Then you’ll try to deploy a version of the to-do app that fails all those policies.
这更容易阅读，并且您可以从同一个模板部署许多约束。 OPA 方法允许您构建一个标准策略库，用户可以在他们的应用程序规范中应用该库，而无需深入研究 Rego。在下一个练习中，您将部署清单 16.7 中的约束和另一个要求所有 Deployments、Services 和 ConfigMaps 都具有 kiamol 标签的约束。然后，您将尝试部署一个不符合所有这些政策的待办事项应用程序版本。

TRY IT NOW
Deploy required label policies with Gatekeeper, and see how they are applied.
现在就试试，使用 Gatekeeper 部署所需的标签策略，并查看它们的应用方式。


   ```
   # create the constraint template first:
   kubectl apply -f opa/templates/requiredLabels-template.yaml
   # then create the constraint:
   kubectl apply -f opa/constraints/requiredLabels.yaml
   # the to-do list spec doesn’t meet the policies:
   kubectl apply -f todo-list/
   # confirm the app isn’t deployed:
   kubectl get all -l app=todo-web
   ```

You can see in figure 16.15 that this user experience is clean—the objects we’re trying to create don’t have the required labels, so they get blocked, and we see the message from the Rego policy in the output from kubectl.
您可以在图 16.15 中看到这种用户体验很干净——我们尝试创建的对象没有所需的标签，因此它们被阻止了，我们在 kubectl 的输出中看到了来自 Rego 策略的消息。

Gatekeeper evaluates constraints using a validating webhook, and it’s very obvious when failures arise in the object you’re creating. It’s a bit less clear when objects created by controllers fail validation, because the controller itself can be fine. We saw that in section 16.3, and because Gatekeeper uses the same validation mechanism, it has the same issue. You’ll see that if you update the to-do app so the Deployment meets the label requirements but the Pod spec doesn’t.
Gatekeeper 使用验证 webhook 评估约束，当您创建的对象出现故障时，这一点非常明显。当控制器创建的对象验证失败时不太清楚，因为控制器本身可能没问题。我们在 16.3 节中看到，由于 Gatekeeper 使用相同的验证机制，因此存在相同的问题。您会看到，如果您更新待办事项应用程序，那么 Deployment 会满足标签要求，但 Pod 规范不会。

TRY IT NOW
Deploy an updated to-do list spec, which has the correct labels for all objects except the Pod.
现在就试试，部署更新的待办事项列表规范，其中包含除 Pod 之外的所有对象的正确标签。

   ```
   # deploy the updated manifest:
   kubectl apply -f todo-list/update/web-with-kiamol-labels.yaml
   # show the status of the ReplicaSet:
   kubectl get rs -l app=todo-web
   ```

![图16.15](./images/Figure16.15.png)
<center>图 16.15 Deployment failures show a clear error message returned from the Rego policy. 部署失败显示从 Rego 策略返回的明确错误消息。</center>

```
   # print the detail:
   kubectl describe rs -l app=todo-web
   # remove the to-do app in preparation for the next exercise:
   kubectl delete -f todo-list/update/web-with-kiamol-labels.yaml
```

You’ll find in this exercise that the admission policy worked, but you see the problem only when you dig into the description for the failing ReplicaSet, as in figure 16.16. That’s not such a great user experience. You could fix this with a more sophisticated policy that applies at the Deployment level and checks labels in the Pod template—that could be done with extended logic in the Rego for the constraint template.
在本练习中，您会发现准入策略有效，但只有当您深入了解失败的 ReplicaSet 的描述时，您才会发现问题，如图 16.16 所示。那不是很好的用户体验。您可以使用更复杂的策略来解决此问题，该策略适用于 Deployment 级别并检查 Pod 模板中的标签——这可以通过约束模板的 Rego 中的扩展逻辑来完成。

We’ll finish this section with the following set of admission policies that cover some more production best practices, all of which help to make your apps more secure:
我们将以下面一组涵盖更多生产最佳实践的准入政策来结束本节，所有这些都有助于提高您的应用程序的安全性：
+ All Pods must have container probes defined. This is for keeping your apps healthy, but a failed healthcheck could also indicate unexpected activity from an attack. 所有 Pod 都必须定义容器探测器。这是为了保持您的应用程序健康，但失败的健康检查也可能表明来自攻击的意外活动。

![图16.16](./images/Figure16.16.png)
<center>图 16.16 OPA Gatekeeper makes for a better process, but it’s still a wrapper around validating webhooks. OPA Gatekeeper 实现了更好的流程，但它仍然是验证 webhook 的包装器。 </center>

+ Pods can run containers only from approved image repositories. Restricting containers to a set of “golden” repositories with secured production images ensures malicious payloads can’t be deployed. Pod 只能从已批准的镜像存储库运行容器。将容器限制在一组具有安全生产映像的“黄金”存储库中，可确保无法部署恶意负载。
+ All containers must have memory and CPU limits set. This prevents a compromised container maxing out the compute resources of the node and starving all the other Pods. 所有容器都必须设置内存和 CPU 限制。这可以防止受损容器最大化节点的计算资源并使所有其他 Pod 挨饿。

These generic policies apply to pretty much every organization. You can add to them with constraints that require network policies for every app and security contexts for every Pod. As you’ve learned in this chapter, not all rules are universal, so you might need to be selective on how you apply those constraints. In the next exercise, you’ll apply the production constraint set to a single namespace.
这些通用政策几乎适用于每个组织。您可以向它们添加约束，要求每个应用程序的网络策略和每个 Pod 的安全上下文。正如您在本章中了解到的，并非所有规则都是通用的，因此您可能需要选择如何应用这些约束。在下一个练习中，您将把生产约束集应用到单个命名空间。

TRY IT NOW
Deploy a new set of constraints and a version of the to-do app where the Pod spec fails most of the policies.
现在就试试，部署一组新的约束和一个待办应用程序版本，其中 Pod 规范不符合大多数策略。

   ```
   # create templates for the production constraints:
   kubectl apply -f opa/templates/production/
   # create the constraints:
   kubectl apply -f opa/constraints/production/
   # deploy the new to-do spec:
   kubectl apply -f todo-list/production/
   # confirm the Pods aren’t created:
   kubectl get rs -n kiamol-ch16 -l app=todo-web
   # show the error details:
   kubectl describe rs -n kiamol-ch16 -l app=todo-web
   ```

Figure 16.17 shows that the Pod spec fails all the rules except one—my image repository policy allows any images from Docker Hub in the kiamol organization, so the todo app image is valid. But there’s no version label, no health probes, and no resource limits, and this spec is not fit for production.
图 16.17 显示 Pod 规范不符合所有规则，除了一个——我的图像存储库策略允许来自 kiamol 组织中的 Docker Hub 的任何图像，因此待办事项应用程序图像是有效的。但是没有版本标签，没有健康探测，也没有资源限制，这个规范不适合生产。

![图16.17](./images/Figure16.17.png)
<center>图 16.17 All constraints are evaluated, and you see the full list of errors in the Rego output. 计算所有约束，您可以在 Rego 输出中看到完整的错误列表。 </center>

Just to prove those policies are achievable and OPA Gatekeeper will actually let the todo app run, you can apply an updated spec that meets all the rules for production. If you compare the YAML files in the production folder and the update folder, you’ll see the new spec just adds the required fields to the Pod template; there are no significant changes in the app.
只是为了证明这些政策是可以实现的，并且 OPA Gatekeeper 实际上会让待办事项应用程序运行，您可以应用满足所有生产规则的更新规范。如果比较生产文件夹和更新文件夹中的 YAML 文件，您会看到新规范只是将必填字段添加到 Pod 模板；应用程序没有重大变化。

TRY IT NOW
Apply a production-ready version of the to-do spec, and confirm the app really runs.
现在就试试，应用待办事项规范的生产就绪版本，并确认应用程序真正运行。

   ```
   # this spec meets all production policies:
   kubectl apply -f todo-list/production/update
   # wait for the Pod to start:
   kubectl wait --for=condition=ContainersReady pod -l app=todo-web -n
   kiamol-ch16
   # confirm it’s running:
   kubectl get pods -n kiamol-ch16 --show-labels
   # get the URL for the app and browse:
   kubectl get svc todo-web -n kiamol-ch16 -o jsonpath='http://{.status
   .loadBalancer.ingress[0].*}:8019'
   ```

Figure 16.18 shows the app running, after the updated deployment has been permitted by OPA Gatekeeper.图 16.18 显示了在 OPA Gatekeeper 允许更新部署后应用程序正在运行。

Open Policy Agent is a much cleaner way to apply admission controls than custom validating webhooks, and the sample policies we’ve looked at are only some simple ideas to get you started. Gatekeeper doesn’t have mutation functionality, but you can combine it with your own webhooks if you have a clear case to modify specs. You could use constraints to ensure every Pod spec includes an application-profile label and then mutate specs based on your profiles—setting your .NET Core apps to run as a nonroot user and switching to a read-only filesystem for all your Go apps.
Open Policy Agent 是一种比自定义验证 webhook 更简洁的应用准入控制的方法，我们查看的示例策略只是一些简单的想法，可以帮助您入门。 Gatekeeper 没有突变功能，但如果您有明确的修改规范的案例，您可以将其与您自己的 webhooks 结合使用。您可以使用约束来确保每个 Pod 规范都包含一个应用程序配置文件标签，然后根据您的配置文件改变规范——将您的 .NET Core 应用程序设置为以非根用户身份运行，并为所有 Go 应用程序切换到只读文件系统。

Securing your apps is about closing down exploit paths, and a thorough approach includes all the tools we’ve covered in this chapter and more. We’ll finish up with a look at a secure Kubernetes landscape.
保护你的应用程序就是关闭漏洞利用路径，一个彻底的方法包括我们在本章中介绍的所有工具等等。最后，我们将了解一个安全的 Kubernetes 环境。

## 16.5 深入了解 Kubernetes 中的安全性
Build pipelines can be compromised, container images can be modified, containers can run vulnerable software as privileged users, and attackers with access to the Kubernetes API could even take control of your cluster. You won’t know your app is 100% secure until it has been replaced and you can confirm no security breaches occurred during its operation. Getting to that happy place means applying security in depth across your whole software supply chain. This chapter has focused on securing apps at run time, but you should start before that by scanning container images for known vulnerabilities.
构建管道可能会遭到破坏，容器映像可能会被修改，容器可能会以特权用户身份运行易受攻击的软件，而有权访问 Kubernetes API 的攻击者甚至可能会控制您的集群。在您的应用程序被替换之前，您不会知道它是 100% 安全的，并且您可以确认在其运行期间没有发生安全漏洞。到达那个快乐的地方意味着在整个软件供应链中深入应用安全性。本章的重点是在运行时保护应用程序，但您应该在此之前扫描容器镜像以查找已知漏洞。

![图16.18](./images/Figure16.18.png)
<center>图 16.18 Constraints are powerful, but you need to make sure apps can actually comply.约束很强大，但您需要确保应用程序能够真正遵守。  </center>

Security scanners look inside an image, identify the binaries, and check them on CVE (Common Vulnerabilities and Exposures) databases. Scans tell you if known exploits
are in the application stack, dependencies, or operating system tools in your image. Commercial scanners have integrations with managed registries (you can use Aqua Security with Azure Container Registry), or you can run your own (Harbor is the CNCF registry project, and it supports the open source scanners Clair and Trivy; Docker Desktop has an integration with Snyk for local scans).
安全扫描器查看图像内部，识别二进制文件，并在 CVE（常见漏洞和暴露）数据库中检查它们。扫描会告诉您是否存在已知漏洞位于映像中的应用程序堆栈、依赖项或操作系统工具中。商业扫描器与托管注册表集成（您可以将 Aqua Security 与 Azure 容器注册表结合使用），或者您可以运行自己的（Harbor 是 CNCF 注册表项目，它支持开源扫描器 Clair 和 Trivy；Docker Desktop 与Snyk 用于本地扫描）。

You can set up a pipeline where images are pushed to a production repository only if the scan is clear. Combine that with a repository admission policy, and you can effectively ensure that containers run only if the image is safe. A secure image running in a securely configured container is still a target, though, and you should look at run-time security with a tool that monitors containers for unusual activity and can generate alerts or shut down suspicious behavior. Falco is the CNCF project for run-time security, and there are supported commercial options from Aqua and Sysdig (among others).
您可以设置一个管道，只有在扫描清晰时图像才会被推送到生产存储库。将其与存储库准入策略相结合，您可以有效地确保容器仅在镜像安全时运行。不过，在安全配置的容器中运行的安全映像仍然是一个目标，您应该使用一种工具来查看运行时安全性，该工具可以监视容器的异常活动，并可以生成警报或关闭可疑行为。 Falco 是用于运行时安全的 CNCF 项目，Aqua 和 Sysdig（以及其他）提供支持的商业选项。

Overwhelmed? You should think about securing Kubernetes as a road map that starts with the techniques I’ve covered in this chapter. You can adopt security contexts first, then network policies, and then move on to admission control when you’re clear about the rules that matter to you. Role-based access control, which we cover in chapter 17, is the next stage. Security scanning and run-time monitoring are further steps you can take if your organization has enhanced security requirements. But I won’t throw anything more at you now—let’s tidy up and get ready for the lab.
您可以设置一个管道，只有在扫描清晰时图像才会被推送到生产存储库。将其与存储库准入策略相结合，您可以有效地确保容器仅在镜像安全时运行。不过，在安全配置的容器中运行的安全映像仍然是一个目标，您应该使用一种工具来查看运行时安全性，该工具可以监视容器的异常活动，并可以生成警报或关闭可疑行为。 Falco 是用于运行时安全的 CNCF 项目，Aqua 和 Sysdig（以及其他）提供支持的商业选项。

TRY IT NOW
Delete all the objects we created.
现在就试试，删除我们创建的所有对象：

   ```
   kubectl delete -f opa/constraints/ -f opa/templates/ -f
   opa/gatekeeper.yaml
   kubectl delete all,ns,secret,networkpolicy -l kiamol=ch16
   ```

## 16.6 实验室
At the start of the chapter, I said that volume mounts for host paths are a potential attack vector, but we didn’t address that in the exercises, so we’ll do it in the lab. This is a perfect scenario for admission control, where Pods should be blocked if they use volumes that mount sensitive paths on the host. We’ll use OPA Gatekeeper, and I’ve written the Rego for you, so you just need to write a constraint.
在本章开头，我说过主机路径的卷挂载是一个潜在的攻击向量，但我们没有在练习中解决这个问题，所以我们将在实验室中进行。这是准入控制的完美场景，如果 Pod 使用在主机上挂载敏感路径的卷，则应该阻止它们。我们将使用 OPA Gatekeeper，我已经为您编写了 Rego，因此您只需编写一个约束即可。

+ Start by deploying gatekeeper.yaml in the lab folder. 首先在实验室文件夹中部署 gatekeeper.yaml。
+ Then deploy the constraint template in `restrictedPaths-template.yaml` you’ll need to look at the spec to see how to build your constraint. 然后在 restrictedPaths-template.yaml 中部署约束模板，您需要查看规范以了解如何构建约束。
+ Write and deploy a constraint that uses the template and restricts these host paths: /, /bin, and /etc. The constraint should apply only to Pods with the label kiamol=ch16-lab. 编写并部署使用模板并限制这些主机路径的约束：/、/bin 和 /etc。该约束应仅适用于标签为 kiamol=ch16-lab 的 Pod。
+ Deploy sleep.yaml in the lab folder. Your constraint should prevent the Pod from being created because it uses restricted volume mounts. 在实验室文件夹中部署 sleep.yaml。您的约束应该阻止创建 Pod，因为它使用受限的卷安装。

This one is fairly straightforward, although you’ll need to read about match expressions, which is how Gatekeeper implements label selectors. My solution is up on GitHub: <https://github.com/sixeyed/kiamol/blob/master/ch16/lab/README.md>.
这一个相当简单，尽管您需要阅读匹配表达式，这就是 Gatekeeper 实现标签选择器的方式。我的解决方案在 GitHub 上： https://github.com/sixeyed/kiamol/blob/master/ch16/lab/README.md 。