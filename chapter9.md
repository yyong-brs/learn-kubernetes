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

<b>现在就试试</b> 对 Deployment 应用更新，它使用相同的 Docker 镜像，但更改 Pod 的版本标签。这是对 Pod spec 的更改，因此它将创建一个新的 rollout。

```
# 通过 record 参数应用新的变更:
kubectl apply -f vweb/update/vweb-v11.yaml --record

# 检查 ReplicaSets 以及它们的标签:
kubectl get rs -l app=vweb --show-labels

# 检查当前 rollout 状态:
kubectl rollout status deploy/vweb

# 检查 rollout history:
kubectl rollout history deploy/vweb

# 根据 ReplicaSets 查看 rollout revision:
kubectl get rs -l app=vweb -o=custom-columns=NAME:.metadata.name,
  REPLICAS:.status.replicas,REVISION:.metadata.annotations.deployment
  \.kubernetes\.io/revision
```

我的输出如图 9.4 所示。添加 record 标志将 kubectl 命令保存为 rollout 详细信息，如果您的 YAML 文件具有标识名称，这将很有帮助。通常情况下，他们不会这样做，因为您将部署整个文件夹，所以Pod spec 中的版本号标签是一个有用的补充。然后，您需要一些笨拙的 JSONPath 来查找 rollout revision 和ReplicaSet之间的链接。

![图9.4](.\images\Figure9.4.png)
<center>图 9.4 Kubernetes 使用标签作为关键信息，额外的细节存储在 annotations 中.</center>

随着 Kubernetes 成熟度的提高，您将希望拥有一组包含在所有对象 spec 中的标准标签。标签和选择器是一个核心特性，您将一直使用它们来查找和管理对象。应用程序名称、组件名称和版本都是很好的标签，但是区分为方便而包含的标签和Kubernetes用于映射对象关系的标签是很重要的。

清单 9.1 显示了 Pod 标签和前面练习中 Deployment 的选择器。app 标签在选择器中使用，Deployment 使用该选择器来查找它的Pods。为了方便起见，Pod 还包含一个版本标签，但那不是选择器的一部分。如果是，则Deployment将链接到一个版本，因为一旦创建了Deployment就不能更改选择器。

> 清单9.1 vweb-v11。yaml，在 Pod spec 中有附加标签的 Deployment

```
spec:
  replicas: 3
  selector:
    matchLabels:
      app: vweb # 用于选择器.
  template:
    metadata:
      labels:
        app: vweb
        version: v1.1 # 多配置了一个 version 标签.
```

你需要事先仔细规划你的选择器，但你应该在 Pod spec 中添加任何你需要的标签，以使你的更新易于管理。Deployment 会保留多个replicaset(10是默认值)，并且名称中的 Pod 模板散列使得它们很难直接使用，即使在进行了几次更新之后也是如此。让我们看看我们部署的应用程序实际做了什么，然后在另一个 rollout 中查看ReplicaSets。

<b>现在就试试</b> 向web服务发起HTTP调用以查看响应，然后启动另一次更新并再次检查响应

```
# 因为将会使用 url 很多次，所以把它保存到本地文件中:
kubectl get svc vweb -o
  jsonpath='http://{.status.loadBalancer.ingress[0].*}:8090/v.txt' >
  url.txt
  
# 然后通过保存地址进行 HTTP 请求:
curl $(cat url.txt)

# 部署 v2 更新:
kubectl apply -f vweb/update/vweb-v2.yaml --record

# 再次检查响应:
curl $(cat url.txt)

# 检查 ReplicaSet 详细信息:
kubectl get rs -l app=vweb --show-labels
```

在本练习中，您将看到 replicaset 并不是易于管理的对象，因此需要使用标准化标签。通过检查 ReplicaSet 的标签，可以很容易地看到应用程序的哪个版本处于活动状态(如图9.5所示)，但标签只是文本字段，因此需要流程保障以确保它们是可靠的。

![图9.5](.\images\Figure9.5.png)
<center>图 9.5 Kubernetes 为您管理 rollouts，但如果您添加标签，它会有所帮助.</center>

