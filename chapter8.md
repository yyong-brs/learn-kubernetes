# 第八章 使用 StatfulSets 和 Jobs 运行数据量大的应用

“数据量大” 不是一个很科学的术语，但本章是关于运行一个应用程序类，它不仅是有状态的，而且还要求它如何使用状态。数据库就是此类的一个例子。它们需要跨多个实例运行以实现高可用性，每个实例都需要一个本地数据存储以实现快速访问，而且这些独立的数据存储需要保持同步。数据有其自身的可用性要求，您需要定期运行备份以防止终端故障或损坏。其他数据密集型应用程序，如消息队列和分布式缓存，也有类似的需求。

你可以在 Kubernetes 中运行这些类型的应用程序，但你需要围绕一个固有的冲突进行设计:Kubernetes 是一个动态环境，而数据量大的应用程序通常希望在一个稳定的环境中运行。群集应用程序希望在已知的网络地址中找到对等点，但在ReplicaSet 中不能很好地工作，而备份作业希望从磁盘驱动器中读取数据，因此在 persistentvolumecclaims 中不能很好地工作。如果你的应用程序有严格的数据要求，你需要对它进行不同的建模，我们将在本章中介绍一些更高级的控制器:StatefulSets, Jobs 和 CronJobs。

## 8.1 Kubernetes 如何用 StatefulSets 建模稳定性

StatefulSet 是一个具有可预测管理特性的 Pod 控制器:它允许您在一个稳定的框架内大规模运行应用程序。当您部署 ReplicaSet 时，它会使用随机名称创建 Pod，这些名称不能通过域名系统(DNS)单独寻址，并且它会并行启动它们。当你部署一个 StatefulSet 时，它会创建具有可预测名称的 Pod，这些 Pod 可以通过 DNS 单独访问，并按顺序启动它们;第一个 Pod 需要启动并运行在第二个 Pod 创建之前。

集群应用程序是 StatefulSet的一个很好的候选。它们通常设计有一个主实例和一个或多个辅助实例，这使它们具有高可用性。您可能能够扩展辅助服务器，但它们都需要到达主服务器，然后使用主服务器同步它们自己的数据。你不能用Deployment 来建模，因为在 ReplicaSet 中，没有办法将单个 Pod 识别为主节点，所以你最终会遇到多个主节点或零主节点的奇怪且不可预测的情况。

图 8.1 显示了一个例子，它可以用来运行我们在前几章中为待办事项列表应用程序使用的 Postgres 数据库，但是它使用了一个 StatefulSet 来实现复制数据和高可用性。

![图8.1](.\images\Figure8.1.png)
​<center>图 8.1 在 StatefulSet 中，每个 Pod 都可以从第一个 Pod 复制自己的数据副本</center>

为此的设置相当复杂，我们将分阶段花几个部分来完成，以便您了解工作中的 StatefulSet 的所有部分是如何组合在一起的。这种模式不仅对数据库有用——许多旧的应用程序是为静态运行时环境设计的，并且对稳定性做出了假设，但这些假设在Kubernetes 中并不成立。StatfulSets 允许你对这种稳定性建模，如果你的目标是将你现有的应用程序转移到 Kubernetes，那么它们可能是你在这一旅程的早期使用的东西。

让我们从一个简单的 StatefulSet 开始，它展示了基础知识。清单 8.1 显示了 StatefulSets 具有与其他 Pod 控制器几乎相同的 spec，除了它们还需要包含服务的名称。

> 清单 8.1	todo-db.yaml, 一个简单的 StatefulSet

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: todo-db
spec:
  selector: # StatefulSets 使用了相同的选择器机制
    matchLabels:
	  app: todo-db
  serviceName: todo-db # StatefulSets 必须关联 Service.
  replicas: 2
  template:
	# pod spec...
