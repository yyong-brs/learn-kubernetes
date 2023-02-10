# 第九章 通过 rollouts 和 rollbacks 管理应用发布

你会更频繁地更新现有应用，而不是部署新应用。容器化应用程序从它们使用的基本镜像继承多个发布节奏;Docker Hub上针对操作系统、平台sdk和运行时的官方镜像通常每个月都会发布一个新版本。您应该有一个重建镜像的过程，并在这些依赖项更新时发布更新，因为它们可能包含关键的安全补丁。这个过程的关键是能够安全地推出更新，并在出现错误时为自己提供暂停和回滚更新的选项。Kubernetes 为 Deployments、DaemonSets 和 StatefulSets 提供了这些场景。

单一的更新方法并不适用于所有类型的应用程序，因此 Kubernetes 为控制器提供了不同的更新策略，并提供了调整策略工作方式的选项。我们将在本章探讨所有这些选项。如果你正在考虑跳过这一点，因为你对 6000 字的应用程序更新不感兴趣，我建议你坚持下去。更新是导致应用程序宕机的最大原因，但如果您了解 Kubernetes 提供的工具，就可以显著降低风险。我会试着在这个过程中注入一些有趣的内容。

## 9.1 Kubernetes 如何管理 rollouts

我们将从 Deployments 开始——实际上，您已经完成了大量的 Deployment 更新。每次我们对现有的 Deployment 进行更改(我们一章要做10次)，Kubernetes 都会进行一次 rollout。在展开过程中，Deployment 将创建一个新的 ReplicaSet，并将其扩展到所需的副本数量，同时将先前的 ReplicaSet 缩小到零副本。图 9.1 显示了正在进行的更新。

![图9.1](.\images\Figure9.1.png)
<center>图 9.1 Deployment 控制多个 ReplicaSet，以便它们可以管理滚动更新</center>

Rollouts aren’t triggered from every change to a Deployment, only from a change to the Pod spec. If you make a change that the Deployment can manage with the current ReplicaSet, like updating the number of replicas, that’s done without a rollout.

不是 Deployment 的每次更改都会触发 rollout 的，只有 Pod spec 的更改才会触发 rollout。如果您做了一个 Deployemt 可以使用当前 ReplicaSet 管理的更改，比如更新副本的数量，则无需 rollout 即可完成。

<b>现在就试试</b> 部署一个简单的两个副本应用程序，然后更新它以增加规模，并查看如何管理 ReplicaSet

```
# 切换到章节练习目录:
cd ch09

# 部署一个简单的 web app:
kubectl apply -f vweb/

# 检查 ReplicaSets:
kubectl get rs -l app=vweb

# 现在扩展规模:
kubectl apply -f vweb/update/vweb-v1-scale.yaml

# 检查 ReplicaSets:
kubectl get rs -l app=vweb

# 检查 deployment 历史:
kubectl rollout history deploy/vweb
```

kubectl rollout 命令提供了查看和管理 rollout 的选项。您可以从图9.2的输出中看到，本练习中只有一次rollout，即创建ReplicaSet的初始部署。规模更新只更改了现有的ReplicaSet，因此没有第二次 rollout。

![图9.2](.\images\Figure9.2.png)
<center>图 9.2 Deployment 通过 rollout 管理变更，但仅限 Pod spec 更改时</center>

您正在进行的应用程序更新将集中于部署运行容器镜像的更新版本的新 Pods。您应该通过更新 YAML spec 来管理这个问题，但是 kubectl 使用 set 命令提供了一个快速的替代方案。使用此命令是更新现有部署的必要方法，您应该将其视为与scale 命令相同的方法—它是摆脱棘手情况的有用方法，但需要随后更新YAML文件。

<b>现在就试试</b> 使用 kubectl 更新部署的镜像版本。这是对Pod spec 的更改，因此它将触发一个新的 rollout

```
# 更新web应用程序的镜像:
kubectl set image deployment/vweb web=kiamol/ch09-vweb:v2

# 再次检查 ReplicaSets:
kubectl get rs -l app=vweb

# 检查 rollouts:
kubectl rollout history deploy/vweb
```

kubectl set 命令改变一个现有对象的 spec 。您可以使用它来更改 Pod 的镜像或环境变量或 Service 的选择器。这是应用新的 YAML spec 的捷径，但它的实现方式是相同的。在这个练习中，更改导致了一个 rollout，创建了一个新的ReplicaSet 来运行新的 Pod spec，而旧的ReplicaSet缩小到零。在图9.3中可以看到这一点。

![图9.3](.\images\Figure9.3.png)
<center>图 9.3 必要的更新经过相同的 rollout 过程，但现在您的 YAML 和实际状态是不同步的</center>

Kubernetes 对其他 Pod 控制器(DaemonSets和StatefulSets)使用了相同的 rollout 概念。它们是API中一个奇怪的部分，因为它们不直接映射到对象(您不需要创建具有“rollout”类型的资源)，但是它们是用于您的版本的重要管理工具。您可以使用 rollout 来跟踪版本历史并恢复到以前的版本。

## 9.2 使用 rollouts 和 rollbacks 更新 Deployments

如果您再次查看图 9.3，您将看到 rollout 历史记录非常没有帮助。每次 rollout 都会记录一个修订号，但没有其他记录。目前还不清楚是什么原因导致了这个变化，也不清楚哪个 ReplicaSet 与哪个修订相关。最好包含一个版本号(或Git提交ID)作为 Pods 的标签，然后 Deployment 也将该标签添加到 ReplicaSet 中，这样可以更容易地跟踪更新。

<b>现在就试试</b> 对 Deployment 应用更新，它使用相同的 Docker 镜像，但更改 Pod 的版本标签。这是对 Pod spec 的更改，因此它将创建一个新的 rollout。**

```
# apply the change using the record flag:
kubectl apply -f vweb/update/vweb-v11.yaml --record

# check the ReplicaSets and their labels:
kubectl get rs -l app=vweb --show-labels

# check the current rollout status:
kubectl rollout status deploy/vweb

# check the rollout history:
kubectl rollout history deploy/vweb

# show the rollout revision for the ReplicaSets:
kubectl get rs -l app=vweb -o=custom-columns=NAME:.metadata.name,
  REPLICAS:.status.replicas,REVISION:.metadata.annotations.deployment
  \.kubernetes\.io/revision
```

My output appears in figure 9.4. Adding the record flag saves the kubectl command as a detail to the rollout, which can be helpful if your YAML files have identifying names. Often they won’t because you’ll be deploying a whole folder, so the version number label in the Pod spec is a useful addition. Then, however, you need some awkward JSONPath to find the link between a rollout revision and a ReplicaSet.
我的输出如图9.4所示。添加记录标志将kubectl命令作为详细信息保存到滚出中，如果您的YAML文件具有标识名称，这将很有帮助。通常情况下，他们不会这样做，因为您将部署整个文件夹，所以Pod规范中的版本号标签是一个有用的补充。然后，您需要一些笨拙的JSONPath来查找推出修订和ReplicaSet之间的链接。

