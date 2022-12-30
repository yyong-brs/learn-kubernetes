 # 第四章 通过 ConfigMaps 和 Secrets 配置应用程序

 在容器中运行应用程序的最大优势之一是消除了环境之间的差距。部署过程在所有测试环境直到生产环境中都使用相同的容器镜像，因此每个部署都使用与前一个环境完全相同的二进制文件集。您再也不会看到生产部署失败，因为服务器缺少某人手动安装在测试服务器上并且忘记记录的依赖项。当然，环境之间确实存在差异，您可以通过将配置设置注入到容器中来提供这种差异。

Kubernetes支持两种资源类型的配置注入:ConfigMap和Secrets。这两种类型都可以以任何合理的格式存储数据，并且这些数据独立于任何其他资源存在于集群中。可以通过访问ConfigMaps和Secrets中的数据来定义Pods，并为数据如何展现提供不同的选项。在本章中，您将学习在Kubernetes中管理配置的所有方法，这些方法足够灵活，可以满足任何应用程序的需求。

 ## 4.1 Kubernetes 如何为应用提供配置

 使用 kubectl 创建ConfigMap和Secret对象，就像在kubernete中创建其他资源一样，可以使用create命令，也可以应用YAML规范。不像其他资源，它们什么都不做;它们只是存储少量数据的存储单元。这些存储单元可以加载到Pod中，成为容器环境的一部分，因此容器中的应用程序可以读取数据。在讨论这些对象之前，我们先来看看提供配置设置的最简单方法:使用环境变量。

<b>现在就试试</b> 环境变量是Linux和Windows中的核心操作系统特性，它们可以在机器级别设置，以便任何应用程序都可以读取它们。环境变量是常用的，所有容器都有一些环境变量，由容器内的操作系统和Kubernetes设置。确保您的Kubernetes实验室已经启动并运行。

```
# 切换到本章练习目录:
cd ch04
# 使用 sleep image 部署一个没有额外配置的 Pod:
kubectl apply -f sleep/sleep.yaml
# 等待 pod ready:
kubectl wait --for=condition=Ready pod -l app=sleep
# 检查容器中的一些环境变量:
kubectl exec deploy/sleep -- printenv HOSTNAME KIAMOL_CHAPTER
```

从图4.1所示的输出中可以看到，容器中存在hostname变量，并由Kubernetes填充，但自定义Kiamol变量不存在。

![图4.1 所有Pod容器都有Kubernetes和容器操作系统设置的一些环境变量.](./images/Figure4.1.png)

在本练习中，应用程序只是Linux printenv工具，但原理对任何应用程序都是相同的。许多技术栈使用环境变量作为基本配置系统。在Kubernetes中提供这些设置的最简单方法是在Pod Spec 中添加环境变量。清单4.1显示了 sleep Deployment 的更新后的 Pod Spec，其中添加了Kiamol环境变量。

> Listing 4.1 sleep-with-env.yaml, 一个 Pod Spec 带了环境变量

```
spec:
  containers:
    - name: sleep
      image: kiamol/ch03-sleep
      env:                    # 设置环境变量
      - name: KIAMOL_CHAPTER  # 定义环境变量名称
        value: "04"           # 定义环境变量值
```

环境变量在Pod的生命周期中是静态的;在Pod运行时，你不能更新任何值。如果需要更改配置，则需要使用替换Pod执行更新。你应该习惯这样的想法:部署不仅仅是为了新特性的发布;你也会使用它们来进行配置更改和软件补丁，你必须设计你的应用程序来处理频繁的Pod更换。

<b>现在就试试</b> 使用清单4.1中的新Pod 配置更新 sleep Deployment，添加一个Pod容器内可见的环境变量。

```
# 更新 Deployment:
kubectl apply -f sleep/sleep-with-env.yaml
# 在新的 Pod 中检查同样的环境变量:
kubectl exec deploy/sleep -- printenv HOSTNAME KIAMOL_CHAPTER
```

我的输出(如图4.2所示)显示了结果——一个设置了Kiamol环境变量的新容器，在一个新的Pod中运行。

![图4.2 将环境变量添加到Pod Spec 中可以在Pod容器中使用这些值.](./images/Figure4.2.png)

关于前面的练习，重要的是新应用程序使用相同的Docker 镜像;这是一个具有相同二进制文件的相同应用程序——只是配置设置在部署之间发生了更改。在Pod Spec 中内联设置环境值对于简单设置很好，但实际应用程序通常有更复杂的配置需求，这就是使用ConfigMaps时的情况。

ConfigMap只是一个资源，它存储了一些可以加载到Pod中的数据。数据可以是一组键-值对、文本简介，甚至是二进制文件。您可以使用键-值对加载带有环境变量的Pods，使用文本加载任何类型的配置文件—json、XML、YAML、TOML、ini—以及二进制文件加载许可密钥。一个Pod可以使用多个ConfigMap，每个ConfigMap可以被多个Pod使用。图4.3显示了其中的一些选项。