```

当您部署这个 YAML 文件时，您将得到一个运行两个 Postgres pod 的 StatefulSet，但不要太兴奋——它们只是两个单独的数据库服务器，恰好由同一个控制器管理。要使两个 pod 成为一个复制的数据库集群，还需要做更多的工作，我们将在接下来的几节中实现这一点。

<b>现在就试试</b> 部署清单 8.1 中的 StatefulSet，看看它创建的 pod 与 ReplicaSet 管理的 pod 有什么不同

```
# 切换到本章的源码目录:
cd ch08
# 部署 StatefulSet, Service, 以及用于 Postgres 密码的 Secret:
kubectl apply -f todo-list/db/
# 检查 StatefulSet:
kubectl get statefulset todo-db
# 检查 Pods:
kubectl get pods -l app=todo-db
# 查看 Pod 0 的主机名:
kubectl exec pod/todo-db-0 -- hostname
# 检查 Pod 1 的日志:
kubectl logs todo-db-1 --tail 1
```

从图 8.2 中可以看到，StatefulSet 的工作方式与 ReplicaSet 或 DaemonSet 非常不同。Pod 有一个可预测的名称，即 StatefulSet 名称后面跟着 Pod 的索引，因此您可以使用它们的名称来管理 Pod，而不必使用标签选择器。

Pod 仍然由控制器管理，但以一种比 ReplicaSet 更可预测的方式。Pod 是按照从 0 到 n 的顺序创建的;如果您缩小集合，控制器将以相反的顺序删除它们，从 n 开始向下。如果删除 Pod，控制器将创建一个替换。它的名称和配置将与原来的相同，但它将是一个新的Pod。

![图8.2](.\images\Figure8.2.png)
<center>图 8.2 StatefulSet 可以为集群应用程序创建环境，但应用程序需要自行配置</center>

<b>现在就试试</b> 删除Pod 0 的 StatefulSet，并看到 Pod 0 再次返回

```
# 检查 Pod 0 的内部 id:
kubectl get pod todo-db-0 -o jsonpath='{.metadata.uid}'
# 删除 Pod:
kubectl delete pod todo-db-0
# 检查 Pods:
kubectl get pods -l app=todo-db
# 检查新的 Pod 完全是全新的:
kubectl get pod todo-db-0 -o jsonpath='{.metadata.uid}'
```

你可以在图 8.3 中看到，StatefulSet 为 app 提供了一个稳定的环境。Pod 0 被替换为一个相同的Pod 0，但这不会触发一个全新的设置; 原来的 Pod 1仍然存在。仅在创建和扩展时应用顺序，不用于替换缺失的 Pod。

[图8.3](.\images\Figure8.3.png)
<center>图 8.3 StatefulSets 替换丢失的副本，就像它们之前的模样</center>

StatefulSet 只是对稳定环境建模的第一部分。您可以获取每个 Pod 的 DNS 名称，将 StatefulSet 链接到一个服务，这意味着您可以配置 Pod，通过使用已知地址上的其他副本来初始化它们自己。

## 8.2 在 StatefulSets 中使用 init 容器引导 Pod

Kubernetes API 从其他对象中组合对象: 在 StatefulSet 定义中的 Pod 模板与在 Deployment 模板和 裸Pod定义中使用的对象类型相同。这意味着 Pod 的所有功能都可以用于 StatefulSets，尽管 Pod 本身是以不同的方式管理的。我们在第7章学习了 init 容器，对于集群应用程序中经常需要的复杂初始化步骤，它们是一个完美的工具。

清单 8.2 显示了用于更新 Postgres 部署的第一个init容器。这个Pod Spec 中的多个 init 容器是按顺序运行的，而且由于 Pod 也是按顺序启动的，所以可以保证Pod 1 中的第一个init容器在 Pod 0 完全初始化并准备就绪之前不会运行。

> 清单8.2 todo-db.yaml，复制的 Postgres 设置初始化

```
initContainers:
  - name: wait-service
    image: kiamol/ch03-sleep
    envFrom: # env 文件用于在容器之间共享
      - configMapRef:
        name: todo-db-env
    command: ['/scripts/wait-service.sh']
    volumeMounts:
      - name: scripts # 卷从 ConfigMap 装载脚本
        mountPath: "/scripts"
```

在这个 init 容器中运行的脚本有两个功能: 如果它在 Pod 0 中运行，它只是打印一个日志来确认这是数据库主容器，然后容器退出;如果它在任何其他 Pod 中运行，它会对主 Pod 进行 DNS 查找调用，以确保在继续之前可以访问它。下一个init 容器将启动复制进程，因此这个容器将确保一切就绪。

本例中的具体步骤是特定于 Postgres 的，但许多集群和复制应用程序的模式是相同的——mysql、Elasticsearch、RabbitMQ和nat都有大致相似的需求。图 8.4 显示了如何在 StatefulSet 中使用init容器对该模式建模。

[图8.4](.\images\Figure8.png)
<center>图 8.4 StatefulSet的稳定环境保证了您可以在初始化中使用</center>

您可以通过在 spec 中标识服务来为 StatefulSet 中的各个 Pods 定义 DNS 名称，但它需要是 headless Service 的特殊配置。清单 8.3 显示了如何在没有 ClusterIP 地址的情况下配置数据库服务，并为 Pods 配置一个选择器。

> 清单8.3 todo-db-service。yaml，一个 StatefulSet 的 headless Service

```
apiVersion: v1
kind: Service
metadata:
  name: todo-db