​	As your Kubernetes maturity increases, you’ll want to have a standard set of labels that you include in all your object specs. Labels and selectors are a core feature, and you’ll use them all the time to find and manage objects. Application name, component name, and version are good labels to start with, but it’s important to distinguish between the labels you include for your convenience and the labels that Kubernetes uses to map object relationships.
随着Kubernetes成熟度的提高，您将希望拥有一组包含在所有对象规范中的标准标签。标签和选择器是一个核心特性，您将一直使用它们来查找和管理对象。应用程序名称、组件名称和版本都是很好的标签，但是区分为方便而包含的标签和Kubernetes用于映射对象关系的标签是很重要的。


​	Listing 9.1 shows the Pod labels and the selector for the Deployment in the previous exercise. The app label is used in the selector, which the Deployment uses to find  its Pods. The Pod also contains a version label for our convenience, but that’s not part of the selector. If it were, then the Deployment would be linked to one version, because you can’t change the selector once a Deployment is created.
清单9.1显示了Pod标签和前面练习中Deployment的选择器。app标签在选择器中使用，Deployment使用该选择器来查找它的Pods。为了方便起见，Pod还包含一个版本标签，但那不是选择器的一部分。如果是，则Deployment将链接到一个版本，因为一旦创建了Deployment就不能更改选择器。

![图9.4](.\images\Figure9.4.png)
<center>图 9.4 Kubernetes使用标签作为关键信息，额外的细节存储在注释中 Kubernetes uses labels for key information, and extra detail is stored in annotations.</center>

**Listing 9.1	vweb-v11.yaml, a Deployment with additional labels in the Pod spec**
**清单9.1 vweb-v11。yaml，在Pod规范中有附加标签的部署**

```
spec:
  replicas: 3
  selector:
    matchLabels:
      app: vweb # The app name is used as the selector.
  template:
    metadata:
      labels:
        app: vweb
        version: v1.1 # The Pod spec also includes a version label.
```

You need to plan your selectors carefully up front, but you should add whatever labels you need to your Pod spec to make your updates manageable. Deployments retain multiple ReplicaSets (10 is the default), and the Pod template hash in the name makes them hard to work with directly, even after just a few updates. Let’s see what the app we’ve deployed actually does and then look at the ReplicaSets in another rollout.
你需要事先仔细规划你的选择器，但你应该在Pod规范中添加任何你需要的标签，以使你的更新易于管理。部署会保留多个replicaset(10是默认值)，并且名称中的Pod模板散列使得它们很难直接使用，即使在进行了几次更新之后也是如此。让我们看看我们部署的应用程序实际做了什么，然后在另一个rollout中查看ReplicaSets。

​	**TRY IT NOW	Make an HTTP call to the web Service to see the response, then start another update and check the response again.**
**立即尝试**向web服务发起HTTP调用以查看响应，然后启动另一次更新并再次检查响应

```
# we’ll use the app URL a lot, so save it to a local file:
kubectl get svc vweb -o
  jsonpath='http://{.status.loadBalancer.ingress[0].*}:8090/v.txt' >
  url.txt
  
# then use the contents of the file to make an HTTP request:
curl $(cat url.txt)

# deploy the v2 update:
kubectl apply -f vweb/update/vweb-v2.yaml --record

# check the response again:
curl $(cat url.txt)

# check the ReplicaSet details:
kubectl get rs -l app=vweb --show-labels
```

You’ll see in this exercise that ReplicaSets aren’t easy objects to manage, which is where standardized labels come in. It’s easy to see which version of the app is active by checking the labels for the ReplicaSet which has all the desired replicas—as you see in figure 9.5—but labels are just text fields, so you need process safeguards to make sure they’re reliable.
在本练习中，您将看到replicaset并不是易于管理的对象，因此需要使用标准化标签。通过检查ReplicaSet的标签，可以很容易地看到应用程序的哪个版本处于活动状态(如图9.5所示)，但标签只是文本字段，因此需要流程保障以确保它们是可靠的。

​	Rollouts do help to abstract away the details of the ReplicaSets, but their main use is to manage releases. We’ve seen the rollout history from kubectl, and you can also run commands to pause an ongoing rollout or roll back a deployment to an earlier revision. A simple command will roll back to the previous deployment, but if you want to roll back to a specific version, you need some more JSONPath trickery to find the revision you want. We’ll see that now and use a very handy feature of kubectl that tells you what will happen when you run a command, without actually executing it.
rollout确实有助于抽象ReplicaSets的细节，但它们的主要用途是管理发布。我们已经看到了kubectl的推出历史，您还可以运行命令暂停正在进行的推出或将部署回滚到较早的版本。一个简单的命令将回滚到以前的部署，但是如果您想回滚到特定的版本，则需要更多的JSONPath技巧来查找所需的版本。现在我们将看到这一点，并使用kubectl的一个非常方便的特性，该特性可以告诉您运行命令时会发生什么，而无需实际执行该命令。

​	**TRY IT NOW	Check the rollout history and try rolling back to v1 of the app.**
**现在就试一下，检查应用的推出历史记录，并尝试回滚到应用的v1

```
# look at the revisions:
kubectl rollout history deploy/vweb

# list ReplicaSets with their revisions:
kubectl get rs -l app=vweb -o=custom-
  columns=NAME:.metadata.name,REPLICAS:.status.replicas,VERSION:.meta
  data.labels.version,REVISION:.metadata.annotations.deployment\.kube
  rnetes\.io/revision
  
# see what would happen with a rollback:
kubectl rollout undo deploy/vweb --dry-run

# then start a rollback to revision 2:
kubectl rollout undo deploy/vweb --to-revision=2

# check the app—this should surprise you:
curl $(cat url.txt)
```

![图9.5](.\images\Figure9.5.png)
<center>图 9.5 Kubernetes为您管理部署，但如果您添加标签，它会有所帮助 Kubernetes manages rollouts for you, but it helps if you add labels to see what’s what.</center>

Hands up if you ran that exercise and got confused when you saw the final output shown in figure 9.6 (this is the exciting part of the chapter). My hand is up, and I already knew what was going to happen. This is why you need a consistent release
如果您在运行该练习时看到图9.6所示的最终输出时感到困惑(这是本章令人兴奋的部分)，请举手。我举起手来，我已经知道会发生什么。这就是为什么你需要持续的释放

![图9.6](.\images\Figure9.6.png)
<center>图 9.6 标签是一个关键的管理特性，但它们是由人类设置的，所以它们很容易出错 Labels are a key management feature, but they’re set by humans so they’re fallible.</center>

process, preferably one that is fully automated, because as soon as you start mixing approaches, you get confusing results. I rolled back to revision 2, and that should have reverted back to v1 of the app, judging by the labels on the ReplicaSets. But revision 2 was actually from the kubectl set image exercise in section 9.1, so the container image is v2, but the ReplicaSet label is v1.
过程，最好是完全自动化的过程，因为一旦开始混合方法，就会得到令人困惑的结果。我回滚到版本2，从ReplicaSets上的标签判断，应该已经恢复到应用程序的v1。但是修订2实际上来自9.1节中的kubectl集合图像练习，因此容器图像是v2，但ReplicaSet标签是v1。