![图4.3 configmap是单独的资源，可以附加到0个或多个pod.](./images/Figure4.3.png)

我们将继续使用简单的 sleep Deployment，以展示创建和使用configmap的基础知识。清单4.2显示了更新后Pod Spec的环境部分，其中使用了一个在YAML中定义的环境变量和一个从ConfigMap中加载的环境变量。

> 清单 4.2 sleep-with-configMap-env.yaml, 加载 ConfigMap 到 Pod 中

```
env:                             # 容器 Spec 环境变量配置部分
- name: KIAMOL_CHAPTER
  value: "04"                    # 变量值.
- name: KIAMOL_SECTION
  valueFrom:
    configMapKeyRef:             # 值来自于 ConfigMap.
      name: sleep-config-literal # ConfigMap 名称
      key: kiamol.section        # 加载的数据项名称
```
如果在 Pod Spec 中引用了ConfigMap，那么在部署Pod之前，ConfigMap必须已经存在。该配置期望在数据中找到一个名为sleep-config-literal的具有键值对的ConfigMap，最简单的创建方法是将键和值传递给kubectl命令。

<b>现在就试试</b> 通过指定命令中的数据创建ConfigMap，然后检查数据并部署更新后的sleep 应用程序来使用ConfigMap。

```
# 基于命令行创建 ConfigMap:
kubectl create configmap sleep-config-literal --from-literal=kiamol.section='4.1'
# 检查 ConfigMap 详情:
kubectl get cm sleep-config-literal
# 显示 ConfigMap 友好的描述信息:
kubectl describe cm sleep-config-literal
# 基于 清单 4.2 部署更新后的 Pod 配置:
kubectl apply -f sleep/sleep-with-configMap-env.yaml
# 检查 Kiamol 环境变量:
kubectl exec deploy/sleep -- sh -c 'printenv | grep "^KIAMOL"'
```

在本书中，我们不会经常使用kubectl describe 命令，因为输出通常很冗长，会占用大部分屏幕，但它绝对是值得尝试的东西。描述Services和Pods以可读的格式为您提供了许多有用的信息。您可以在图4.4中看到我的输出，其中包括描述ConfigMap时显示的键值数据。

![图4.4 Pods可以从ConfigMaps加载单独的数据项并重命名键.](./images/Figure4.4.png)

从文字值创建 ConfigMap 对于单独的设置来说很好，但是如果您有很多配置数据，它会变得非常麻烦。除了在命令行上指定文本值外，Kubernetes还允许你从文件中加载configmap。

 ## 4.2 在 ConfigMaps 中存储和使用配置文件

在许多 Kubernetes 版本中，创建和使用configmap的选项已经得到了发展，所以它们现在几乎支持您能想到的所有配置变体。这些 sleep Pod 练习是展示变化的好方法，但它们有点无聊，所以在我们进入更有趣的内容之前，我们再做一个。清单4.3显示了一个环境文件——一个具有键-值对的文本文件，可以加载它来创建一个带有多个数据项的ConfigMap。

> 清单 4.3 ch04.env, 一个包含环境变量的文件

```
# 环境变量文件，使用每个新行定义每个变量
KIAMOL_CHAPTER=ch04
KIAMOL_SECTION=ch04-4.1
KIAMOL_EXERCISE=try it now
```

环境文件是将多个设置分组的有效方法，Kubernetes明确支持将它们加载到ConfigMaps中，并将所有设置作为Pod容器中的环境变量呈现出来。

<b>现在就试试</b> 创建一个从清单4.3中的环境文件填充的新的ConfigMap，然后将更新部署到 sleep 应用程序以使用新的设置。

```
# 加载环境变量到新的 ConfigMap:
kubectl create configmap sleep-config-env-file --from-env-file=sleep/ch04.env
# 检查 ConfigMap 明细信息:
kubectl get cm sleep-config-env-file
# 更新 Pod 使用新的 ConfigMap:
kubectl apply -f sleep/sleep-with-configMap-env-file.yaml
# 在容器中检查值:
kubectl exec deploy/sleep -- sh -c 'printenv | grep "^KIAMOL"'
```

在图4.5中，我的输出显示 printenv 命令读取所有环境变量并显示具有Kiamol名称的变量，但这可能不是您所期望的结果。

![图4.5 一个ConfigMap可以有多个数据项，Pod可以全部加载它们.](./images/Figure4.5.png)

这个练习向您展示了如何从文件创建ConfigMap。它还向您展示了Kubernetes在应用环境变量时具有优先级规则。您刚刚部署的Pod规范(如清单4.4所示)从ConfigMap加载所有环境变量，但它也使用一些相同的键显式指定环境值。