spec:
  selector:
    app: todo-db # Pod 选择器与 StatefulSet 相匹配.
  clusterIP: None # Service 将不会拥有自己的 Ip 地址.
  ports:
    # ports follow
```

没有 ClusterIP 的 Service 仍然可以作为集群中的 DNS 条目，但它不会为该服务使用固定的 IP 地址。没有由网络层路由到真实目的地的虚拟 IP。相反，服务的 DNS 条目为 StatefulSet 中的每个 Pod 返回一个 IP 地址，并且每个 Pod 还获得自己的 DNS 条目。

<b>现在就试试</b> 我们已经部署了 headless Service，所以我们可以使用 sleep Deployment 来查询 StatefulSet 的 DNS，看看它与典型的 ClusterIP 服务相比如何

```
# 查看 Service 明细:
kubectl get svc todo-db
# 运行一个 sleep Pod 用于网络查找:
kubectl apply -f sleep/sleep.yaml
# 运行 Service 名称的 DNS 查询:
kubectl exec deploy/sleep -- sh -c 'nslookup todo-db | grep "^[^*]"'
# 运行 Pod 0 的 DNS 查找:
kubectl exec deploy/sleep -- sh -c 'nslookup todo-db-0.todo-
  db.default.svc.cluster.local | grep "^[^*]"'
```

在本练习中，您将看到 Service 的 DNS 查找返回两个 IP 地址，它们是 Pod 内部IP。Pods 本身有自己的格式为 pod-name 的 DNS 条目。带有通常集群域后缀的服务名称。图 8.5 显示了我的输出。

![图8.5](.\images\Figure8.5.png)
</center>图 8.5 StatefulSets 为每个 Pod 提供自己的 DNS 条目，因此它们是单独可寻址的</center>

可预测的启动顺序和单独可寻址的 Pod 是在 StatefulSet 中初始化集群应用的基础。具体细节在不同的应用程序之间会有很大的不同，但从广义上讲，Pod 的启动逻辑将是这样的: 如果我是Pod 0，那么我是主设备，所以我做所有的主设备设置工作;否则，我是一个辅助，所以我将给主服务器一些时间来设置，检查一切都正常工作，然后使用 Pod 0 地址进行同步。

Postgres 的实际设置相当复杂，所以我在这里略过。它使用 ConfigMaps 中的脚本和 init 容器来设置主服务器和备用服务器。到目前为止，我在 StatefulSet 的 spec 中使用了书中介绍的各种技术，值得探索，但脚本的细节都是特定于Postgres 的。


<b>现在就试试</b> 更新数据库，使其成为一个复制的设置。ConfigMaps 中有配置文件和启动脚本，并且更新 StatefulSet 以在 init 容器中使用它们

```
# 部署副本方式的 StatefulSet 应用设置:
kubectl apply -f todo-list/db/replicated/

# 等待 Pods 就绪
kubectl wait --for=condition=Ready pod -l app=todo-db

# 检查 Pod 0日志—主节点:
kubectl logs todo-db-0 --tail 1

# 以及 Pod 1—从节点:
kubectl logs todo-db-1 --tail 2
```

Postgres 使用主-被动模型进行复制，因此主服务器用于数据库读写，从服务器从主服务器同步数据，可供客户端使用，但仅用于读访问。图 8.6 显示了 init 容器如何识别每个 Pod 的角色并初始化它们。

![图8.6](.\images\Figure8.6.png)
<center>图 8.6 Pod 是副本，但它们可以有不同的行为，使用 init 容器选择一个角色</center>

像这样初始化复制应用的大部分复杂性都是围绕工作流建模，这是特定于应用的。这里的 init 容器脚本使用 pg_isready 工具来验证主应用是否准备好接收连接，并使用 pb_basebackup 工具来启动复制。这些实现细节从管理系统的操作员那里抽象出来。他们可以通过扩大 StatefulSet 来添加更多的副本，就像使用任何其他复制控制器一样。

<b>现在就试试</b> 扩大数据库以添加另一个副本，并确认新的 Pod 也作为备用启动

```
# 添加另一个副本:
kubectl scale --replicas=3 statefulset/todo-db