​	You see that the moving parts of the release process are fairly simple: Deployments create and reuse ReplicaSets, scaling them up and down as required, and changes to ReplicaSets are recorded as rollouts. Kubernetes gives you control of the key factors in the rollout strategy, but before we move on to that, we’re going to look at releases which also involve a configuration change, because that adds another complicating factor.
您可以看到，发布过程的移动部分相当简单:部署创建和重用ReplicaSets，根据需要扩大和缩小它们，对ReplicaSets的更改被记录为rollout。Kubernetes让您可以控制部署策略中的关键因素，但在我们继续讨论之前，我们将看看还涉及配置更改的版本，因为这增加了另一个复杂因素。


​	In chapter 4, I talked about different approaches to updating the content of Config-Maps and Secrets, and the choice you make impacts your ability to roll back cleanly. The first approach is to say that configuration is mutable, so a release might include a ConfigMap change, which is an update to an existing ConfigMap object. But if your release is only a configuration change, then you have no record of that as a rollout and no option to roll back.
在第4章中，我讨论了更新Config-Maps和Secrets内容的不同方法，您所做的选择会影响您干净地回滚的能力。第一种方法是说配置是可变的，因此发布可能包括ConfigMap更改，这是对现有ConfigMap对象的更新。但是，如果您的发布只是一个配置更改，那么您就没有作为推出的记录，也没有回滚的选项。

​	**TRY IT NOW	Remove the existing Deployment so we have a clean history, then deploy a new version that uses a ConfigMap, and see what happens when you update the same ConfigMap.**
**现在就尝试删除现有的部署，这样我们就有了一个干净的历史记录，然后部署一个使用ConfigMap的新版本，看看当你更新相同的ConfigMap时会发生什么

```
# remove the existing app:
kubectl delete deploy vweb

# deploy a new version that stores content in config:
kubectl apply -f vweb/update/vweb-v3-with-configMap.yaml --record

# check the response:
curl $(cat url.txt)

# update the ConfigMap, and wait for the change to propagate:
kubectl apply -f vweb/update/vweb-configMap-v31.yaml --record
sleep 120

# check the app again:
curl $(cat url.txt)

# check the rollout history:
kubectl rollout history deploy/vweb
```

As you see in figure 9.7, the update to the ConfigMap changes the behavior of the app, but it’s not a change to the Deployment, so there is no revision to roll back to if the configuration change causes an issue.
如图9.7所示，对ConfigMap的更新改变了应用程序的行为，但它不是对Deployment的更改，因此如果配置更改导致问题，也没有回滚的修订。


​	This is the hot reload approach, which works nicely if your apps support it, precisely because a configuration-only change doesn’t require a rollout. The existing Pods and containers keep running, so there’s no risk of service interruption. The cost is the loss of the rollback option, and you’ll have to decide whether that’s more important than a hot reload.
这就是热重载方法，如果您的应用程序支持它，那么它工作得很好，因为仅配置的更改不需要rollout。现有的pod和容器保持运行，因此不存在服务中断的风险。代价是失去回滚选项，您必须决定这是否比热重载更重要。

​	Your alternative is to consider all ConfigMaps and Secrets as immutable, so you include some versioning scheme in the object name and never update a config object once it’s created. Instead you create a new config object with a new name and release it along with an update to your Deployment, which references the new config object.
您的替代方案是将所有configmap和Secrets视为不可变的，因此在对象名称中包含一些版本控制方案，并且在配置对象创建后永远不要更新它。相反，您可以创建一个具有新名称的新配置对象，并将其与对Deployment的更新一起发布，后者引用了新的配置对象。

​	**TRY IT NOW	Deploy a new version of the app with an immutable config, so you can compare the release process.**
**现在就尝试使用不可更改的配置部署应用的新版本，这样你就可以比较发布过程

```
# remove the old Deployment:
kubectl delete deploy vweb

# create a new Deployment using an immutable config:
kubectl apply -f vweb/update/vweb-v4-with-configMap.yaml --record

# check the output:
curl $(cat url.txt)

# release a new ConfigMap and updated Deployment:
kubectl apply -f vweb/update/vweb-v41-with-configMap.yaml --record

# check the output again:
curl $(cat url.txt)

# the update is a full rollout:
kubectl rollout history deploy/vweb

# so you can rollback:
kubectl rollout undo deploy/vweb
curl $(cat url.txt)
```

![图9.7](.\images\Figure9.7.png)
<center>图 9.7 配置更新可能会改变应用程序的行为，但不会记录滚出 Configuration updates might change app behavior but without recording a rollout.</center>

Figure 9.8 shows my output, where the config update is accompanied by a Deployment update, which preserves the rollout history and enables the rollback.
图9.8显示了我的输出，其中配置更新伴随着Deployment更新，后者保留了滚出历史并启用了回滚。

![图9.8](.\images\Figure9.8.png)
<center>图 9.8 不可变配置保留了滚出历史，但它意味着每次配置更改都要滚出 An immutable config preserves rollout history, but it means a rollout for every  configuration change.</center>

Kubernetes doesn’t really care which approach you take, and your choice will partly depend on who owns the configuration in your organization. If the project team also owns deployment and configuration, then you might prefer mutable config objects to simplify the release process and the number of objects to manage. If a separate team owns the configuration, then the immutable approach will be better because they can deploy new config objects ahead of the release. The scale of your apps will affect the decision, too: at a high scale, you may prefer to reduce the number of app deployments and rely on mutable configuration.
Kubernetes并不真正关心您采用哪种方法，您的选择在一定程度上取决于组织中谁拥有配置。如果项目团队还拥有部署和配置，那么您可能更喜欢可变配置对象，以简化发布过程和要管理的对象数量。如果一个单独的团队拥有配置，那么不可变的方法会更好，因为他们可以在发布之前部署新的配置对象。应用程序的规模也会影响决策:在规模大的情况下，你可能更倾向于减少应用程序部署的数量，并依赖可变配置。

​	There’s a cultural impact to this decision, because it frames how application releases are perceived—as everyday events that are no big deal, or as something slightly scary that is to be avoided as much as possible. In the container world, releases should be trivial events that you’re happy to do with minimal ceremony as soon as they’re needed. Testing and tweaking your release strategy will go a long way to giving you that confidence.
这个决定有文化上的影响，因为它框定了应用程序发布是如何被感知的——把它看作是没什么大不了的日常事件，或者看作是要尽可能避免的有点可怕的事情。在容器世界中，发布应该是一些微不足道的事件，只要需要，您就会乐于以最小的仪式来完成。测试和调整你的发行策略会给你带来很大的信心。

## 9.3 为 Deployments 配置滚动更新

Deployments support two update strategies: RollingUpdate is the default and the one we’ve used so far, and the other is Recreate. You know how rolling updates work—by scaling down the old ReplicaSet while scaling up the new ReplicaSet, which provides service continuity and the ability to stagger the update over a longer period. The Recreate strategy gives you neither of those. It still uses ReplicaSets to implement changes, but it scales down the previous set to zero before scaling up the replacement. Listing 9.2 shows the Recreate strategy in a Deployment spec. It’s just one setting, but it has a significant impact.
部署支持两种更新策略:RollingUpdate是默认的，也是我们目前使用的一种，另一种是rebuild。您知道滚动更新是如何工作的——通过缩小旧的ReplicaSet，同时扩大新的ReplicaSet，这提供了服务连续性和在较长时间内错开更新的能力。而rebuild策略则不提供这两种情况。它仍然使用ReplicaSets来实现更改，但是在扩展替换之前，它将先前的设置缩小到零。清单9.2显示了部署规范中的rebuild策略。这只是一个设置，但它具有重大影响。