> 清单 4.4 sleep-with-configMap-env-file.yaml, 一个 Pod 关联多个 ConfigMaps

```
env:                              # 已经存在的环境配置部分
- name: KIAMOL_CHAPTER
  value: "04"
- name: KIAMOL_SECTION
  valueFrom:
    configMapKeyRef:
      name: sleep-config-literal
      key: kiamol.section
envFrom:                          # envFrom 加载多个变量
- configMapRef:                   # 基于 ConfigMap
    name: sleep-config-env-file
```

因此，如果存在重复的键，Pod规范中用env定义的环境变量将覆盖用envFrom定义的值。记住一点很有用，您可以通过显式地在Pod规范中设置容器镜像或ConfigMaps中设置的任何环境变量来覆盖它们——这是在跟踪问题时更改配置设置的快速方法。

环境变量得到了很好的支持，但它们只能提供到此为止，大多数应用程序平台更喜欢一种更结构化的方法。在本章的其余练习中，我们将使用一个支持配置源层次结构的web应用程序。默认设置被打包在Docker 镜像中的JSON文件中，应用程序在运行时在其他位置寻找具有覆盖默认设置的JSON文件——所有JSON设置都可以用环境变量覆盖。清单4.5显示了我们将使用的第一个部署的Pod规范。

> 清单 4.5 todo-web.yaml, 带有配置设置的web应用程序

```
spec:
  containers:
  - name: web
    image: kiamol/ch04-todo-list
    env:
    - name: Logging__LogLevel__Default
      value: Warning
```

这个应用程序的运行将使用镜像中JSON配置文件中的所有默认设置，除了默认日志级别，它被设置为一个环境变量。

<b>现在就试试</b> 在没有任何额外配置的情况下运行应用程序，并检查其行为。

```
# 部署应用程序和 Service 来访问它:
kubectl apply -f todo-list/todo-web.yaml
# 等待 Pod ready:
kubectl wait --for=condition=Ready pod -l app=todo-web
# 获取应用 地址:
kubectl get svc todo-web -o jsonpath='http://{.status.loadBalancer.ingress[0].*}:8080'

# 浏览应用程序，玩一玩，然后尝试浏览/config 检查应用程序日志
kubectl logs -l app=todo-web
```

演示应用程序是一个简单的待办事项列表(对于《在一个月的午餐中学习Docker》的读者来说，这将是令人痛苦的熟悉)。在当前的设置中，它允许您添加和查看项，但是还应该有一个/config页面，我们可以在非生产环境中使用它来查看所有的配置设置。如图4.6所示，该页面为空，应用程序记录了有人试图访问它的警告。

![图4.6 应用程序大部分工作，但我们需要设置额外的配置值.](./images/Figure4.6.png)

这里使用的配置层次结构是一种非常常见的方法。如果你对它不熟悉，电子书的附录C是《Learn Docker in a Month of lunch》中的“容器中的应用程序配置管理”章节，其中详细解释了它。这个例子是一个使用JSON的. net Core应用程序，但是你可以在Java Spring应用程序、Node.js、Go、Python等中看到使用各种文件格式的类似配置系统。在Kubernetes中，你使用相同的应用配置方法。

- 这可能只是适用于每个环境的设置，也可能是一组完整的配置选项，所以不需要任何额外的设置，应用程序就可以在开发模式下运行(这对那些可以用简单的Docker run命令快速启动应用程序的开发人员很有帮助)。
- 每个环境的实际设置存储在ConfigMap中，并呈现到容器文件系统中。Kubernetes将配置数据显示为一个位于已知位置的文件，应用程序检查并将其与默认文件中的内容合并。
- 任何需要调整的设置都可以作为部署Pod规范中的环境变量应用。

清单4.6显示了todo应用程序开发配置的YAML规范。它包含一个JSON文件的内容，应用程序将该JSON文件与容器镜像中的默认JSON配置文件合并，并设置使配置页面可见。

> 清单 4.6 todo-web-config-dev.yaml, 一个 ConfigMap的配置信息

```
apiVersion: v1
kind: ConfigMap                # ConfigMap 是一种资源类型
metadata:
  name: todo-web-config-dev    # ConfigMap 名字
data:
  config.json: |-              # 数据 key 是文件名
    {                          # 文件内容可以是任何格式
      "ConfigController": {
        "Enabled" : true
      }
    }
```

您可以将任何类型的文本配置文件嵌入到YAML规范中，只要小心处理空白即可。我更喜欢直接从配置文件加载ConfigMaps，因为这意味着你可以始终使用kubectl apply命令来部署应用程序的每个部分。如果我想直接加载JSON文件，我需要使用kubectl create命令来配置资源，并应用其他所有内容。

清单4.6中的ConfigMap定义只包含一个设置，但它以应用程序的本机配置格式存储。当我们部署更新的Pod规范时，该设置将被应用，配置页面将可见。