# 等待 Pod 2 就绪
kubectl wait --for=condition=Ready pod -l app=todo-db

# 检查新 Pod 是否设置为另一个备用:
kubectl logs todo-db-2 --tail 2
```

我不认为这是一个企业级的产品设置，但这是一个很好的起点，真正的 Postgres 专家可以接管它。现在，您有了一个功能良好的复制 Postgres 数据库集群，其中有一个主数据库和两个备用数据库——Postgres称它们为备用数据库。如图8.7所示，所有备用服务器都以相同的方式启动，从主服务器同步数据，客户端都可以使用它们进行只读访问。

![图8.7](.\images\Figure8.7.png)
<center>图 8.7 使用单独可寻址的 pod 意味着辅助设备总能找到主设备</center>

这里遗漏了一个明显的部分——数据的实际存储。我们所拥有的设置并不是真正可用的，因为它没有任何用于存储的卷，因此每个数据库容器都在自己的可写层中写入数据，而不是在持久卷中写入数据。StatefulSets 有一种定义卷需求的简洁方法:您可以在 spec 中包含一组持久卷声明(PVC)模板。

## 8.3 使用卷声明模板请求存储

Volumes are part of the standard Pod spec, and you can load ConfigMaps and Secrets into the Pods for a StatefulSet. You can even include a PVC and mount it into the app container, but that gives you volumes that are shared among all the Pods. That’s fine for read-only configuration settings where you want every Pod to have the same data, but if you mount a standard PVC for data storage, then every Pod will try to write to the same volume.

​	You actually want each Pod to have its own PVC, and Kubernetes provides that for StatefulSets with the volumeClaimTemplates field in the spec. Volume claim templates can include a storage class as well as capacity and access mode requirements. When you deploy a StatefulSet with volume claim templates, it creates a PVC for each Pod, and they’re linked, so if Pod 0 is replaced, the new Pod 0 will attach to the PVC used by the previous Pod 0.

**Listing 8.4	sleep-with-pvc.yaml, a StatefulSet with volume claim templates**

```
spec:
  selector:
    # pod selector...
  serviceName:
    # headless service name...
  replicas: 2
  template:
    # pod template...
    
  volumeClaimTemplates:
    - metadata:
        name: data # The name to use for volume mounts in the Pod
      spec: # This is a standard PVC spec.
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 5Mi
```

We’ll use this exercise to see how volume claim templates in StatefulSets work in a simple environment before adding them as the storage layer for our database cluster.

​	**TRY IT NOW	Deploy the StatefulSet from listing 8.4, and explore the PVCs it creates.**

```
# deploy the StatefulSet with volume claim templates:
kubectl apply -f sleep/sleep-with-pvc.yaml

# check that the PVCs are created:
kubectl get pvc

# write some data to the PVC mount in Pod 0:
kubectl exec sleep-with-pvc-0 -- sh -c 'echo Pod 0 > /data/pod.txt'

# confirm Pod 0 can read the data:
kubectl exec sleep-with-pvc-0 -- cat /data/pod.txt

# confirm Pod 1 can’t—this will fail:
kubectl exec sleep-with-pvc-1 -- cat /data/pod.txt
```

You’ll see that each Pod in the set gets a PVC created dynamically, which in turn creates a PersistentVolume using the default storage class (or the requested storage class, if I had included one in the spec). The PVCs all have the same configuration, and they use the same stable approach as Pods in the StatefulSet: they have a predictable name, and, as you see in figure 8.8, each Pod has its own PVC, giving the replicas independent storage.

![图8.8](.\images\Figure8.8.png)
<center>图 8.8 Volume claim templates dynamically create storage for Pods in StatefulSets</center>

The link between the Pod and its PVC is maintained when Pods are replaced, which is what really gives StatefulSets the power to run data-heavy applications. When you roll out an update to your app, the new Pod 0 will attach to the PVC from the previous Pod 0, and the new app container will have access to the exact same state as the replaced app container.

​	**TRY IT NOW	Trigger a Pod replacement by removing Pod 0. It will be replaced with another Pod 0 that attaches to the same PVC.**

```
# delete the Pod:
kubectl delete pod sleep-with-pvc-0

