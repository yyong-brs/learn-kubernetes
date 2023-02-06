# 第八章 使用 StatfulSets 和 Jobs 运行数据量大的应用

“数据量大” 不是一个很科学的术语，但本章是关于运行一个应用程序类，它不仅是有状态的，而且还要求它如何使用状态。数据库就是此类的一个例子。它们需要跨多个实例运行以实现高可用性，每个实例都需要一个本地数据存储以实现快速访问，而且这些独立的数据存储需要保持同步。数据有其自身的可用性要求，您需要定期运行备份以防止终端故障或损坏。其他数据密集型应用程序，如消息队列和分布式缓存，也有类似的需求。

你可以在 Kubernetes 中运行这些类型的应用程序，但你需要围绕一个固有的冲突进行设计:Kubernetes 是一个动态环境，而数据量大的应用程序通常希望在一个稳定的环境中运行。群集应用程序希望在已知的网络地址中找到对等点，但在ReplicaSet 中不能很好地工作，而备份作业希望从磁盘驱动器中读取数据，因此在 persistentvolumecclaims 中不能很好地工作。如果你的应用程序有严格的数据要求，你需要对它进行不同的建模，我们将在本章中介绍一些更高级的控制器:StatefulSets, Jobs 和 CronJobs。

## 8.1 Kubernetes 如何用 StatefulSets 建模稳定性

A StatefulSet is a Pod controller with predictable management features: it lets you run applications at scale within a stable framework. When you deploy a ReplicaSet,it creates Pods with random names, which are not individually addressable over the domain name system (DNS), and it starts them in parallel. When you deploy a StatefulSet, it creates Pods with predictable names, which can be individually accessed over DNS, and starts them in order; the first Pod needs to be up and running
before the second Pod is created.

​Clustered applications are a great candidate for StatefulSets. Typically they’re designed with a primary instance and one or more secondaries, which gives them high availability. You might be able to scale the secondaries, but they all need to reach the primary and then use it to synchronize their own data. You can’t model that with a Deployment because in the ReplicaSet, there is no way to identify a single Pod as the primary, so you’d end up with bizarre and unpredictable conditions with multiple primaries or zero primaries.

​	Figure 8.1 shows an example of that, which could be used to run the Postgres database we’ve used in previous chapters for the to-do list application, but it uses a StatefulSet to achieve replicated data and high availability.

![图8.1](.\images\Figure8.1.png)
​<center>图 8.1 In a StatefulSet,each Pod can have its own copy of data replicated from the first Pod</center>

The setup for this is quite involved, and we’ll spend a couple of sections getting there in stages, so you learn how all the pieces of a working StatefulSet fit together. It’s a pattern that is useful for more than just databases—many older applications were designed for a static runtime environment and made assumptions about stability that don’t hold true in Kubernetes. StatefulSets allow you to model that stability, and if your goal is to move your existing apps to Kubernetes, then they may be something you use early in that journey.

​	Let’s start with a simple StatefulSet that shows the basics. Listing 8.1 shows that StatefulSets have pretty much the same specs as other Pod controllers, except that they also need to include the name of a Service.

**Listing 8.1	todo-db.yaml, a simple StatefulSet**

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: todo-db
spec:
  selector: # StatefulSets use the same selector mechanism.
    matchLabels:
	  app: todo-db
  serviceName: todo-db # StatefulSets must be linked to a Service.
  replicas: 2
  template:
	# pod spec...