<b>现在就试试</b> 新的Pod规范引用了ConfigMap，因此需要首先通过应用YAML来创建ConfigMap，然后我们更新待办事项应用部署。

```
# 创建 JSON ConfigMap:
kubectl apply -f todo-list/configMaps/todo-web-config-dev.yaml
# 更新应用使用ConfigMap:
kubectl apply -f todo-list/todo-web-dev.yaml
# 刷新网页指向 /config 路由页面
```

可以在图4.7中看到我的输出。配置页面现在可以正确加载，因此新的部署配置正在合并ConfigMap中的设置，以覆盖镜像中的默认设置，该设置阻止了对该页的访问。

![图4.7 将ConfigMap数据加载到容器文件系统中，应用程序将在其中加载配置文件.](./images/Figure4.7.png)

这种方法需要两个前提:应用程序需要能够合并ConfigMap数据，Pod规范需要将ConfigMap中的数据加载到容器文件系统中的预期文件路径中。我们将在下一节中看到它是如何工作的。

 ## 4.3 从 ConfigMaps 中查找配置数据

 将配置项加载到环境变量中的替代方法是将它们作为容器目录中的文件表示。容器文件系统是一个虚拟构造，由容器镜像和其他源构建。Kubernetes可以使用ConfigMaps作为文件系统源——它们作为一个目录挂载，每个数据项都有一个文件。图4.8显示了您刚刚部署的设置，其中ConfigMap中的数据项显示为一个文件。

![图4.8 configmap可以作为容器文件系统中的目录加载.](./images/Figure4.8.png)

Kubernetes通过Pod 配置 Spec 的两个特性来管理这个奇怪的魔法:卷，它使ConfigMap的内容对Pod可用，卷挂载，它将ConfigMap卷的内容加载到Pod容器的指定路径中。清单4.7显示了在前面的练习中部署的卷和挂载。

> 清单 4.7 todo-web-dev.yaml, 将 ConfigMap 作为卷挂载

```
spec:
  containers:
    - name: web
      image: kiamol/ch04-todo-list
      volumeMounts:                  # 挂载卷到容器中
        - name: config               # 卷名称
          mountPath: "/app/config"   # 卷挂载的目录
          readOnly: true             # 卷 read-only 配置
  volumes:                           # Pod 层面定义卷
    - name: config                   # 匹配 volumeMount 部分的名称.
      configMap:                     # 卷来源是一个 ConfigMap.
        name: todo-web-config-dev    # ConfigMap 名称
```

这里需要意识到的重要一点是，ConfigMap被视为一个目录，具有多个数据项，每个数据项都成为容器文件系统中的文件。在这个例子中，应用程序从/app/appsettings.json文件中加载它的默认设置，然后它在/app/config 目录下寻找一个文件config.Json，它可以包含覆盖默认值的设置。容器镜像中不存在/app/config目录;它是由Kubernetes创建和填充的。

<b>现在就试试</b> 容器文件系统对于应用程序来说是一个单独的存储单元，但是它是由镜像和ConfigMap构建的。这些源有不同的行为。

```
# 显示默认的 config 文件:
kubectl exec deploy/todo-web -- sh -c 'ls -l /app/app*.json'
# 显示卷挂载进来的 config 文件:
kubectl exec deploy/todo-web -- sh -c 'ls -l /app/config/*.json'
# 检查它是否真的只读:
kubectl exec deploy/todo-web -- sh -c 'echo ch04 >> /app/config/config.json'
```

如图4.9所示，我的输出显示JSON配置文件存在于应用程序的预期位置，但ConfigMap文件由Kubernetes管理，并作为只读文件交付。

![图4.9 容器文件系统由Kubernetes从镜像和ConfigMap构建。.](./images/Figure4.9.png)

将ConfigMaps加载为目录是很灵活的，你可以使用它来支持不同的应用配置方法。如果您的配置被拆分到多个文件中，您可以将其全部存储在一个ConfigMap中，并将其全部加载到容器中。清单4.8显示了使用两个JSON文件更新to-do ConfigMap的数据项，这些JSON文件分离了应用程序行为和日志记录的设置。

> 清单 4.8 todo-web-config-dev-with-logging.yaml, 包含两个文件的 ConfigMap

```
data:
  config.json: |-                 # 原始的 app 配置文件
    {
      "ConfigController": {
        "Enabled" : true
      }
    }
  logging.json: |-                # 第二个 JSON file, 将会出现在卷挂载中
    {                             
      "Logging": {
        "LogLevel": {
          "ToDoList.Pages" : "Debug"
        }
      }
    }
```