# check that the replacement gets created:
kubectl get pods -l app=sleep-with-pvc

# check that the new Pod 0 can see the old data:
kubectl exec sleep-with-pvc-0 -- cat /data/pod.txt
```

This simple example makes this clear—you can see in figure 8.9 that the new Pod 0 has access to all the data from the original Pod. In a production cluster, you would specify a storage class that uses a volume type that any node can access, so replace- ment Pods can run on any node, and the app container can still mount the PVC.

![图8.9](.\images\Figure8.9.png)
<center>图 8.9 Stability in a StatefulSet extends to preserving the PVC link between Pod replacements</center>

Volume claim templates are the final piece we need to add to the Postgres deployment to model a fully reliable database. StatefulSets are intended to present a stable environment for your app, so they’re less flexible than other controllers when it comes to updates—you can’t update an existing StatefulSet and make a fundamental change, like adding volume claims. You need to make sure your design meets the app requirements for a StatefulSet because it’s hard to maintain service levels during big changes.

​	**TRY IT NOW	We’ll update the Postgres deployment, but first we need to remove the existing StatefulSet.**

```
# apply the update with volume claim templates—this will fail:
kubectl apply -f todo-list/db/replicated/update/todo-db-pvc.yaml

# delete the existing set:
kubectl delete statefulset todo-db

# create a new one with volume claims:
kubectl apply -f todo-list/db/replicated/update/todo-db-pvc.yaml

# check the volume claims:
kubectl get pvc -l app=todo-db
```

When you run this exercise, you should see clearly how the StatefulSet preserves order and waits for each Pod to be running before it starts the next Pod. PVCs are created for each Pod in sequence, too, as you can see from my output, shown in figure 8.10.

![图8.10](.\images\Figure8.10.png)
<center>图 8.10 PVCs are created and allocated to the Postgres Pods</center>

It feels like we’ve spent a long time on StatefulSets, but it’s a topic you should understand well so you’re not taken by surprise when someone asks you to move their database to Kubernetes (which they will). StatefulSets come with a good deal of complexity, and you’ll avoid using them most of the time. But if you are looking to migrate existing apps to Kubernetes, StatefulSets could be the difference between being able to run everything on the same platform or having to keep a handful of VMs just to run one or two apps.

​	We’ll finish the section with an exercise to show the power of our clustered database. The Postgres secondary replicates all the data from the primary, and it can be used by clients for read-only access. If we had a serious production issue with our todo list app that was causing it to lose data, we have the option to switch to read-only mode and use the secondary while we investigate the problem. That keeps the app running safely with minimal functionality, which is definitely preferable to taking it offline.

​	**TRY IT NOW	Run the to-do web app and enter some items. In the default configuration, it connects to the Postgres primary in Pod 0 of the StatefulSet. Then we’ll switch the app configuration to put it into read-only mode. This makes it connect to the read-only Postgres standby in Pod 1, which has replicated all the data from Pod 0.**

```
# deploy the web app:
kubectl apply -f todo-list/web/

# get the URL for the app:
kubectl get svc todo-web -o
  jsonpath='http://{.status.loadBalancer.ingress[0].*}:8081/new'

# browse and add a new item

# switch to read-only mode, using the database secondary:
kubectl apply -f todo-list/web/update/todo-web-readonly.yaml

# refresh the app—the /new page is read-only;
# browse to /list and you'll see your original data

# check that there are no clients using the primary in Pod 0:
kubectl exec -it todo-db-0 -- sh -c "psql -U postgres -t -c 'SELECT
  datname, query FROM pg_stat_activity WHERE datid > 0'"

# check that the web app really is using the secondary in Pod 1:
kubectl exec -it todo-db-1 -- sh -c "psql -U postgres -t -c 'SELECT
  datname, query FROM pg_stat_activity WHERE datid > 0'"