**Listing 9.2	vweb-recreate-v2.yaml, a Deployment using the recreate update strategy**
**清单9.2 vweb-recreate-v2。yaml，一个使用重建更新策略**的部署

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vweb
spec:
  replicas: 3
  strategy:                                # This is the update strategy.
    type: Recreate                         # Recreate is the alternative to the
                                           # default strategy, RollingUpdate.
  # selector & Pod spec follow
```

When you deploy this, you’ll see it’s just a normal app with a Deployment, a ReplicaSet, and some Pods. If you look at the details of the Deployment, you’ll see it uses the Recreate update strategy, but that has an effect only when the Deployment is updated.
当你部署这个时，你会看到它只是一个普通的应用程序，有一个Deployment，一个ReplicaSet和一些Pods。如果查看Deployment的详细信息，您将看到它使用了rebuild更新策略，但这仅在更新Deployment时才会起作用。

​	**TRY IT NOW	Deploy the app from listing 9.2, and explore the objects. This is just like a normal Deployment.**
**TRY IT now部署清单9.2中的应用程序，并浏览对象。这就像一个正常的部署

```
# delete the existing app:
kubectl delete deploy vweb

# deploy with the Recreate strategy:
kubectl apply -f vweb-strategies/vweb-recreate-v2.yaml

# check the ReplicaSets:
kubectl get rs -l app=vweb

# test the app:
curl $(cat url.txt)

# look at the details of the Deployment:
kubectl describe deploy vweb
```

As shown in Figure 9.9, this is a new deployment of the same old web app, using version 2 of the container image. There are three Pods, they’re all running, and the app works as expected—so far so good.
如图9.9所示，这是同一个旧web应用程序的新部署，使用容器映像的版本2。有三个pod，它们都在运行，应用程序按预期运行——到目前为止还不错

![图9.9](.\images\Figure9.9.png)
<center>图 9.9 在你发布更新之前，重新创建更新策略不会影响行为	The Recreate update strategy doesn’t affect behavior until you release an update.</center>

This configuration is dangerous, though, and one you should use only if different versions of your app can’t coexist—something like a database schema update, where you need to be sure that only one version of your app connects to the database. Even in that case, you have better options, but if you have a scenario that definitely needs this approach, then you’d better be sure you test all your updates before you go live. If you deploy an update where the new Pods fail, you won’t know that until your old Pods have all been terminated, and your app will be completely unavailable.
不过，这种配置很危险，只有在应用程序的不同版本不能共存时才应该使用这种配置——就像数据库模式更新一样，需要确保只有一个版本的应用程序连接到数据库。即使在这种情况下，您也有更好的选择，但如果您的场景确实需要这种方法，那么您最好确保在发布之前测试所有更新。如果你部署了一个更新，而新Pods失败了，你会直到旧Pods全部被终止才知道，你的应用将完全不可用。

​	**TRY IT NOW	Version 3 of the web app is ready to deploy. It’s broken, as you’ll see when the app goes offline because no Pods are running.**
**尝试它现在版本3的web应用程序已经准备好部署。当应用程序离线时，你会看到它坏了，因为没有Pods在运行

```
# deploy the updated Pod spec:
kubectl apply -f vweb-strategies/vweb-recreate-v3.yaml

# check the status, with a time limit for updates:
kubectl rollout status deploy/vweb --timeout=2s

# check the ReplicaSets:
kubectl get rs -l app=vweb

# check the Pods:
kubectl get pods -l app=vweb

# test the app–this will fail:
curl $(cat url.txt)
```

You’ll see in this exercise that Kubernetes happily takes your app offline, because that’s what you’ve requested. The Recreate strategy creates a new ReplicaSet with the updated Pod template, then scales down the previous ReplicaSet to zero and scales up the new ReplicaSet to three. The new image is broken, so the new Pods fail, and there’s nothing to respond to requests, as you see in figure 9.10.
在这个练习中，你会看到Kubernetes很乐意让你的应用脱机，因为这是你所请求的。再造策略使用更新后的Pod模板创建一个新的ReplicaSet，然后将先前的ReplicaSet缩小到0，并将新的ReplicaSet扩大到3。新的映像被破坏了，所以新的Pods失败了，没有任何东西可以响应请求，如图9.10所示。

![图9.10](.\images\Figure9.10.png)
<center>图 9.10 如果新的Pod规格被打破，rebuild策略会愉快地关闭你的应用 The Recreate strategy happily takes down your app if the new Pod spec is broken.</center>

Now that you’ve seen it, you should probably try to forget about the Recreate strategy. In some scenarios, it might seem attractive, but when it does, you should still consider alternative options, even if it means looking again at your architecture. The wholesale takedown of your application is going to cause downtime, and probably more downtime than you plan for.
现在您已经看到了它，您可能应该尝试忘记再造策略。在某些情况下，它可能看起来很有吸引力，但当它出现时，您仍然应该考虑替代选项，即使这意味着重新查看您的体系结构。应用程序的大规模关闭将导致停机，而且停机时间可能比您计划的要长。

​	Rolling updates are the default because they guard against downtime, but even then, the default behavior is quite aggressive. For a production release, you’ll want to tune a few settings that set the speed of the release and how it gets monitored. As part of the rolling update spec, you can add options that control how quickly the new ReplicaSet is scaled up and how quickly the old ReplicaSet is scaled down, using the following two values:
现在您已经看到了它，您可能应该尝试忘记再造策略。在某些情况下，它可能看起来很有吸引力，但当它出现时，您仍然应该考虑替代选项，即使这意味着重新查看您的体系结构。应用程序的大规模关闭将导致停机，而且停机时间可能比您计划的要长。

- maxUnavailable is the accelerator for scaling down the old ReplicaSet. It defines how many Pods can be unavailable during the update, relative to the desired Pod count. You can think of it as the batch size for terminating Pods in the old ReplicaSet. In a Deployment of 10 Pods, setting this to 30%  means three Pods will be terminated immediately.
- maxSurge is the accelerator for scaling up the new ReplicaSet. It defines how many extra Pods can exist, over the desired replica count, like the batch size for creating Pods in the new ReplicaSet. In a Deployment of 10, setting this to 40% will create four new Pods.

- maxUnavailable是用于缩小旧ReplicaSet的加速器。它定义了在更新期间，相对于所需的Pod计数，可以有多少Pod不可用。您可以把它看作是在旧的ReplicaSet中终止pod的批处理大小。在10个pod的部署中，将此设置为30%意味着将立即终止3个pod。
- maxSurge是扩展新ReplicaSet的加速器。它定义了在期望的副本计数上可以存在多少额外的pod，就像在新的ReplicaSet中创建pod的批处理大小一样。在10个部署中，将此设置为40%将创建4个新pod。

Nice and simple, except both settings are used during a rollout, so you have a seesaw effect. The new ReplicaSet is scaled up until the Pod count is the desired replica count plus the maxSurge value, and then the Deployment waits for old Pods to be removed. The old ReplicaSet is scaled down to the desired count minus the maxUnavailable count, then the Deployment waits for new Pods to reach the ready state. You can’t set both values to zero because that means nothing will change. Figure 9.11 shows how you can combine the settings to prefer a create-then-remove, or a remove-then-create, or a remove-and-create approach to new releases.
很好，很简单，除了这两种设置都是在推出时使用的，所以你有一个跷跷板的效果。新的ReplicaSet被放大，直到Pod计数是所需的副本计数加上maxSurge值，然后部署等待旧的Pod被删除。旧的ReplicaSet被缩小到所需的计数减去maxUnavailable计数，然后Deployment等待新Pods达到就绪状态。你不能把两个值都设为0，因为这意味着什么都不会改变。图9.11显示了如何组合这些设置，以便对新版本使用先创建再删除、先删除再创建或先删除再创建的方法。

​	You can tweak these settings for a faster rollout if you have spare compute power in your cluster. You can also create additional Pods over your scale setting, but that’s riskier if you have a problem with the new release. A slower rollout is more conservative: it uses less compute and gives you more opportunity to discover any issues, but it reduces the overall capacity of your app during the release. Let’s see how these look, first by fixing our broken app with a conservative rollout.
如果您的集群中有空闲的计算能力，您可以调整这些设置以更快地推出。您还可以在您的比例设置上创建额外的pod，但如果您对新版本有问题，那么这样做的风险更大。缓慢的发布更保守:它使用更少的计算，给你更多的机会发现任何问题，但它在发布期间降低了应用程序的整体容量。让我们来看看这些是什么样子，首先通过保守的推出来修复我们的坏应用程序。


​	**TRY IT NOW	Revert back to the working version 2 image, using maxSurge=1 and maxUnavailable=0 in the RollingUpdate strategy.**
**尝试现在恢复到工作版本2的映像，使用maxSurge=1和maxUnavailable=0在RollingUpdate策略


```
# update the Deployment to use rolling updates and the v2 image:
kubectl apply -f vweb-strategies/vweb-rollingUpdate-v2.yaml