rollout 确实有助于抽象 ReplicaSets 的细节，但它们的主要用途是管理发布。我们已经看到了 kubectl 的 rollout 历史，您还可以运行命令暂停正在进行的 rollout 或将部署回滚到较早的版本。一个简单的命令将回滚到以前的部署，但是如果您想回滚到特定的版本，则需要更多的JSONPath技巧来查找所需的版本。现在我们将看到这一点，并使用kubectl的一个非常方便的特性，该特性可以告诉您运行命令时会发生什么，而无需实际执行该命令。

<b>现在就试试</b> 检查应用的 rollout 历史记录，并尝试回滚到应用的v1

```
# 查看 revisions:
kubectl rollout history deploy/vweb

# 查看 ReplicaSets,带出 revisions 信息:
kubectl get rs -l app=vweb -o=custom-
  columns=NAME:.metadata.name,REPLICAS:.status.replicas,VERSION:.meta
  data.labels.version,REVISION:.metadata.annotations.deployment\.kube
  rnetes\.io/revision
  
# 看一下执行 rollout 会发生什么:
kubectl rollout undo deploy/vweb --dry-run

# 然后运行一个 rollback 到 revision 2:
kubectl rollout undo deploy/vweb --to-revision=2

# 检查 app—可能会惊到你:
curl $(cat url.txt)
```

如果您在运行该练习时看到图9.6所示的最终输出时感到困惑(这是本章令人兴奋的部分)，请举手。我举起手来，我已经知道会发生什么。这就是为什么你需要持续的发布。

![图9.6](.\images\Figure9.6.png)
<center>图 9.6 标签是一个关键的管理特性，但它们是由人类设置的，所以它们很容易出错</center>

过程，最好是完全自动化的过程，因为一旦开始混合方法，就会得到令人困惑的结果。我回滚到版本2，从ReplicaSets上的标签判断，应该已经恢复到应用程序的v1。但是修订2实际上来自9.1节中的kubectl集合镜像练习，因此容器镜像是v2，但ReplicaSet标签是v1。


您可以看到，发布过程的移动部分相当简单:部署创建和重用ReplicaSets，根据需要扩大和缩小它们，对ReplicaSets的更改被记录为rollout。Kubernetes让您可以控制部署策略中的关键因素，但在我们继续讨论之前，我们将看看还涉及配置更改的版本，因为这增加了另一个复杂因素。

在第4章中，我讨论了更新Config-Maps和Secrets内容的不同方法，您所做的选择会影响您干净地回滚的能力。第一种方法是说配置是可变的，因此发布可能包括ConfigMap更改，这是对现有ConfigMap对象的更新。但是，如果您的发布只是一个配置更改，那么您就没有作为rollout的记录，也没有回滚的选项。

<b>现在就试试</b>删除现有的部署，这样我们就有了一个干净的历史记录，然后部署一个使用ConfigMap的新版本，看看当你更新相同的ConfigMap时会发生什么

```
# 删除 app:
kubectl delete deploy vweb

# 部署一个用到 configmap 的新应用:
kubectl apply -f vweb/update/vweb-v3-with-configMap.yaml --record

# 检查响应:
curl $(cat url.txt)

# 更新 ConfigMap, 等待一段时间成功:
kubectl apply -f vweb/update/vweb-configMap-v31.yaml --record
sleep 120

# 检查 app :
curl $(cat url.txt)

# 检查 rollout history:
kubectl rollout history deploy/vweb
```

如图9.7所示，对ConfigMap的更新改变了应用程序的行为，但它不是对Deployment的更改，因此如果配置更改导致问题，也没有回滚的修订。

![图9.7](.\images\Figure9.7.png)
<center>图 9.7 配置更新可能会改变应用程序的行为，但不会记录 rollout.</center>

这就是热重载方法，如果您的应用程序支持它，那么它工作得很好，因为仅配置的更改不需要rollout。现有的pod和容器保持运行，因此不存在服务中断的风险。代价是失去回滚选项，您必须决定这是否比热重载更重要。

您的替代方案是将所有configmap和Secrets视为不可变的，因此在对象名称中包含一些版本控制方案，并且在配置对象创建后永远不要更新它。相反，您可以创建一个具有新名称的新配置对象，并将其与对Deployment的更新一起发布，后者引用了新的配置对象。

<b>现在就试试</b> 使用不可更改的配置部署应用的新版本，这样你就可以比较发布过程