```

You can see my output in figure 8.11, with some tiny screenshots to show the app running in read-only mode but still with access to all the data.

​	Postgres has existed as a SQL database engine since 1996—it predates Kubernetes by almost 25 years. Using a StatefulSet, you can model an application environment that suits Postgres and other clustered applications like it, providing stable networking, storage, and initialization in the dynamic world of containers.

## 8.4 使用 Jobs 和 CronJobs 运行维护任务

Data-intensive apps need replicated data with storage aligned to compute, and they usually also need some independent nurturing of the storage layer. Data backups and reconciliation are well suited to another type of Pod controller: the Job. Kubernetes Jobs are defined with a Pod spec, and they run the Pod as a batch job, ensuring it runs to completion.

​	Jobs aren’t just for stateful apps; they’re a great way to bring a standard approach to any batch-processing problems, where you can hand off all the scheduling and monitoring and retry logic to the cluster. You can run any container image in the Pod for a Job, but it should start a process that ends; otherwise, your jobs will keep running forever. Listing 8.5 shows a Job spec that runs the Pi application in batch mode.

![图8.11](.\images\Figure8.11.png)
<center>图 8.11 Switching an app to read-only mode is a useful option if there's a data issue</center>

**Listing 8.5	pi-job.yaml, a simple Job to calculate Pi**

```
apiVersion: batch/v1
kind: Job # Job is the object type.
metadata:
  name: pi-job
spec:
  template:
    spec: # The standard Pod spec
      containers:
        - name: pi # The container should run and exit.
          image: kiamol/ch05-pi
          command: ["dotnet", "Pi.Web.dll", "-m", "console", "-dp", "50"]
      restartPolicy: Never # If the container fails, replace the Pod.
```

The Job template contains a standard Pod spec, with the addition of a required restartPolicy field. That field controls the behavior of the Job in response to failure. You can choose to have Kubernetes restart the same Pod with a new container if the run fails or always create a replacement Pod, potentially on a different node. In a normal run of the Job where the Pod completes successfully, the Job and the Pod are retained so the container logs are available.

​	**TRY IT NOW	Run the Pi Job from listing 8.5, and check the output from the Pod.**

```
# deploy the Job:
kubectl apply -f pi/pi-job.yaml

# check the logs for the Pod:
kubectl logs -l job-name=pi-job

# check the status of the Job:
kubectl get job pi-job
```

Jobs add their own labels to the Pods they create. The job-name label is always added, so you can navigate to Pods from the Job. My output in figure 8.12 shows that the Job has had one successful completion and the calculation result is available in the logs.

![图8.12](.\images\Figure8.12.png)
<center>图 8.12 Jobs create Pods, make sure they complete, and then leave them in the cluster</center>

It’s always useful to have different options for computing Pi, but this is just a simple example. You can use any container image in the Pod spec so you can run any kind of batch process with a Job. You might have a set of input items that need the same work done on them; you can create one Job for the whole set, which creates a Pod for each item, and Kubernetes distributes the work all throughout the cluster. The Job spec supports this with the following two optional fields:

- completions—specifies how many times the Job should run. If your Job is processing a work queue, then the app container needs to understand how to fetch the next item to work on. The Job itself just ensures that it runs a number of Pods equal to the desired number of completions.

- parallelism—specifies how many Pods to run in parallel for a Job with multiple completions set. This setting lets you tweak the speed of running the Job, balancing that with the compute requirements on the cluster.

One last Pi example for this chapter: a new Job spec that runs multiple Pods in parallel, each computing Pi to a random number of decimal places. This spec uses an init container to generate the number of decimal places to use, and the app container reads that input using a shared EmptyDir mount. This is a nice approach because the app container doesn’t need to be modified to work in a parallel environment. You could extend this with an init container that fetched a work item from a queue, so the app itself wouldn’t need to be aware of the queue.

​	**TRY IT NOW	Run an alternative Pi Job that uses parallelism and shows that multiple Pods from the same spec can process different workloads.**

```
# deploy the new Job:
kubectl apply -f pi/pi-job-random.yaml

# check the Pod status:
kubectl get pods -l job-name=pi-job-random

# check the Job status:
kubectl get job pi-job-random