当您将一个更新部署到一个live Pod正在使用的ConfigMap时会发生什么?Kubernetes将更新后的文件交付到容器，但接下来会发生什么取决于应用程序。一些应用程序在启动时将配置文件加载到内存中，然后忽略配置目录中的任何更改，因此更改ConfigMap实际上不会改变应用程序的配置，直到替换Pods。这个应用程序更加周到——它监视配置目录并重新加载任何文件更改，因此将更新部署到ConfigMap将更新应用程序配置。

<b>现在就试试</b> 使用清单4.9中的ConfigMap更新应用程序配置。这增加了日志记录级别，所以同一个Pod现在将开始写入更多的日志条目。

```
# 检查当前应用日志:
kubectl logs -l app=todo-web
# 部署更新后的 ConfigMap:
kubectl apply -f todo-list/configMaps/todo-web-config-dev-with-logging.yaml
# 等待配置变更生效到 Pod :
sleep 120
# 检查新的设置:
kubectl exec deploy/todo-web -- sh -c 'ls -l /app/config/*.json'

# 从您的 Service IP地址的站点加载几个页面，再次检查日志：
kubectl logs -l app=todo-web
```

可以在图4.10中看到我的输出。sleep 是为了让Kubernetes API有时间将新的配置文件在Pod 重新加载;几分钟后，新配置被加载，应用程序在增强的日志记录下运行。

![图4.10 ConfigMap数据是缓存的，所以更新需要几分钟才能到达Pods.](./images/Figure4.10.png)

卷是加载配置文件的一个强大的选项，尤其是像这样的应用程序，它会对变化做出反应，并实时更新设置。在不重启应用程序的情况下提高日志级别对跟踪问题有很大帮助。但是，您需要小心您的配置，因为卷挂载不一定以您期望的方式工作。如果容器镜像中已经存在卷的挂载路径，那么ConfigMap目录将覆盖它，替换所有内容，这可能导致应用程序以令人兴奋的方式失败。清单4.9显示了一个示例。

> 清单 4.9 todo-web-dev-broken.yaml, 一个 Pod配置了错误的挂载

```
spec:
  containers:
    - name: web
      image: kiamol/ch04-todo-list
      volumeMounts:
        - name: config                # 挂载 ConfigMap 卷
          mountPath: "/app"           # 覆盖目录
```

这是一个坏的Pod spec 配置，其中ConfigMap被加载到/app目录而不是/app/config目录。作者可能是想合并目录，将JSON配置文件添加到现有的应用目录中。相反，它将清除应用程序二进制文件。

<b>现在就试试</b> 清单4.9中的Pod 配置删除了所有应用程序二进制文件，因此替换的Pod将无法启动。看看接下来会发生什么。

```
# 部署错误配置的 Pod:
kubectl apply -f todo-list/todo-web-dev-broken.yaml
# 访问应用程序，看看它的样子
# 检查应用日志:
kubectl logs -l app=todo-web
# 检查 pod 状态:
kubectl get pods -l app=todo-web
```

这里的结果很有趣:部署破坏了应用程序，但应用程序继续工作。这是Kubernetes在保护你。应用更改会创建一个新的Pod，该Pod中的容器立即退出并报错，因为它试图加载的二进制文件不再存在于app目录中。Kubernetes重新启动容器几次，给它一个机会，但它一直失败。经过三次尝试后，Kubernetes开始休息，如图4.11所示。

![图4.11 如果更新的deployment 失败，则不会替换原来的Pod](./images/Figure4.11.png)

现在我们有了两个Pod，但Kubernetes不会删除旧的Pod，直到替换的Pod成功运行，在这种情况下它永远不会删除，因为我们破坏了容器设置。旧的Pod没有被移除，仍然愉快地服务于请求;新的Pod处于失败状态，但Kubernetes周期性地重新启动容器，希望它可能已经自我修复。这是一种需要注意的情况:apply命令似乎可以工作，应用程序继续工作，但它没有使用您所应用的清单。

现在我们将修复这个问题，并展示在容器文件系统中显示ConfigMaps的最后一个选项。您可以有选择地将数据项加载到目标目录中，而不是将每个数据项作为其自己的文件加载。清单4.10显示了更新后的Pod规范。挂载路径已经固定，但是卷被设置为只交付一个项。

> 清单 4.10 todo-web-dev-no-logging.yaml, 挂载 ConfigMap 单项

```
spec:
  containers:
    - name: web
      image: kiamol/ch04-todo-list
      volumeMounts:
        - name: config               # 挂载 ConfigMap 卷到正常的目录
          mountPath: "/app/config"   
          readOnly: true
volumes:
  - name: config
    configMap:
      name: todo-web-config-dev      # 加载 ConfigMap 卷
      items:                         # 指定数据项进行加载
      - key: config.json             # 加载 config.json 项
        path: config.json            # 作为 文件 config.json
```

该规范使用相同的ConfigMap，因此它只是对部署的更新。这将是一个级联更新:它将创建一个新的Pod，它将正确启动，并且然后Kubernetes会移除之前的两个pod。