# check the Pods for the app:
kubectl get po -l app=vweb

# check the rollout status:
kubectl rollout status deploy/vweb

# check the ReplicaSets:
kubectl get rs -l app=vweb

# test the app:
curl $(cat url.txt)
```

![图9.11](.\images\Figure9.11.png)
<center>图 9.11 正在进行部署更新，使用不同的推出选项 Deployment updates in progress, using different rollout options</center>

In this exercise, the new Deployment spec changed the Pod image back to version 2, and it also changed the update strategy to a rolling update. You can see in figure 9.12 that the strategy change is made first, and then the Pod update is made in line with the new strategy, creating one new Pod at a time.
在本练习中，新的部署规范将Pod映像更改回版本2，并将更新策略更改为滚动更新。你可以在图9.12中看到，首先进行了策略更改，然后根据新策略进行Pod更新，一次创建一个新的Pod。

![图9.12](.\images\Figure9.12.png)
<center>图 9.12 如果Pod模板匹配新规范，部署更新将使用现有的ReplicaSet Deployment updates will use an existing ReplicaSet if the Pod template matches the new spec.</center>

You’ll need to work fast to see the rollout in progress in the previous exercise, because this simple app starts quickly, and as soon as one new Pod is running, the rollout continues with another new Pod. You can control the pace of the rollout with the following two fields in the Deployment spec:
在前面的练习中，您需要快速工作以查看推出的进度，因为这个简单的应用程序很快启动，只要一个新的Pod正在运行，就会继续使用另一个新Pod进行推出。在部署规范中，您可以通过以下两个字段来控制部署的速度:

- minReadySeconds adds a delay where the Deployment waits to make sure new Pods are stable. It specifies the number of seconds the Pod should be up with no containers crashing before it’s considered to be successful. The default is zero, which is why new Pods are created quickly during rollouts.
- progressDeadlineSeconds specifies the amount of time a Deployment update can run before it’s considered as failing to progress. The default is 600 seconds, so if an update is not completed within 10 minutes, it’s flagged as not progressing.
- minReadySeconds增加了一个延迟，即部署等待以确保新pod稳定。它指定了Pod在没有容器崩溃之前应该启动的秒数。默认值为0，这就是为什么在推出过程中会快速创建新pod的原因。
- progressdeadlinesecseconds指定部署更新在被视为进展失败之前可以运行的时间。默认值是600秒，因此如果更新没有在10分钟内完成，它将被标记为未进行。


Monitoring how long the release takes sounds useful, but as of Kubernetes 1.19, exceeding the deadline doesn’t actually affect the rollout—it just sets a flag on the Deployment. Kubernetes doesn’t have an automatic rollback feature for failed rollouts, but when that feature does come, it will be triggered by this flag. Waiting and checking a Pod for failed containers is a fairly blunt tool, but it’s better than having no checks at all, and you should consider having minReadySeconds specified in all your Deployments.
监视发布需要多长时间听起来很有用，但从Kubernetes 1.19开始，超过最后期限实际上并不会影响部署—它只是在部署上设置了一个标志。Kubernetes没有自动回滚功能，但是当该功能出现时，它将由这个标志触发。等待和检查Pod中失败的容器是一个相当生硬的工具，但这比根本不检查要好，您应该考虑在所有部署中指定minReadySeconds。

​	These safety measures are useful to add to your Deployment, but they don’t really help with our web app because the new Pods always fail. We can make this Deployment safe and keep the app online using a rolling update. The next version 3 update sets both maxUnavailable and maxSurge to 1. Doing so has the same effect as the default values (each 25%), but it’s clearer to use exact values in the spec, and Pod counts are easier to work with than percentages in small deployments.
这些安全措施可以添加到你的部署中，但它们对我们的web应用没有真正的帮助，因为新的Pods总是失败。我们可以使此部署安全，并使用滚动更新保持应用程序在线。下一个版本3更新将maxUnavailable和maxSurge设置为1。这样做与默认值(每个25%)具有相同的效果，但在规范中使用确切的值更清楚，并且在小型部署中Pod计数比百分比更容易处理。

​	**TRY IT NOW	Deploy the version 3 update again. It will still fail, but by using a RollingUpdate strategy, it doesn’t take the app offline.**
**现在就尝试再次部署版本3更新。它仍然会失败，但通过使用RollingUpdate策略，它不会让应用程序离线

```
# update to the failing container image:
kubectl apply -f vweb-strategies/vweb-rollingUpdate-v3.yaml

# check the Pods:
kubectl get po -l app=vweb

# check the rollout:
kubectl rollout status deploy/vweb

# see the scale in the ReplicaSets:
kubectl get rs -l app=vweb