```
# 删除 old Deployment:
kubectl delete deploy vweb

# 使用不可变配置创建一个新的部署:
kubectl apply -f vweb/update/vweb-v4-with-configMap.yaml --record

# 检查输出:
curl $(cat url.txt)

# 发布一个新的 ConfigMap 并更新 Deployment:
kubectl apply -f vweb/update/vweb-v41-with-configMap.yaml --record

# 检查输出:
curl $(cat url.txt)

# 更新将产生完整的 rollout:
kubectl rollout history deploy/vweb

# 因此可以执行回滚:
kubectl rollout undo deploy/vweb
curl $(cat url.txt)
```

图9.8显示了我的输出，其中配置更新伴随着Deployment更新，后者保留了 rollout 历史并启用了回滚。

![图9.8](.\images\Figure9.8.png)
<center>图 9.8 不可变配置保留了滚出历史，但它意味着每次配置更改都要 rollout.</center>


Kubernetes 并不真正关心您采用哪种方法，您的选择在一定程度上取决于组织中谁拥有配置。如果项目团队还拥有部署和配置，那么您可能更喜欢可变配置对象，以简化发布过程和要管理的对象数量。如果一个单独的团队拥有配置，那么不可变的方法会更好，因为他们可以在发布之前部署新的配置对象。应用程序的规模也会影响决策:在规模大的情况下，你可能更倾向于减少应用程序部署的数量，并依赖可变配置。

这个决定有文化上的影响，因为它框定了应用程序发布是如何被感知的——把它看作是没什么大不了的日常事件，或者看作是要尽可能避免的有点可怕的事情。在容器世界中，发布应该是一些微不足道的事件，只要需要，您就会乐于以最小的仪式来完成。测试和调整你的发行策略会给你带来很大的信心。

## 9.3 为 Deployments 配置滚动更新

Deployment 支持两种更新策略:RollingUpdate 是默认的，也是我们目前使用的一种，另一种是 Recreate。您知道 rollout 更新是如何工作的——通过缩小旧的ReplicaSet，同时扩大新的ReplicaSet，这提供了服务连续性和在较长时间内错开更新的能力。而 Recreate 策略则不提供这两种情况。它仍然使用 ReplicaSets 来实现更改，但是在扩展替换之前，它将先前的设置缩小到零。清单9.2显示了部署 spec 中的recreate策略。这只是一个设置，但它具有重大影响。

> 清单9.2 vweb-recreate-v2.yaml，一个使用重建更新策略的部署

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vweb
spec:
  replicas: 3
  strategy:                                # 更新策略.
    type: Recreate                         
  # selector & Pod spec follow
```

当你部署这个时，你会看到它只是一个普通的应用程序，有一个Deployment，一个ReplicaSet和一些Pods。如果查看Deployment的详细信息，您将看到它使用了 Recreate 更新策略，但这仅在更新 Deployment 时才会起作用。

<b>现在就试试</b> 部署清单9.2中的应用程序，并浏览对象。这就像一个正常的部署

```
# 删除 app:
kubectl delete deploy vweb

# 部署 Recreate 策略的应用:
kubectl apply -f vweb-strategies/vweb-recreate-v2.yaml

# 检查 ReplicaSets:
kubectl get rs -l app=vweb

# 测试 app:
curl $(cat url.txt)

# 查看 Deployment 详细:
kubectl describe deploy vweb
```

如图 9.9 所示，这是同一个旧web应用程序的新部署，使用容器镜像的版本2。有三个pod，它们都在运行，应用程序按预期运行——到目前为止还不错

![图9.9](.\images\Figure9.9.png)
<center>图 9.9 在你发布更新之前，重新创建更新策略不会影响行为</center>

不过，这种配置很危险，只有在应用程序的不同版本不能共存时才应该使用这种配置——就像数据库模式更新一样，需要确保只有一个版本的应用程序连接到数据库。即使在这种情况下，您也有更好的选择，但如果您的场景确实需要这种方法，那么您最好确保在发布之前测试所有更新。如果你部署了一个更新，而新Pods失败了，你会直到旧Pods全部被终止才知道，你的应用将完全不可用。

<b>现在就试试</b>版本3的web应用程序已经准备好部署。当应用程序离线时，你会看到它坏了，因为没有Pods在运行

```
# 部署 v3:
kubectl apply -f vweb-strategies/vweb-recreate-v3.yaml

# 检查状态，有更新的时间限制:
kubectl rollout status deploy/vweb --timeout=2s

# 检查 ReplicaSets:
kubectl get rs -l app=vweb