# list the Pod output:
kubectl logs -l job-name=pi-job-random
```

This exercise may take a while to run, depending on your hardware and the number of decimal places it generates. You’ll see all the Pods running in parallel, working on their own calculations. The final output will be three sets of Pi, probably to thousands of decimal places. I’ve abbreviated my results, shown in figure 8.13.

Jobs are a great tool to have in your pocket. They’re perfect for anything compute intensive or IO intensive, where you want to make sure a process completes but don’t mind when. You can even submit Jobs from your own application—a web app running in Kubernetes has access to the Kubernetes API server, and it can create Jobs to run work for users.

![图8.13](.\images\Figure8.13.png)
<center>图 8.13 Jobs can run multiple Pods from the same spec that each process different workloads</center>

​	The real power of Jobs is that they run in the context of the cluster, so they have all the cluster resources available to them. Back to the Postgres example, we can run a database-backup process in a Job, and the Pod it runs can access the Pods in the StatefulSet or the PVCs, depending on what it needs to do. That takes care of the nurturing aspect of these data-intensive apps, but those Jobs need to be run regularly, which is where the CronJob comes in. The CronJob is a Job controller, which creates Jobs on a regular schedule. Figure 8.14 shows the workflow.

​	CronJob specs include a Job spec, so you can do anything in a CronJob that you can do in a Job, including running multiple completions in parallel. The schedule for running the Job uses the Linux Cron format, which lets you express everything from simple “every minute” or “every day” schedules to more complex “at 4 a.m. and 6 a.m. every Sunday” routines. Listing 8.6 shows part of the CronJob spec for running database backups.

![图8.14](.\images\Figure8.14.png)
<center>图 8.14 CronJobs are the ultimate owner of the Job Pods, so everything can be removed with cascading deletes</center>

**Listing 8.6	todo-db-backup-cronjob.yaml, a CronJob for database backups**

```
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: todo-db-backup
spec:
  schedule: "*/2 * * * *"   # Creates a Job every 2 minutes
  concurrencyPolicy: Forbid # Prevents overlap so a new Job won’t be
  jobTemplate:              # created if the previous one is running
    spec:
      # job template...
```

The full spec uses the Postgres Docker image, with a command to run the pg_dump backup tool. The Pod loads environment variables and passwords from the same ConfigMaps and Secrets that the StatefulSet uses, so there’s no duplication in the config file. It also uses its own PVC as the storage location to write the backup files.

​	**TRY IT NOW	Create a CronJob from the spec in listing 8.6 to run a database backup Job every two minutes.**

```
# deploy the CronJob and target PVC for backup files:
kubectl apply -f todo-list/db/backup/

# wait for the Job to run—this is a good time to make tea:
sleep 150

# check the CronJob status:
kubectl get cronjob todo-db-backup

# now run a sleep Pod that mounts the backup PVC:
kubectl apply -f sleep/sleep-with-db-backup-mount.yaml

# check if the CronJob Pod created the backup:
kubectl exec deploy/sleep -- ls -l /backup
```

The CronJob is set to run every two minutes, so you’ll need to give it time to fire up during this exercise. On schedule, the CronJob creates a Job, which creates a Pod, which runs the backup command. The Job ensures the Pod completes successfully. You can confirm the backup file is written by mounting the same PVC in another Pod. You can see it all works correctly in figure 8.15.

![图8.15](.\images\Figure8.15.png)
<center>图 8.15 CronJobs run Pods, which can access other Kubernetes objects. This one connects to a database Pod</center>

CronJobs don’t perform an automatic cleanup for Pods and Jobs. The time-to-live(TTL) controller does this, but it’s an alpha-grade feature that isn’t available in many Kubernetes platforms. Without it you need to manually delete the child objects when you’re sure you no longer need them. You can also move CronJobs to a suspended state, which means the object spec still exists in the cluster, but it doesn’t run until the CronJob is activated again.

​	**TRY IT NOW	Suspend the CronJob so it doesn’t keep creating backup Jobs, and then explore the status of the CronJob and its Jobs.**

```
# update the CronJob, and set it to suspend:
kubectl apply -f todo-list/db/backup/update/todo-db-backup-cronjob-
  suspend.yaml
  
# check the CronJob:
kubectl get cronjob todo-db-backup

# find the Jobs owned by the CronJob:
kubectl get jobs -o jsonpath="{.items[?(@.metadata.ownerReferences[0]
  .name=='todo-db-backup')].metadata.name}"