# test the app:
curl $(cat url.txt)
```

When you run this exercise, you’ll see the update never completes, and the Deployment is stuck with two ReplicaSets having a desired Pod count of two, as shown in figure 9.13. The old ReplicaSet won’t scale down any further because the Deployment has maxUnavailable set to 1; it has already been scaled down by 1 and no new Pods will become ready to continue the rollout. The new ReplicaSet won’t scale up anymore because maxSurge is set to 1, and the total Pod count for the Deployment has been reached.
当您运行这个练习时，您将看到更新永远不会完成，并且部署卡住了两个ReplicaSets，其Pod数为2，如图9.13所示。旧的ReplicaSet将不会进一步缩小，因为部署已经将maxUnavailable设置为1;它的规模已经缩小了1，不会有新的pod准备继续推出。新的ReplicaSet将不再扩展，因为maxSurge被设置为1，并且已经达到了部署的总Pod数。

​	If you check back on the new Pods in a few minutes, you’ll see they’re in the state CrashLoopBackoff. Kubernetes keeps restarting failed Pods by creating replacement containers, but it adds a pause between each restart so it doesn’t choke the CPU on the node. That pause is the backoff time, and it increases exponentially—10 seconds for the first restart, then 20 seconds, and then 40 seconds, up to a maximum of 5 minutes. These version 3 Pods will never restart successfully, but Kubernetes will keep trying.
如果您在几分钟后检查新pod，您将看到它们处于CrashLoopBackoff状态。Kubernetes通过创建替换容器来重新启动失败的Pods，但它在每次重新启动之间添加了暂停，这样就不会阻塞节点上的CPU。这个暂停就是后退时间，它呈指数级增加——第一次重新启动时为10秒，然后是20秒，然后是40秒，最多5分钟。这些第三版pod将永远无法成功重启，但Kubernetes将继续尝试。

​	Deployments are the controllers you use the most, and it’s worth spending time working through the update strategy and timing settings to be sure you understand
部署是您使用最多的控制器，值得花时间研究更新策略和定时设置，以确保您理解

![图9.13](.\images\Figure9.13.png)
<center>图 9.13 失败的更新不会自动回滚或暂停;他们只是一直在努力 Failed updates don’t automatically roll back or pause; they just keep trying.</center>

the impact for your apps. DaemonSets and StatefulSets also have rolling update functionality, and because they have different ways of managing their Pods, they have different approaches to rollouts, too.
对你的应用的影响。DaemonSets和StatefulSets也有滚动更新功能，因为它们有不同的管理pod的方式，所以它们也有不同的推出方法。
## 9.4 DaemonSets 和 StatefulSets 中的滚动更新

DaemonSets and StatefulSets have two update strategies available. The default is RollingUpdate, which we’ll explore in this section. The alternative is OnDelete, which is for situations when you need close control over when each Pod is updated. You deploy the update, and the controller watches Pods, but it doesn’t terminate any existing Pods. It waits until they are deleted by another process, and then it replaces them with Pods from the new spec.
DaemonSets和StatefulSets有两个可用的更新策略。默认的是RollingUpdate，我们将在本节中探讨它。另一种方法是OnDelete，它用于需要密切控制每个Pod何时更新的情况。你部署更新，控制器会监视pod，但它不会终止任何现有的pod。它会一直等待，直到它们被另一个进程删除，然后用新规范中的pod替换它们。

​	This isn’t quite as pointless as it sounds, when you think about the use cases for these controllers. You may have a StatefulSet where each Pod needs to have flushed data to disk before it’s removed, and you can have an automated process to do that. You may have a DaemonSet where each Pod needs to be disconnected from a hardware component, so it’s free for the next Pod to use. These are rare cases, but the OnDelete strategy lets you take ownership of when Pods are deleted and still have Kubernetes automatically create replacements.
DaemonSets和StatefulSets有两个可用的更新策略。默认的是RollingUpdate，我们将在本节中探讨它。另一种方法是OnDelete，它用于需要密切控制每个Pod何时更新的情况。你部署更新，控制器会监视pod，但它不会终止任何现有的pod。它会一直等待，直到它们被另一个进程删除，然后用新规范中的pod替换它们。

​	We’ll focus on rolling updates in this section, and for that we’ll deploy a version of the to-do list app, which runs the database in a StatefulSet, the web app in a Deployment, and a reverse proxy for the web app in a DaemonSet.
在本节中，我们将重点关注滚动更新，为此，我们将部署一个版本的待办事项列表应用程序，它在StatefulSet中运行数据库，在Deployment中运行web应用程序，在DaemonSet中运行web应用程序的反向代理。

​	**TRY IT NOW	The to-do app runs across six Pods, so start by clearing the existing apps to make room. Then deploy the app, and test that it works correctly.**
**现在就试一试，待办事项应用程序可以在6个pod中运行，所以首先清理现有的应用程序以腾出空间。然后部署应用程序，并测试它是否正常工作


```
# remove all of this chapter’s apps:
kubectl delete all -l kiamol=ch09

# deploy the to-do app, database, and proxy:
kubectl apply -f todo-list/db/ -f todo-list/web/ -f todo-list/proxy/

# get the app URL:
kubectl get svc todo-proxy -o
  jsonpath='http://{.status.loadBalancer.ingress[0].*}:8091'

# browse to the app, add an item, and check that it’s in the list
```

This is just setting us up for the updates. You should now have a working app where you can add items and see the list. My output is shown in figure 9.14.
这只是为更新做准备。你现在应该有一个工作的应用程序，你可以添加项目，并看到列表。我的输出如图9.14所示。

![图9.14](.\images\Figure9.14.png)
<center>图 9.14 运行带有各种免费控制器的to-do应用程序	Running the to-do app with a gratuitous variety of controllers</center>

The first update is for the DaemonSet, where we’ll be rolling out a new version of the Nginx proxy image. DaemonSets run a single Pod on all (or some) of the nodes in the cluster, and with a rolling update, you have no surge option. During the update, nodes will never run two Pods, so this is always a delete-then-remove strategy. You can add the maxUnavailable setting to control how many nodes are updated in parallel, but if you take down multiple Pods, you’ll be running at reduced capacity until the replacements are ready.
第一个更新是DaemonSet，在那里我们将推出一个新版本的Nginx代理映像。DaemonSets在集群中的所有(或部分)节点上运行一个Pod，通过滚动更新，您没有激增选项。在更新期间，节点永远不会运行两个pod，因此这始终是一种先删除再删除的策略。您可以添加maxUnavailable设置来控制并行更新的节点数量，但是如果您关闭多个pod，您将以较低的容量运行，直到替换准备就绪。

​	We’ll update the proxy using a maxUnavailable setting of 1, and a minReadySeconds setting of 90. On a single-node lab cluster, the delay won’t have any effect— there’s only one Pod on one node to replace. On a larger cluster, it would mean replacing one Pod at a time and waiting 90 seconds for the Pod to prove it’s stable before moving on to the next.
我们将使用maxUnavailable设置为1和minReadySeconds设置为90来更新代理。在单节点实验室集群上，延迟不会产生任何影响——一个节点上只有一个Pod需要替换。在更大的集群中，这意味着一次更换一个Pod，并等待90秒，让Pod证明它是稳定的，然后再转移到下一个Pod。

​	**TRY IT NOW	Start the rolling update of the DaemonSet. On a single-node cluster, a short outage will occur while the replacement Pod starts.**
**TRY IT now启动DaemonSet的滚动更新。在单节点集群中，更换Pod启动时会出现短暂停机

```
# deploy the DaemonSet update:
kubectl apply -f todo-list/proxy/update/nginx-rollingUpdate.yaml