# 检查 Pods:
kubectl get pods -l app=vweb

# 测试 app–将会失败:
curl $(cat url.txt)
```

在这个练习中，你会看到Kubernetes很乐意让你的应用脱机，因为这是你所请求的。Recreate 策略使用更新后的Pod模板创建一个新的ReplicaSet，然后将先前的ReplicaSet缩小到0，并将新的ReplicaSet扩大到3。新的镜像被破坏了，所以新的Pods失败了，没有任何东西可以响应请求，如图9.10所示。

![图9.10](.\images\Figure9.10.png)
<center>图 9.10 如果新的Pod规格被打破，Recreate 策略会愉快地关闭你的应用 </center>

现在您已经看到了它，您可能应该尝试忘记 Recreate 策略。在某些情况下，它可能看起来很有吸引力，但当它出现时，您仍然应该考虑替代选项，即使这意味着重新查看您的体系结构。应用程序的大规模关闭将导致停机，而且停机时间可能比您计划的要长。

RollingUpdate 是默认的，因为它们可以防止停机，但即使这样，默认行为也是相当激进的。对于产品版本，您需要调优一些设置，以设置发布的速度以及如何监控它。作为rollout 更新规范的一部分，您可以使用以下两个值添加控制新ReplicaSet的扩展速度和旧ReplicaSet的缩小速度的选项:
- maxUnavailable 是用于缩小旧 ReplicaSet 的加速器。它定义了在更新期间，相对于所需的Pod计数，可以有多少Pod不可用。您可以把它看作是在旧的ReplicaSet中终止pod的批处理大小。在10个pod的部署中，将此设置为30%意味着将立即终止3个pod。
- maxSurge 是扩展新ReplicaSet的加速器。它定义了在期望的副本计数上可以存在多少额外的pod，就像在新的ReplicaSet中创建pod的批处理大小一样。在10个部署中，将此设置为40%将创建4个新pod。

很好，很简单，除了这两种设置都是在 rollout 时使用的，所以你有一个跷跷板的效果。新的ReplicaSet被放大，直到Pod计数是所需的副本计数加上maxSurge值，然后部署等待旧的Pod被删除。旧的ReplicaSet被缩小到所需的计数减去maxUnavailable计数，然后Deployment等待新Pods达到就绪状态。你不能把两个值都设为0，因为这意味着什么都不会改变。图9.11显示了如何组合这些设置，以便对新版本使用先创建再删除、先删除再创建或先删除再创建的方法。

![图9.11](.\images\Figure9.11.png)
<center>图 9.11 正在进行部署更新，使用不同的 rollout 选项</center>

如果您的集群中有空闲的计算能力，您可以调整这些设置以更快地 rollout。您还可以在您的比例设置上创建额外的pod，但如果您对新版本有问题，那么这样做的风险更大。缓慢的发布更保守:它使用更少的计算，给你更多的机会发现任何问题，但它在发布期间降低了应用程序的整体容量。让我们来看看这些是什么样子，首先通过保守的 rollout 来修复我们的坏应用程序。

<b>现在就试试</b>恢复到工作版本2的镜像，使用 maxSurge=1 和 maxUnavailable=0 在RollingUpdate策略

```
# 更新 Deployment 使用 rolling updates 以及 v2 image:
kubectl apply -f vweb-strategies/vweb-rollingUpdate-v2.yaml

# 检查 Pods:
kubectl get po -l app=vweb

# 检查 rollout status:
kubectl rollout status deploy/vweb

# 检查 ReplicaSets:
kubectl get rs -l app=vweb