```

When you deploy this YAML file, you’ll get a StatefulSet running two Postgres pods,but don’t get too excited—they’re just two separate database servers that happen to be managed by the same controller. There’s more work needed to get two Pods to be a replicated database cluster, and we’ll get there over the next few sections.

​	**TRY IT NOW	Deploy the StatefulSet from listing 8.1, and see how the Pods it creates compare to Pods managed by a ReplicaSet.**

```
# switch to the chapter's source:
cd ch08
# deploy the StatefulSet, Service, and a Secret for the Postgres
# password:
kubectl apply -f todo-list/db/
# check the StatefulSet:
kubectl get statefulset todo-db
# check the Pods:
kubectl get pods -l app=todo-db
# find the hostname of Pod 0:
kubectl exec pod/todo-db-0 -- hostname
# check the logs of Pod 1:
kubectl logs todo-db-1 --tail 1
```

You can see from figure 8.2 that a StatefulSet works in a very different way from a ReplicaSet or a DaemonSet. The Pods have a predictable name, which is the StatefulSet name followed by the index of the Pod, so you can manage the Pods using their names instead of having to use a label selector.

​	The Pods are still managed by the controller, but in a more predictable way than with a ReplicaSet. Pods are created in order from zero up to n; if you scale down the set, the controller will remove them in the reverse order, starting from n and working down. If you delete a Pod, the controller will create a replacement. It will have the same name and configuration as the original, but it will be a new Pod.

![图8.2](.\images\Figure8.2.png)
<center>图 8.2 A StatefulSet can create the environment for a clustered application, but the app needs to configure itself</center>

​	**TRY IT NOW	Delete Pod 0 of the StatefulSet, and see that Pod 0 comes back again.**

```
# check the internal ID of Pod 0:
kubectl get pod todo-db-0 -o jsonpath='{.metadata.uid}'
# delete the Pod:
kubectl delete pod todo-db-0
# check Pods:
kubectl get pods -l app=todo-db
# check that the new Pod is a new Pod:
kubectl get pod todo-db-0 -o jsonpath='{.metadata.uid}'
```

You can see in figure 8.3 that a StatefulSet provides a stable environment for the app.Pod 0 is replaced with an identical Pod 0, but that doesn’t trigger a whole new set; the original Pod 1 remains. Ordering is applied only for creation and scaling, not for replacing missing Pods.

​	The StatefulSet is only the first part of modeling a stable environment. You can get DNS names for each Pod linking the StatefulSet to a service, and that means you can configure Pods to initialize themselves by working with other replicas at known addresses.

![图8.3](.\images\Figure8.3.png)
<center>图 8.3 StatefulSets replace missing replicas exactly as they were</center>

## 8.2 在 StatefulSets 中使用 init 容器引导 Pod

The Kubernetes API composes objects from other objects: the Pod template in a StatefulSet definition is the same object type you use in the template for a Deployment and in a bare Pod definition. That means all the Pod features are available for StatefulSets even though the Pods themselves are managed in a different way. We learned about init containers in chapter 7, and they’re a perfect tool for the complicated initialization steps you often need in clustered applications.

​	Listing 8.2 shows the first init container for an update to the Postgres deployment.Multiple init containers in this Pod spec run in sequence, and because the Pods also start in sequence, you can guarantee that the first init container in Pod 1 won’t run until Pod 0 is fully initialized and ready.

**Listing 8.2	todo-db.yaml, the replicated Postgres setup with initialization**

```
initContainers:
  - name: wait-service
    image: kiamol/ch03-sleep
    envFrom: # env file for sharing between containers
      - configMapRef:
        name: todo-db-env
    command: ['/scripts/wait-service.sh']
    volumeMounts:
      - name: scripts # Volume loads scripts from ConfigMap.
        mountPath: "/scripts"
```

The script that runs in this init container has two functions: if it’s running in Pod 0, it just prints a log to confirm that this is the database primary, and then the container exits; if it’s running in any other Pod, it makes a DNS lookup call to the primary, to make sure it’s accessible before continuing. The next init container will start the replication process, so this one makes sure everything is in place.

​	The exact steps in this example are specific to Postgres, but the pattern is the same for many clustered and replicated applications—MySQL, Elasticsearch, RabbitMQ,and NATS all have broadly similar requirements. Figure 8.4 shows how you can model that pattern using init containers in a StatefulSet.

![图8.4](.\images\Figure8.4.png)
</center>图 8.4 The stable environment of a StatefulSet gives guarantees you can use in initialization</center>

You define DNS names for the individual Pods in a StatefulSet by identifying a Service in the spec, but it needs to be a special configuration of headless Service. Listing 8.3 shows how the database Service is configured with no ClusterIP address and with a selector for the Pods.

**Listing 8.3	todo-db-service.yaml, a headless Service for a StatefulSet**

```
apiVersion: v1
kind: Service
metadata:
  name: todo-db
spec:
  selector:
    app: todo-db # The Pod selector matches the StatefulSet.
  clusterIP: None # The service will not get its own IP address.
  ports:
    # ports follow