<b>现在就试试</b>  部署清单4.10中的 spec，该 spec 推出更新后的卷挂载以修复应用程序，但也忽略了ConfigMap中的日志JSON文件。

```
# 应用变更:
kubectl apply -f todo-list/todo-web-dev-no-logging.yaml
# 列出配置文件夹的内容:
kubectl exec deploy/todo-web -- sh -c 'ls /app/config'
# 现在浏览应用的几个页面，
# 检查日志:
kubectl logs -l app=todo-web
# 检查 Pods:
kubectl get pods -l app=todo-web
```

图4.12显示了我的输出。应用程序又开始工作了，但它只看到一个配置文件，所以没有应用增强的日志记录设置。

![图4.12 卷可以将ConfigMap中选定的项显示到挂载目录中.](./images/Figure4.12.png)

configmap支持广泛的配置系统。在环境变量和卷挂载之间，你应该能够在ConfigMaps中存储应用程序设置，并根据应用程序的需要应用它们。配置规范和应用规范之间的分离还支持不同的发布工作流，允许不同的团队拥有流程的不同部分。但是，有一件事不应该使用ConfigMaps，那就是任何敏感数据——它们实际上是文本文件的包装器，没有额外的安全语义。对于需要保护的配置数据，Kubernetes提供了Secrets。

 ## 4.4 使用 Secrets 配置敏感数据
 Secrets are a separate type of resource, but they have a similar API to ConfigMaps. You work with them in the same way, but because they’re meant to store sensitive information, Kubernetes manages them differently. The main differences are all around minimizing exposure. Secrets are sent only to nodes that need to use them and are stored in memory rather than on disk; Kubernetes also supports encryption both in transit and at rest for Secrets.

Secrets are not encrypted 100% of the time, though. Anyone who has access to Secret objects in your cluster can read the unencrypted values. There is an obfuscation layer: Kubernetes can read and write Secret data with Base64 encoding, which isn’t really a security feature but does prevent accidental exposure of secrets to someone looking over your shoulder.

TRY IT NOW
You can create Secrets from a literal value, passing the key and data into the kubectl command. The retrieved data is Base64 encoded.
```
# FOR WINDOWS USERS—this script adds a Base64 command to your session:
. .\base64.ps1
# now create a secret from a plain text literal:
kubectl create secret generic sleep-secret-literal --from-
literal=secret=shh...
# show the friendly details of the Secret:
kubectl describe secret sleep-secret-literal
# retrieve the encoded Secret value:
kubectl get secret sleep-secret-literal -o jsonpath='{.data.secret}'
# and decode the data:
kubectl get secret sleep-secret-literal -o jsonpath='{.data.secret}' |
  base64 -d
```

You can see from the output in figure 4.13 that Kubernetes treats Secrets differently from ConfigMaps. The data values aren’t shown in the kubectl describe command,only the names of the item keys, and when you do fetch the data, it’s shown encoded, so you need to pipe it into a decoder to read it.

![图4.13 Secrets have a similar API to ConfigMaps, but Kubernetes tries to avoid accidental exposure.](./images/Figure4.13.png)

That precaution doesn’t apply when Secrets are surfaced inside Pod containers. The container environment sees the original plain text data. Listing 4.11 shows a return to the sleep app, configured to load the new Secret as an environment variable.

> Listing 4.11 sleep-with-secret.yaml, a Pod spec loading a Secret

```
spec:
  containers:
    - name: sleep
      image: kiamol/ch03-sleep
      env:                                   # Environment variables
      - name: KIAMOL_SECRET                  # Variable name in the container
        valueFrom:                           # loaded from an external source
          secretKeyRef:                      # which is a Secret
            name: sleep-secret-literal       # Names the Secret
            key: secret                      # Key of the Secret data item
```

The specification to consume Secrets is almost the same as for ConfigMaps—a named environment variable can be loaded from a named item in a Secret. This Pod spec delivers the Secret item in its original form to the container.


TRY IT NOW
Run a simple sleep Pod that uses the Secret as an environment variable.

```
# update the sleep Deployment:
kubectl apply -f sleep/sleep-with-secret.yaml
# check the environment variable in the Pod:
kubectl exec deploy/sleep -- printenv KIAMOL_SECRET
```

Figure 4.14 shows the output. In this case the Pod is using only a Secret, but Secrets and ConfigMaps can be mixed in the same Pod spec, populating environment variables or files or both.

![图4.14 Secrets loaded into Pods are not Base64 encoded.](./images/Figure4.14.png)


You should be wary of loading Secrets into environment variables. Securing sensitive data is all about minimizing its exposure. Environment variables can be read from any process
in the Pod container, and some application platforms log all environment variable values if they hit a critical error. The alternative is to surface Secrets as files, if the application supports it, which gives you the option of securing access with file permissions.