# 测试 app:
curl $(cat url.txt)
```

在本练习中，新的部署规范将Pod镜像更改回版本2，并将更新策略更改为滚动更新。你可以在图9.12中看到，首先进行了策略更改，然后根据新策略进行Pod更新，一次创建一个新的Pod。

![图9.12](.\images\Figure9.12.png)
<center>图 9.12 如果Pod模板匹配新规范，部署更新将使用现有的ReplicaSet</center>

在前面的练习中，您需要快速工作以查看 rollout 的进度，因为这个简单的应用程序很快启动，只要一个新的Pod正在运行，就会继续使用另一个新Pod进行 rollout。在部署规范中，您可以通过以下两个字段来控制部署的速度:
- minReadySeconds 增加了一个延迟，即部署等待以确保新pod稳定。它指定了Pod在没有容器崩溃之前应该启动的秒数。默认值为0，这就是为什么在 rollout 过程中会快速创建新pod的原因。
- progressDeadlineSeconds 指定部署更新在被视为进展失败之前可以运行的时间。默认值是600秒，因此如果更新没有在10分钟内完成，它将被标记为未进行。

监视发布需要多长时间听起来很有用，但从Kubernetes 1.19开始，超过最后期限实际上并不会影响部署—它只是在部署上设置了一个标志。Kubernetes没有自动回滚功能，但是当该功能出现时，它将由这个标志触发。等待和检查Pod中失败的容器是一个相当生硬的工具，但这比根本不检查要好，您应该考虑在所有部署中指定minReadySeconds。

这些安全措施可以添加到你的部署中，但它们对我们的web应用没有真正的帮助，因为新的Pods总是失败。我们可以使此部署安全，并使用滚动更新保持应用程序在线。下一个版本3更新将maxUnavailable和maxSurge设置为1。这样做与默认值(每个25%)具有相同的效果，但在规范中使用确切的值更清楚，并且在小型部署中Pod计数比百分比更容易处理。

<b>现在就试试</b>再次部署版本3更新。它仍然会失败，但通过使用RollingUpdate策略，它不会让应用程序离线

```
# 更新失败的容器镜像:
kubectl apply -f vweb-strategies/vweb-rollingUpdate-v3.yaml

# 检查 Pods:
kubectl get po -l app=vweb

# 检查 rollout:
kubectl rollout status deploy/vweb

# 查看  ReplicaSets:
kubectl get rs -l app=vweb

# 测试 app:
curl $(cat url.txt)
```

当您运行这个练习时，您将看到更新永远不会完成，并且部署卡住了两个ReplicaSets，其Pod数为2，如图9.13所示。旧的ReplicaSet将不会进一步缩小，因为部署已经将maxUnavailable设置为1;它的规模已经缩小了1，不会有新的pod准备继续推出。新的ReplicaSet将不再扩展，因为maxSurge被设置为1，并且已经达到了部署的总Pod数。

![图9.13](.\images\Figure9.13.png)
<center>图 9.13 失败的更新不会自动回滚或暂停;他们只是一直在努力</center>

如果您在几分钟后检查新pod，您将看到它们处于CrashLoopBackoff状态。Kubernetes通过创建替换容器来重新启动失败的Pods，但它在每次重新启动之间添加了暂停，这样就不会阻塞节点上的CPU。这个暂停就是后退时间，它呈指数级增加——第一次重新启动时为10秒，然后是20秒，然后是40秒，最多5分钟。这些第三版pod将永远无法成功重启，但Kubernetes将继续尝试。

Deployments 是您使用最多的控制器，值得花时间研究更新策略和定时设置，以确保您理解.对你的应用的影响。DaemonSets和StatefulSets也有滚动更新功能，因为它们有不同的管理pod的方式，所以它们也有不同的 rollout 方法。

## 9.4 DaemonSets 和 StatefulSets 中的滚动更新

DaemonSets 和 StatefulSets 有两个可用的更新策略。默认的是RollingUpdate，我们将在本节中探讨它。另一种方法是OnDelete，它用于需要密切控制每个Pod何时更新的情况。你部署更新，控制器会监视pod，但它不会终止任何现有的pod。它会一直等待，直到它们被另一个进程删除，然后用新规范中的pod替换它们。

当您考虑这些控制器的用例时，这并不像听起来那么毫无意义。您可能有一个StatefulSet，其中每个Pod在删除之前都需要将数据刷新到磁盘，您可以有一个自动化的过程来完成这一点。您可能有一个DaemonSet，其中每个Pod都需要从硬件组件断开连接，以便下一个Pod可以使用它。这种情况很少见，但是OnDelete策略可以让你在删除Pods时拥有所有权，并且仍然可以让Kubernetes自动创建替换。

在本节中，我们将重点关注 rollingupdate，为此，我们将部署一个版本的待办事项列表应用程序，它在StatefulSet中运行数据库，在Deployment中运行web应用程序，在DaemonSet中运行web应用程序的反向代理。

<b>现在就试试</b> 待办事项应用程序可以在6个pod中运行，所以首先清理现有的应用程序以腾出空间。然后部署应用程序，并测试它是否正常工作

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