# watch the Pod update:
kubectl get po -l app=todo-proxy --watch

# Press ctrl-c when the update completes
```

The watch flag in kubectl is useful for monitoring changes—it keeps looking at an object and prints an update line whenever the state changes. In this exercise you’ll see that the old Pod is terminated before the new one is created, which means the app has downtime while the new Pod starts up. Figure 9.15 shows I had one second of downtime in my release.
kubectl中的watch标志对于监视更改非常有用——它会一直查看对象，并在状态更改时打印更新行。在本练习中，您将看到旧的Pod在创建新Pod之前被终止，这意味着新Pod启动时应用程序有停机时间。图9.15显示了我在发布中有1秒的停机时间。

![图9.15](.\images\Figure9.15.png)
<center>图 9.15 DaemonSets更新通过删除现有Pod之前创建一个替换 DaemonSets update by removing the existing Pod before creating a replacement.</center>

A multinode cluster wouldn’t have any downtime because the Service sends traffic only to Pods that are ready, and only one Pod at a time gets updated, so the other Pods are always available. You will have reduced capacity, though, and if you tune a faster rollout with a higher maxUnavailable setting, that means a greater reduction in capacity as more Pods are updated in parallel. That’s the only setting you have for DaemonSets, so it’s a simple choice between manually controlling the update by deleting Pods or having Kubernetes roll out the update by a specified number of Pods in parallel.
多节点集群不会有任何停机时间，因为服务只向准备就绪的Pod发送流量，并且一次只更新一个Pod，因此其他Pod始终可用。但是，您的容量将会减少，如果您使用更高的maxUnavailable设置来优化更快的推出，这意味着随着更多pod并行更新，容量将会减少更多。这是你对DaemonSets的唯一设置，所以这是一个简单的选择，通过删除pod手动控制更新或让Kubernetes通过指定数量的pod并行推出更新。

​	StatefulSets are more interesting, although they have only one option to configure the rollout. Pods are managed in order by the StatefulSet, which also applies to updates—the rollout proceeds backward from the last Pod in the set down to the first. That’s especially useful for clustered applications where Pod 0 is the primary, because it validates the update on the secondaries first.
statefulset更有趣，尽管它们只有一个选项来配置rollout。Pod由StatefulSet按顺序管理，这也适用于更新——从集合中的最后一个Pod向后推出到第一个Pod。这对于Pod 0是主节点的集群应用程序特别有用，因为它首先验证从节点上的更新。

​	There is no maxSurge or maxUnavailable setting for StatefulSets. The update is always by one Pod at a time. Your configuration option is to define how many Pods should be updated in total, using the partition setting. This setting defines the cutoff point where the rollout stops, and it’s useful for performing a staged rollout of a stateful app. If you have five replicas in your set and your spec includes partition=3, then only Pod 4 and Pod 3 will be updated; Pods 0, 1, and 2 are left running the previous spec.
StatefulSets没有maxSurge或maxUnavailable设置。更新总是由一个Pod在一个时间。您的配置选项是使用分区设置定义总共应该更新多少pod。这个设置定义了滚出停止的截止点，它对于执行有状态应用的分阶段滚出非常有用。如果你的set中有五个副本，并且spec中包含partition=3，那么只有Pod 4和Pod 3将被更新;pod 0、1和2仍然运行前一个规范。

​	**TRY IT NOW	Deploy a partitioned update to the database image in the StatefulSet, which stops after Pod 1, so Pod 0 doesn’t get updated.**
**TRY IT now在StatefulSet中部署一个分区更新到数据库映像，它在Pod 1之后停止，所以Pod 0不会被更新

```
# deploy the update:
kubectl apply -f todo-list/db/update/todo-db-rollingUpdate-
  partition.yaml

# check the rollout status:
kubectl rollout status statefulset/todo-db

# list the Pods, showing the image name and start time:
kubectl get pods -l app=todo-db -o=custom-columns=NAME:.metadata.name,
  IMAGE:.spec.containers[0].image,START_TIME:.status.startTime

# switch the web app to read-only mode, so it uses the secondary
# database:
kubectl apply -f todo-list/web/update/todo-web-readonly.yaml

# test the app—the data is there, but now it’s read-only
```

This exercise is a partitioned update that rolls out a new version of the Postgres container image, but only to the secondary Pods, which is a single Pod in this case, as shown in figure 9.16. When you use the app in read-only mode, you’ll see that it connects to the updated secondary, which still contains the replicated data from the previous Pod.
这个练习是一个分区更新，它推出了Postgres容器映像的新版本，但只对次要Pod进行更新，在本例中是单个Pod，如图9.16所示。当你在只读模式下使用应用程序时，你会看到它连接到更新后的备用程序，备用程序仍然包含从上一个Pod复制的数据。

![图9.16](.\images\Figure9.16.png)
<center>图 9.16 对StatefulSets的分区更新允许您更新次要文件，而保持主文件不变 Partitioned updates to StatefulSets let you update secondaries and leave the primary unchanged.</center>

This rollout is complete, even though the Pods in the set are running from different specs. For a data-heavy application in a StatefulSet, you may have a suite of verification jobs that you need to run on each updated Pod before you’re happy to continue the rollout, and a partitioned update lets you do that. You can manually control the pace of the release by running successive updates with decreasing partition values, until you remove the partition altogether in the final update to finish the set.
这是完整的展示，即使在一组pod运行不同的规格。对于StatefulSet中的数据量大的应用程序，您可能需要在每个更新的Pod上运行一组验证作业，然后才愿意继续推出，而分区更新可以让您做到这一点。您可以通过运行分区值递减的连续更新来手动控制发布的节奏，直到在最后的更新中完全删除分区以完成集合。

​	**TRY IT NOW	Deploy the update to the database primary. This spec is the same as the previous exercise but with the partition setting removed.**
**现在就尝试将更新部署到数据库主数据库。此规范与前面的练习相同，但删除了分区设置

```
# apply the update:
kubectl apply -f todo-list/db/update/todo-db-rollingUpdate.yaml

# check its progress:
kubectl rollout status statefulset/todo-db

# Pods should now all have the same spec:
kubectl get pods -l app=todo-db -o=custom-
  columns=NAME:.metadata.name,IMAGE:.spec.containers[0].image,START
  _TIME:.status.startTime
  # reset the web app back to read-write mode:
  kubectl apply -f todo-list/web/todo-web.yaml
  
  # test that the app works and is connected to the updated primary Pod