To round off this chapter, we’ll run the to-do app in a different configuration where it uses a separate database to store items, running in its own Pod. The database
server is Postgres using the official image on Docker Hub, which reads logon credentials from configuration values in the environment. Listing 4.12 shows a YAML spec
for creating the database password as a Secret.

> Listing 4.12 -todo-db-secret-test.yaml, a Secret for the database user

```
apiVersion: v1
kind: Secret                         # Secret is the resource type.
metadata:
  name: todo-db-secret-test          # Names the Secret
type: Opaque                         # Opaque secrets are for text data.
stringData:                          # stringData is for plain text.
  POSTGRES_PASSWORD: "kiamol-2*2*"   # The secret key and value.
```

This approach states the password in plain text in the stringData field, which gets encoded to Base64 when you create the Secret. Using YAML files for Secrets poses a
tricky problem: it gives you a nice consistent deployment approach, at the cost of having all your sensitive data visible in source control.

In a production scenario, you would keep the real data out of the YAML file, using placeholders instead, and do some additional processing as part of your deployment—something like injecting the data into the placeholder from a GitHub Secret. Whichever approach you take, remember that once the Secret exists in Kubernetes, it’s easy for anyone who has access to read the value.

TRY IT NOW
Create a Secret from the manifest in listing 4.12, and check the data.

```
# deploy the Secret:
kubectl apply -f todo-list/secrets/todo-db-secret-test.yaml
# check the data is encoded:
kubectl get secret todo-db-secret-test -o
  jsonpath='{.data.POSTGRES_PASSWORD}'
# see what annotations are stored:
kubectl get secret todo-db-secret-test -o
  jsonpath='{.metadata.annotations}'
```

You can see in figure 4.15 that the string is encoded to Base64. The outcome is the same as it would be if the specification used the normal data field and set the password value in Base64 directly in the YAML.

![图4.15 Secrets created from string data are encoded, but the original data is also stored in the object.](./images/Figure4.15.png)

To use that Secret as the Postgres password, the image gives us a couple of options.We can load the value into an environment variable called POSTGRES_PASSWORD—not ideal—or we can load it into a file and tell Postgres where to load the file, by setting the POSTGRES_PASSWORD_FILE environment variable. Using a file means we can control access permissions at the volume level, which is how the database is configured in code listing 4.13.

> Listing 4.13 todo-db-test.yaml, a Pod spec mounting a volume from a secret
```
spec:
  containers:
    - name: db
    image: postgres:11.6-alpine
    env:
    - name: POSTGRES_PASSWORD_FILE            # Sets the path to the file
      value: /secrets/postgres_password
    volumeMounts:                             # Mounts a Secret volume
      - name: secret                          # Names the volume
        mountPath: "/secrets"
  volumes:
    - name: secret
    secret:                                   # Volume loaded from a Secret
    secretName: todo-db-secret-test           # Secret name
    defaultMode: 0400                         # Permissions to set for files
    items:                                    # Optionally names the data items
    - key: POSTGRES_PASSWORD
    path: postgres_password
```

When this Pod is deployed, Kubernetes loads the value of the Secret item into a file at the path /secrets/postgres_password. That file will be set with 0400 permissions, which means it can be read by the container user but not by any other users. The environment variable is set for Postgres to load the password from that file, which the Postgres user has access to, so the database will start with credentials set from the Secret.

TRY IT NOW
Deploy the database Pod, and verify the database starts correctly.

```
# deploy the YAML from listing 4.13
kubectl apply -f todo-list/todo-db-test.yaml
# check the database logs:
kubectl logs -l app=todo-db --tail 1
# verify the password file permissions:
kubectl exec deploy/todo-db -- sh -c 'ls -l $(readlink -f
  /secrets/postgres_password)'
```

Figure 4.16 shows the database starting up and waiting for connections—indicating it has been configured correctly—and the final output verifies that the file permissions are set as expected.

![图4.16 If the app supports it, configuration settings can be read by files populated from Secrets.](./images/Figure4.16.png)

All that’s left is to run the app itself in the test configuration, so it connects to the Postgres database rather than using a local database file for storage. There’s lots more
YAML for that, to create a ConfigMap, Secret, Deployment, and Service, but it’s all using features we’ve covered already, so we’ll just go ahead and deploy.

TRY IT NOW
Run the to-do app so it uses the Postgres database for storage.

```
# the ConfigMap configures the app to use Postgres:
kubectl apply -f todo-list/configMaps/todo-web-config-test.yaml

# the Secret contains the credentials to connect to Postgres:
kubectl apply -f todo-list/secrets/todo-web-secret-test.yaml
# the Deployment Pod spec uses the ConfigMap and Secret:
kubectl apply -f todo-list/todo-web-test.yaml
# check the database credentials are set in the app:
kubectl exec deploy/todo-web-test -- cat /app/secrets/secrets.json
# browse to the app and add some items
```
My output is shown in figure 4.17, where the plain text contents of the Secret JSON
file are shown inside the web Pod container.