```

A Service with no ClusterIP is still available as a DNS entry in the cluster, but it doesn’t use a fixed IP address for the Service. There’s no virtual IP that is routed to the real destination by the networking layer. Instead, the DNS entry for the service returns an IP address for each Pod in the StatefulSet, and each Pod additionally gets its own DNS entry.

​	**TRY IT NOW	We’ve already deployed the headless Service, so we can use a sleep Deployment to query DNS for the StatefulSet and see how it compares to a typical ClusterIP service.**

```
# show the Service details:
kubectl get svc todo-db
# run a sleep Pod to use for network lookups:
kubectl apply -f sleep/sleep.yaml
# run a DNS query for the Service name:
kubectl exec deploy/sleep -- sh -c 'nslookup todo-db | grep "^[^*]"'
# run a DNS lookup for Pod 0:
kubectl exec deploy/sleep -- sh -c 'nslookup todo-db-0.todo-
  db.default.svc.cluster.local | grep "^[^*]"'
```

You’ll see in this exercise that the DNS lookup for the service returns two IP addresses, which are the internal Pod IPs. The Pods themselves have their own DNS entry in the format pod-name.service-name with the usual cluster domain suffix. Figure 8.5 shows my output.

​	Predictable startup order and individually addressable Pods are the foundation for initializing a clustered app in a StatefulSet. The details will differ wildly between applications, but broadly, the startup logic for the Pod will be something like this: if I am Pod 0, then I’m the primary, so I do all the primary setup stuff; otherwise, I’m a secondary, so I’ll give the primary some time to get set up, check that everything’s working, and then synchronize using the Pod 0 address.

The actual setup for Postgres is quite involved, so I’ll skip over it here. It uses scripts in ConfigMaps with init containers to set up the primary and secondaries. I use various techniques we’ve covered in the book so far in the spec for the StatefulSet, which is worth exploring, but the details of the scripts are all specific to Postgres.

![图8.5](.\images\Figure8.5.png)
</center>图 8.5 StatefulSets give each Pod its own DNS entry, so they are individually addressable</center>

​	**TRY IT NOW	Update the database to make it a replicated setup. There are configuration files and startup scripts in ConfigMaps, and the StatefulSet is updated to use them in init containers.**

```
# deploy the replicated StatefulSet setup:
kubectl apply -f todo-list/db/replicated/

# wait for the Pods to spin up
kubectl wait --for=condition=Ready pod -l app=todo-db

# check the logs of Pod 0—the primary:
kubectl logs todo-db-0 --tail 1

# and of Pod 1—the secondary:
kubectl logs todo-db-1 --tail 2
```

Postgres uses an active-passive model for replication, so the primary is used for database reads and writes, and the secondaries sync data from the primary and can be used by clients, but only for read access. Figure 8.6 shows how the init containers recognize the role for each Pod and initialize them.

![图8.6](.\images\Figure8.6.png)
<center>图 8.6 Pods are replicas, but they can have different behavior, using init containers to choose a role</center>

Most of the complexity in initializing replicated apps like this is around modelling the workflow, which is specific to the app. The init container scripts here use the pg_isready tool to verify that the primary is ready to receive connections and the pb_basebackup tool to start the replication. Those implementation details are abstracted away from operators managing the system. They can add more replicas by scaling up the StatefulSet, like with any other replication controller.

​	**TRY IT NOW	Scale up the database to add another replica, and confirm that the new Pod also starts as a secondary.**

```
# add another replica:
kubectl scale --replicas=3 statefulset/todo-db

# wait for Pod 2 to spin up
kubectl wait --for=condition=Ready pod -l app=todo-db

# check that the new Pod sets itself up as another secondary:
kubectl logs todo-db-2 --tail 2
```

I wouldn’t call this an enterprise-grade production setup, but it’s a good starting point where a real Postgres expert could take over. You now have a functional, replicated Postgres database cluster with a primary and two secondaries—Postgres calls them standbys. As you can see in figure 8.7, all the standbys start in the same way, syncing data from the primary, and they can all be used by clients for read-only access.

![图8.7](.\images\Figure8.7.png)
<center>图 8.7 Using individually addressable Pods means secondaries can always find the primary</center>

One obvious part is missing here—the actual storage of the data. The setup we have isn’t really usable because it doesn’t have any volumes for storage, so each database container writes data in its own writable layer, not in a persistent volume. StatefulSets have a neat way of defining volume requirements: you can include a set of Persistent Volume Claim (PVC) templates in the spec.

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