```

You can see my output in figure 9.17, where the full update has completed and the primary is using the same updated version of Postgres as the secondary. If you’ve done updates to replicated databases before, you’ll know that this is about as simple as it gets—unless you’re using a managed database service, of course.
你可以在图9.17中看到我的输出，其中完整的更新已经完成，主服务器使用与备用服务器相同的更新版本的Postgres。如果您以前对复制的数据库进行过更新，您就会知道这非常简单——当然，除非您使用的是托管数据库服务。

![图9.17](.\images\Figure9.17.png)
<center>图 9.17 使用未分区的更新完成StatefulSet滚出 Completing the StatefulSet rollout, with an update that is not partitioned</center>

Rolling updates are the default for Deployments, DaemonSets, and StatefulSets, and they all work in broadly the same way: gradually replacing Pods running the previous application spec with Pods running the new spec. The actual details differ because the controllers work in different ways and have different goals, but they impose the same requirement on your app: it needs to work correctly when multiple versions are live. That’s not always possible, and there are alternative ways to deploy app updates in Kubernetes.
滚动更新是部署、DaemonSets和StatefulSets的默认设置，它们的工作方式大致相同:逐渐用运行新规范的Pods取代运行前一个应用程序规范的Pods。实际的细节不同，因为控制器的工作方式不同，目标也不同，但它们对应用程序的要求是相同的:当多个版本处于活动状态时，它需要正确工作。这并不总是可行的，还有其他方法可以在Kubernetes中部署应用程序更新。

## 9.5 理解发布策略

Take the example of a web application. A rolling update is great because it lets each Pod close gracefully when all its client requests are dealt with, and the rollout can be as fast or as conservative as you like. The practical side of the rollout is simple, but you have to consider the user experience (UX) side, too.
以一个web应用程序为例。滚动更新非常棒，因为它可以让每个Pod在处理完所有客户端请求后优雅地关闭，并且可以按照您的喜好快速或保守地推出。推出的实际方面很简单，但您也必须考虑用户体验(UX)方面。

​	Application updates might well change the UX—with a different design, new features, or an updated workflow. Any changes like that will be pretty strange for the user if they see the new version during a rollout, then refresh and find themselves with the old version, because the requests have been served by Pods running different versions of the app.
应用程序更新很可能会改变ux——使用不同的设计、新功能或更新的工作流。对于用户来说，如果他们在推出过程中看到新版本，然后刷新并发现自己使用了旧版本，那么任何这样的更改都会非常奇怪，因为这些请求是由运行不同版本应用程序的Pods提供的。

​	The strategies to deal with that go beyond the RollingUpdate spec in your controllers. You can set cookies in your web app to link a client to a particular UX, and then use a more advanced traffic routing system to ensure users keep seeing the new version. When we cover that in chapter 15, you’ll see it introduces several more moving parts. For cases where that method is too complex or doesn’t solve the problem of dual running multiple versions, you can manage the release yourself with a blue-green deployment.
处理这种情况的策略超出了控制器中的RollingUpdate规范。您可以在web应用程序中设置cookie，将客户端链接到特定的用户体验，然后使用更高级的流量路由系统，以确保用户一直看到新版本。当我们在第15章讲到它时，你会看到它引入了更多的活动部分。对于这种方法过于复杂或不能解决双运行多个版本的问题的情况，您可以使用蓝绿部署自己管理发布。

​	Blue-green deployments are a simple concept: you have both the old and new versions of your app deployed at the same time, but only one version is active. You can flip a switch to choose which version is the active one. In Kubernetes, you can do that by updating the label selector in a Service to send traffic to the Pods in a different Deployment, as shown in figure 9.18.
蓝绿部署是一个简单的概念:你同时部署了应用程序的新版本和旧版本，但只有一个版本是活动的。您可以通过拨动开关来选择激活的版本。在Kubernetes中，你可以通过更新Service中的标签选择器来将流量发送到不同部署中的Pods，如图9.18所示。

​	You need to have the capacity in your cluster to run two complete copies of your app. If it’s a web or API component, then the new version should be using minimal
你需要在你的集群中有能力运行两个完整的应用程序副本。如果它是一个web或API组件，那么新版本应该使用最小的.


![图9.18](.\images\Figure9.18.png)
<center>图 9.18 在蓝绿部署中运行多个版本的应用程序，但只有一个是活的 You run multiple versions of the app in a blue-green deployment, but only one is live.</center>

memory and CPU because it’s not receiving any traffic. You switch between versions by updating the label selector for the Service, so the update is practically instant because all the Pods are running and ready to receive traffic. You can flip back and forth easily, so you can roll back a problem release without waiting for ReplicaSets to scale up and down.
内存和CPU，因为它没有接收任何流量。您可以通过更新服务的标签选择器来切换版本，因此更新实际上是即时的，因为所有Pods都在运行并准备接收流量。您可以轻松地来回翻转，因此您可以回滚有问题的版本，而无需等待ReplicaSets的伸缩。

​	Blue-green deployments are less sophisticated than rolling updates, but they’re simpler because of that. They can be a better fit for organizations moving to Kubernetes who have a history of big-bang deployments, but they’re a compute-intensive approach that requires multiple steps and doesn’t preserve the rollout history of your app. You should look to rolling updates as your preferred deployment strategy, but blue-green deployments are a good stepping-stone to use while you gain confidence.
蓝绿部署没有滚动更新那么复杂，但正因为如此，它们更简单。它们可能更适合那些迁移到Kubernetes的组织，这些组织有大规模部署的历史，但它们是一种计算密集型的方法，需要多个步骤，并且不能保存应用程序的推出历史。您应该将滚动更新作为首选的部署策略，但蓝绿部署是一个很好的跳板，可以在您获得信心的同时使用。


​	That’s all on rolling updates for now, but we will return to the concepts when we cover topics in production readiness, network ingress, and monitoring. We just need to tidy up the cluster now before going on to the lab.
以上是关于滚动更新的全部内容，但是当我们讨论生产准备、网络进入和监视等主题时，我们将回到这些概念。我们只需要在去实验室之前整理一下集群。

​	**TRY IT NOW	Remove all the objects created for this chapter.**
**现在就尝试删除为本章创建的所有对象

```
kubectl delete all -l kiamol=ch09
kubectl delete cm -l kiamol=ch09
kubectl delete pvc -l kiamol=ch09
```

## 9.6 实验室

We learned the theory of blue-green deployments in the previous section, and now in the lab, you’re going to make it happen. Working through this lab will help make it clear how selectors relate Pods to other objects and give you experience working with the alternative to rolling updates.
我们在前一节中学习了蓝绿部署的理论，现在在实验室中，您将实现它。通过本实验将有助于您清楚地了解选择器如何将Pods与其他对象关联起来，并为您提供使用滚动更新的替代方案的经验。

- The starting point is version 1 of the web app, which you can deploy from the lab/v1 folder.
- You need to create a blue-green deployment for version 2 of the app. The spec will be similar to the version 1 spec but using the :v2 container image.
- When you deploy your update, you should be able to flip between the version 1 and version 2 release just by changing the Service and without any updates to Pods.

- 起点是web应用程序的版本1，您可以从lab/v1文件夹部署。
- 你需要为应用程序的版本2创建一个蓝绿色的部署。规范将类似于版本1的规范，但使用:v2容器镜像。
- 当您部署更新时，您应该能够通过更改服务在版本1和版本2之间切换，而无需对Pods进行任何更新。

This is good practice in copying YAML files and trying to work out which fields you need to change. You can find my solution on GitHub: https://github.com/sixeyed/kiamol/blob/master/ch09/lab/README.md.
这是复制YAML文件并尝试确定需要更改哪些字段的良好实践。你可以在GitHub上找到我的解决方案:https://github.com/sixeyed/kiamol/blob/master/ch09/lab/README.md。