```

If you explore the object hierarchy, you’ll see that CronJobs don’t follow the standard controller model, with a label selector to identify the Jobs it owns. You can add your own labels in the Job template for the CronJob, but if you don’t do that, you need to identify Jobs where the owner reference is the CronJob, as shown in figure 8.16.

![图8.16](.\images\Figure8.16.png)
<center>图 8.16 CronJobs don’t use a label selector to model ownership, because they don’t keep track of Jobs</center>

As you start to make more use of Jobs and CronJobs, you’ll realize that the simplicity of the spec masks some complexity in the process and presents some interesting failure modes. Kubernetes does its best to make sure your batch jobs start when you want them to and run to completion, which means your containers need to be resilient.Completing a Job might mean restarting a Pod with a new container or replacing the Pod on a new node, and for CronJobs, multiple Pods could be running if the process takes longer than the schedule interval. Your container logic needs to allow for all those scenarios.

Now you know how to run data-heavy apps in Kubernetes, with StatefulSets to model a stable runtime environment and initialize the app, and CronJobs to process data backups and other regular maintenance work. We’ll close out the chapter thinking about whether this is really a good idea.

## 8.5 为有状态应用程序选择平台

The great promise of Kubernetes is that it gives you a single platform that can run all your apps, on any infrastructure. It’s hugely appealing to think that you can model all
the aspects of any application in a chunk of YAML, deploy it with some kubectl commands, and know it will run in the same way on any cluster, taking advantage of all the extended features the platform offers. But data is precious and usually irreplaceable,so you need to think carefully before you decide that Kubernetes is the place to run data-heavy apps.

Figure 8.17 shows the full setup we’ve built in this chapter to run an almost-production-grade SQL database in Kubernetes. Just look at all the moving parts—do


you really want to manage all that? And how much time will you need to invest just testing this setup with your own data sizes: validating that the replicas are syncing correctly, verifying the backups can be restored, running chaos experiments to be sure that failures are handled in the way you expect?

![图8.17](.\images\Figure8.17.png)
​	<center>图 8.17 Yikes! And this is a simplification that doesn’t show volumes or init containers</center>

​	Compare that to a managed database in the cloud. Azure, AWS, and GCP all offer managed services for Postgres, MySQL, and SQL Server, as well as their own custom cloud-scale databases. The cloud provider takes care of scale and high availability, including features for backups to cloud storage and more advanced options like threat detection. An alternative architecture just uses Kubernetes for compute and plugs in to managed cloud services for data and communication.

​	Which is the better option? Well, I’m a consultant by day, and I know the only real answer is: “It depends.” If you’re running in the cloud, then I think you need a very good reason not to use managed services in production, where data is critical. In nonproduction environments, it often makes sense to run equivalent services in Kubernetes instead, so you run a containerized database and message queue in your development and test environments for lower costs and ease of deployment and swap out to managed versions in production. Kubernetes makes a swap like that very simple, with all the Pod and Service configuration options.

​	In the data center, the picture is a little different. If you’re already invested in running Kubernetes on your own infrastructure, you’re taking on a lot of management, and it might make sense to maximize utilization of your clusters and use them for everything. If you choose to go that way, Kubernetes gives you the tools to migrate data-heavy apps into the cluster and run them with the levels of availability and scale that you need. Just don’t underestimate the complexity of getting there.

​	We’re done with StatefulSets and Jobs now, so we can clean up before going on to the lab.

​	**TRY IT NOW	All the top-level objects are labeled, so we can remove everything with cascading deletes.**

```
# delete all the owning objects:
kubectl delete all -l kiamol=ch08

# delete the PVCs
kubectl delete pvc -l kiamol=ch08
```

## 8.6 实验室

So, how much of your lunchtime do you have left for this lab? Can you model a MySQL database from scratch, with backups? Probably not, but don’t worry—this lab isn’t as involved as that. The goal is just to give you some experience working with StatefulSets and PVCs, so we’ll use a much simpler app. You’re going to run the Nginx web server in a StatefulSet, where each Pod writes log files to a PVC of its own, and then you’ll run a Job that prints the size of each Pod’s log file. The basic pieces are there for you, so it’s about applying some of the techniques from the chapter.

- The starting point is the Nginx spec in ch08/lab/nginx, which runs a single Pod writing logs to an EmptyDir volume.
- The Pod spec needs to be migrated to a StatefulSet definition, which is configured to run with three Pods and provide separate storage for each of them.

- When you have the StatefulSet working, you should be able to make calls to your Service and see the log files being written in the Pods.
- Then you can complete the Job spec in the file disk-calc-job.yaml, adding the volume mounts so it can read the log files from the Nginx Pods.

It’s not as bad as it looks, and it will get you thinking about storage and Jobs. My solution is on GitHub for you to check in the usual place: https://github.com/sixeyed/kiamol/blob/master/ch08/lab/README.md.