![图4.17 Loading app configuration into Pods and surfacing ConfigMaps and Secrets as JSON files](./images/Figure4.17.png)

Now when you add to-do items in the app, they are stored in the Postgres database, so storage is separated from the application runtime. You can delete the web Pod; its controller will start a replacement with the same configuration, which connects to the same database Pod, so all the data from the original web Pod is still available.

This has been a pretty exhaustive look at configuration options in Kubernetes. The principles are quite simple—loading ConfigMaps or Secrets into environment variables or files—but there are a lot of variations. You need a good understanding of the nuances so you can manage app configuration in a consistent way, even if your apps all have different configuration models.

 ## 4.5 管理 Kubernetes 中的应用程序配置

 Kubernetes gives you the tools to manage app configuration using whatever workflow fits for your organization. The core requirement is for your applications to read configuration settings from the environment, ideally with a hierarchy of files and environment variables. Then you have the flexibility to use ConfigMaps and Secrets to support your deployment process. You have two factors to consider in your design: do you need your apps to respond to live configuration updates, and how will you manage Secrets?

If live updates without a Pod replacement are important to you, then your options are limited. You can’t use environment variables for settings, because any changes to those result in a Pod replacement. You can use a volume mount and load configuration changes from files, but you need to deploy changes by updating the existing ConfigMap or Secret objects. You can’t change the volume to point to a new config object, because that’s a Pod replacement too.

The alternative to updating the same config object is to deploy a new object every time with some versioning scheme in the object name and updating the app Deployment to reference the new object. You lose live updates but gain an audit trail of configuration changes and have an easy option to revert back to previous settings. Figure 4.18 shows those options.

![图4.18 You can choose your own approach to configuration management,supported by Kubernetes](./images/Figure4.18.png)

The other question is how you manage sensitive data. Large organizations might have dedicated configuration management teams who own the process of deploying configuration files. That fits nicely with a versioned approach to ConfigMaps and Secrets,where the configuration management team deploys new objects from literals or controlled files in advance of the deployment.

An alternative is a fully automated deployment, where ConfigMaps and Secrets are created from YAML templates in source control. The YAML files contain placeholders instead of sensitive data, and the deployment process replaces them with real values from a secure store, like Azure KeyVault, before applying them. Figure 4.19 compares those options.

![图4.19 Secret management can be automated in deployment or strictly controlled by a separate team.](./images/Figure4.19.png)

You can use any approach that works for your teams and your application stacks,remembering that the goal is for all configuration settings to be loaded from the platform, so the same container image is deployed in every environment.It’s time to clean up your cluster. If you’ve followed along with all the exercises (and of course you have!), you’ll have a couple of dozen resources to remove. I’ll introduce some useful features of kubectl to help clear everything out.

TRY IT NOW
The kubectl delete command can read a YAML file and delete the resources defined in the file. And if you have multiple YAML files in a directory, you can use the directory name as the argument to delete (or apply),and it will run over all the files.

```
# delete all the resources in all the files in all the directories:
kubectl delete -f sleep/
kubectl delete -f todo-list/
kubectl delete -f todo-list/configMaps/
kubectl delete -f todo-list/secrets/
```

 ## 4.6 实验室

 If you’re reeling from all the options Kubernetes gives you to configure apps, this lab is going to help. In practice, your apps will have their own ideas about configuration
management, and you’ll need to model your Kubernetes Deployments to suit the way your apps expect to be configured. That’s what you need to do in this lab with a simple app called Adminer. Here we go:
- Adminer, a web UI for administering SQL databases, can be a handy tool to run in Kubernetes when you’re troubleshooting database issues.
- Start by deploying the YAML files in the ch04/lab/postgres folder, then deploy the ch04/lab/adminer.yaml file to run Adminer in its basic state.
- Find the external IP for your Adminer Service, and browse to port 8082. Note that you need to specify a database server and that the UI design is stuck in the 1990s. You can confirm the connection to Postgres by using postgres as the database name, username, and password.
- Your job is to create and use some config objects in the Adminer Deployment so that the database server name defaults to the lab’s Postgres Service, and the UI uses the much nicer design called price.
- You can set the default database server in an environment variable called ADMINER_DEFAULT_SERVER. Let’s call this sensitive data, so it should use a Secret.
- The UI design is set in the environment variable ADMINER_DESIGN; that’s not sensitive, so a ConfigMap will do nicely.

This will take a little bit of investigation and some thought on how to surface the configuration settings, so it’s good practice for real application configuration. My solution
is posted on GitHub for you to check your approach: https://github.com/sixeyed/kiamol/blob/master/ch04/lab/README.md.