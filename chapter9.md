# 第九章 通过 rollouts 和 rollbacks 管理应用发布

You’ll update existing apps far more often than you’ll deploy something new. Containerized apps inherit multiple release cadences from the base images they use; official images on Docker Hub for operating systems, platform SDKs, and runtimes typically have a new release every month. You should have a process to rebuild your images and release updates whenever those dependencies get updated, because they could contain critical security patches. Key to that process is being able to roll out an update safely and give yourself options to pause and roll back the update if it goes wrong. Kubernetes has those scenarios covered for Deployments, DaemonSets, and StatefulSets.

​	A single update approach doesn’t work for every type of application, so Kubernetes provides different update strategies for the controllers and options to tune how the strategies work. We’ll explore all those options in this chapter. If you’re thinking of skipping this one because you’re not excited by the thought of 6,000 words on application updates, I’d recommend sticking with it. Updates are the biggest cause of application downtime, but you can reduce the risk significantly if you understand the tools Kubernetes gives you. And I’ll try to inject a little excitement along the way.

## 9.1 Kubernetes 如何管理 rollouts

We’ll start with Deployments—actually, you’ve already done plenty of Deployment updates. Every time we’ve applied a change to an existing Deployment (something we do 10 times a chapter), Kubernetes has implemented that with a rollout. In a rollout, the Deployment creates a new ReplicaSet and scales it up to the desired number of replicas, while scaling down the previous ReplicaSet to zero replicas. Figure 9.1 shows an update in progress.

![图9.1](.\images\Figure9.1.png)

<center>图 9.1 Deployments control multiple ReplicaSets so they can manage rolling updates</center>

Rollouts aren’t triggered from every change to a Deployment, only from a change to the Pod spec. If you make a change that the Deployment can manage with the current ReplicaSet, like updating the number of replicas, that’s done without a rollout.

​	**TRY IT NOW	Deploy a simple app with two replicas, then update it to increase scale, and see how the ReplicaSet is managed.**

```
# change to the exercise directory:
cd ch09

# deploy a simple web app:
kubectl apply -f vweb/

# check the ReplicaSets:
kubectl get rs -l app=vweb

# now increase the scale:
kubectl apply -f vweb/update/vweb-v1-scale.yaml

# check the ReplicaSets:
kubectl get rs -l app=vweb

# check the deployment history:
kubectl rollout history deploy/vweb
```

The kubectl rollout command has options to view and manage rollouts. You can see from my output in figure 9.2 that there’s only one rollout in this exercise, which was the initial deployment that created the ReplicaSet. The scale update changed only the existing ReplicaSet, so there was no second rollout.

![图9.2](.\images\Figure9.2.png)
<center>图 9.2	Deployments manage changes through rollouts but only if the Pod spec changes.</center>

Your ongoing application updates will center on deploying new Pods running an updated version of your container image. You should manage that with an update to your YAML specs, but kubectl provides a quick alternative with the set command. Using this command is an imperative way to update an existing Deployment, and you should view it the same as the scale command—it’s a useful hack to get out of a sticky situation, but it needs to be followed up with an update to the YAML files.

​	**TRY IT NOW	Use kubectl to update the image version for the Deployment. This is a change to the Pod spec, so it will trigger a new rollout.**

```
# update the image for the web app:
kubectl set image deployment/vweb web=kiamol/ch09-vweb:v2

# check the ReplicaSets again:
kubectl get rs -l app=vweb

# check the rollouts:
kubectl rollout history deploy/vweb
```

The kubectl set command changes the spec of an existing object. You can use it to change the image or environment variables for a Pod or the selector for a Service. It’s a shortcut to applying a new YAML spec, but it is implemented in the same way. In this exercise, the change caused a rollout, with a new ReplicaSet created to run the new Pod spec and the old ReplicaSet scaled down to zero. You can see this in figure 9.3.

![图9.3](.\images\Figure9.3.png)
<center>图 9.3 Imperative updates go through the same rollout process, but now your YAML is out of sync.</center>

Kubernetes uses the same concept of rollouts for the other Pod controllers, DaemonSets and StatefulSets. They’re an odd part of the API because they don’t map directly to an object (you don’t create a resource with the kind “rollout”), but they’re an important management tool to work with your releases. You can use rollouts to track release history and to revert back to previous releases.

## 9.2 使用 rollouts 和 rollbacks 更新 Deployments

If you look again at figure 9.3, you’ll see the rollout history is pretty unhelpful. There’s a revision number recorded for each rollout but nothing else. It’s not clear what caused the change or which ReplicaSet relates to which revision. It’s good to include a version number (or a Git commit ID) as a label for the Pods, and then the Deployment adds that label to the ReplicaSet, too, which makes it easier to trace updates.

​	**TRY IT NOW	Apply an update to the Deployment, which uses the same Docker image but changes the version label for the Pod. That’s a change to the Pod spec, so it will create a new rollout.**

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

​	As your Kubernetes maturity increases, you’ll want to have a standard set of labels that you include in all your object specs. Labels and selectors are a core feature, and you’ll use them all the time to find and manage objects. Application name, component name, and version are good labels to start with, but it’s important to distinguish between the labels you include for your convenience and the labels that Kubernetes uses to map object relationships.

​	Listing 9.1 shows the Pod labels and the selector for the Deployment in the previous exercise. The app label is used in the selector, which the Deployment uses to find  its Pods. The Pod also contains a version label for our convenience, but that’s not part of the selector. If it were, then the Deployment would be linked to one version, because you can’t change the selector once a Deployment is created.

![图9.4](.\images\Figure9.4.png)
<center>图 9.4 Kubernetes uses labels for key information, and extra detail is stored in annotations.</center>

**Listing 9.1	vweb-v11.yaml, a Deployment with additional labels in the Pod spec**

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

​	**TRY IT NOW	Make an HTTP call to the web Service to see the response, then start another update and check the response again.**

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

​	Rollouts do help to abstract away the details of the ReplicaSets, but their main use is to manage releases. We’ve seen the rollout history from kubectl, and you can also run commands to pause an ongoing rollout or roll back a deployment to an earlier revision. A simple command will roll back to the previous deployment, but if you want to roll back to a specific version, you need some more JSONPath trickery to find the revision you want. We’ll see that now and use a very handy feature of kubectl that tells you what will happen when you run a command, without actually executing it.

​	**TRY IT NOW	Check the rollout history and try rolling back to v1 of the app.**

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
<center>图 9.5 Kubernetes manages rollouts for you, but it helps if you add labels to see what’s what.</center>

Hands up if you ran that exercise and got confused when you saw the final output shown in figure 9.6 (this is the exciting part of the chapter). My hand is up, and I already knew what was going to happen. This is why you need a consistent release

![图9.6](.\images\Figure9.6.png)
<center>图 9.6 Labels are a key management feature, but they’re set by humans so they’re fallible.</center>

process, preferably one that is fully automated, because as soon as you start mixing approaches, you get confusing results. I rolled back to revision 2, and that should have reverted back to v1 of the app, judging by the labels on the ReplicaSets. But revision 2 was actually from the kubectl set image exercise in section 9.1, so the container image is v2, but the ReplicaSet label is v1.

​	You see that the moving parts of the release process are fairly simple: Deployments create and reuse ReplicaSets, scaling them up and down as required, and changes to ReplicaSets are recorded as rollouts. Kubernetes gives you control of the key factors in the rollout strategy, but before we move on to that, we’re going to look at releases which also involve a configuration change, because that adds another complicating factor.

​	In chapter 4, I talked about different approaches to updating the content of Config-Maps and Secrets, and the choice you make impacts your ability to roll back cleanly. The first approach is to say that configuration is mutable, so a release might include a ConfigMap change, which is an update to an existing ConfigMap object. But if your release is only a configuration change, then you have no record of that as a rollout and no option to roll back.

​	**TRY IT NOW	Remove the existing Deployment so we have a clean history, then deploy a new version that uses a ConfigMap, and see what happens when you update the same ConfigMap.**

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

​	This is the hot reload approach, which works nicely if your apps support it, precisely because a configuration-only change doesn’t require a rollout. The existing Pods and containers keep running, so there’s no risk of service interruption. The cost is the loss of the rollback option, and you’ll have to decide whether that’s more important than a hot reload.

​	Your alternative is to consider all ConfigMaps and Secrets as immutable, so you include some versioning scheme in the object name and never update a config object once it’s created. Instead you create a new config object with a new name and release it along with an update to your Deployment, which references the new config object.

​	**TRY IT NOW	Deploy a new version of the app with an immutable config, so you can compare the release process.**

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
<center>图 9.7 Configuration updates might change app behavior but without recording a rollout.</center>

Figure 9.8 shows my output, where the config update is accompanied by a Deployment update, which preserves the rollout history and enables the rollback.

![图9.8](.\images\Figure9.8.png)
<center>图 9.8 An immutable config preserves rollout history, but it means a rollout for every  configuration change.</center>

Kubernetes doesn’t really care which approach you take, and your choice will partly depend on who owns the configuration in your organization. If the project team also owns deployment and configuration, then you might prefer mutable config objects to simplify the release process and the number of objects to manage. If a separate team owns the configuration, then the immutable approach will be better because they can deploy new config objects ahead of the release. The scale of your apps will affect the decision, too: at a high scale, you may prefer to reduce the number of app deployments and rely on mutable configuration.

​	There’s a cultural impact to this decision, because it frames how application releases are perceived—as everyday events that are no big deal, or as something slightly scary that is to be avoided as much as possible. In the container world, releases should be trivial events that you’re happy to do with minimal ceremony as soon as they’re needed. Testing and tweaking your release strategy will go a long way to giving you that confidence.

## 9.3 为 Deployments 配置滚动更新

Deployments support two update strategies: RollingUpdate is the default and the one we’ve used so far, and the other is Recreate. You know how rolling updates work—by scaling down the old ReplicaSet while scaling up the new ReplicaSet, which provides service continuity and the ability to stagger the update over a longer period. The Recreate strategy gives you neither of those. It still uses ReplicaSets to implement changes, but it scales down the previous set to zero before scaling up the replacement. Listing 9.2 shows the Recreate strategy in a Deployment spec. It’s just one setting, but it has a significant impact.

**Listing 9.2	vweb-recreate-v2.yaml, a Deployment using the recreate update strategy**

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

​	**TRY IT NOW	Deploy the app from listing 9.2, and explore the objects. This is just like a normal Deployment.**

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

![图9.9](.\images\Figure9.9.png)
<center>图 9.9	The Recreate update strategy doesn’t affect behavior until you release an update.</center>

This configuration is dangerous, though, and one you should use only if different versions of your app can’t coexist—something like a database schema update, where you need to be sure that only one version of your app connects to the database. Even in that case, you have better options, but if you have a scenario that definitely needs this approach, then you’d better be sure you test all your updates before you go live. If you deploy an update where the new Pods fail, you won’t know that until your old Pods have all been terminated, and your app will be completely unavailable.

​	**TRY IT NOW	Version 3 of the web app is ready to deploy. It’s broken, as you’ll see when the app goes offline because no Pods are running.**

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

![图9.10](.\images\Figure9.10.png)
<center>图 9.10 The Recreate strategy happily takes down your app if the new Pod spec is broken.</center>

Now that you’ve seen it, you should probably try to forget about the Recreate strategy. In some scenarios, it might seem attractive, but when it does, you should still consider alternative options, even if it means looking again at your architecture. The wholesale takedown of your application is going to cause downtime, and probably more downtime than you plan for.

​	Rolling updates are the default because they guard against downtime, but even then, the default behavior is quite aggressive. For a production release, you’ll want to tune a few settings that set the speed of the release and how it gets monitored. As part of the rolling update spec, you can add options that control how quickly the new ReplicaSet is scaled up and how quickly the old ReplicaSet is scaled down, using the following two values:

- maxUnavailable is the accelerator for scaling down the old ReplicaSet. It defines how many Pods can be unavailable during the update, relative to the desired Pod count. You can think of it as the batch size for terminating Pods in the old ReplicaSet. In a Deployment of 10 Pods, setting this to 30%  means three Pods will be terminated immediately.
- maxSurge is the accelerator for scaling up the new ReplicaSet. It defines how many extra Pods can exist, over the desired replica count, like the batch size for creating Pods in the new ReplicaSet. In a Deployment of 10, setting this to 40% will create four new Pods.

Nice and simple, except both settings are used during a rollout, so you have a seesaw effect. The new ReplicaSet is scaled up until the Pod count is the desired replica count plus the maxSurge value, and then the Deployment waits for old Pods to be removed. The old ReplicaSet is scaled down to the desired count minus the maxUnavailable count, then the Deployment waits for new Pods to reach the ready state. You can’t set both values to zero because that means nothing will change. Figure 9.11 shows how you can combine the settings to prefer a create-then-remove, or a remove-then-create, or a remove-and-create approach to new releases.

​	You can tweak these settings for a faster rollout if you have spare compute power in your cluster. You can also create additional Pods over your scale setting, but that’s riskier if you have a problem with the new release. A slower rollout is more conservative: it uses less compute and gives you more opportunity to discover any issues, but it reduces the overall capacity of your app during the release. Let’s see how these look, first by fixing our broken app with a conservative rollout.

​	**TRY IT NOW	Revert back to the working version 2 image, using maxSurge=1 and maxUnavailable=0 in the RollingUpdate strategy.**

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
<center>图 9.11 Deployment updates in progress, using different rollout options</center>

In this exercise, the new Deployment spec changed the Pod image back to version 2, and it also changed the update strategy to a rolling update. You can see in figure 9.12 that the strategy change is made first, and then the Pod update is made in line with the new strategy, creating one new Pod at a time.

![图9.12](.\images\Figure9.12.png)
<center>图 9.12 Deployment updates will use an existing ReplicaSet if the Pod template matches the new spec.</center>

You’ll need to work fast to see the rollout in progress in the previous exercise, because this simple app starts quickly, and as soon as one new Pod is running, the rollout continues with another new Pod. You can control the pace of the rollout with the following two fields in the Deployment spec:

- minReadySeconds adds a delay where the Deployment waits to make sure new Pods are stable. It specifies the number of seconds the Pod should be up with no containers crashing before it’s considered to be successful. The default is zero, which is why new Pods are created quickly during rollouts.
- progressDeadlineSeconds specifies the amount of time a Deployment update can run before it’s considered as failing to progress. The default is 600 seconds, so if an update is not completed within 10 minutes, it’s flagged as not progressing.

Monitoring how long the release takes sounds useful, but as of Kubernetes 1.19, exceeding the deadline doesn’t actually affect the rollout—it just sets a flag on the Deployment. Kubernetes doesn’t have an automatic rollback feature for failed rollouts, but when that feature does come, it will be triggered by this flag. Waiting and checking a Pod for failed containers is a fairly blunt tool, but it’s better than having no checks at all, and you should consider having minReadySeconds specified in all your Deployments.

​	These safety measures are useful to add to your Deployment, but they don’t really help with our web app because the new Pods always fail. We can make this Deployment safe and keep the app online using a rolling update. The next version 3 update sets both maxUnavailable and maxSurge to 1. Doing so has the same effect as the default values (each 25%), but it’s clearer to use exact values in the spec, and Pod counts are easier to work with than percentages in small deployments.

​	**TRY IT NOW	Deploy the version 3 update again. It will still fail, but by using a RollingUpdate strategy, it doesn’t take the app offline.**

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

​	If you check back on the new Pods in a few minutes, you’ll see they’re in the state CrashLoopBackoff. Kubernetes keeps restarting failed Pods by creating replacement containers, but it adds a pause between each restart so it doesn’t choke the CPU on the node. That pause is the backoff time, and it increases exponentially—10 seconds for the first restart, then 20 seconds, and then 40 seconds, up to a maximum of 5 minutes. These version 3 Pods will never restart successfully, but Kubernetes will keep trying.

​	Deployments are the controllers you use the most, and it’s worth spending time working through the update strategy and timing settings to be sure you understand

![图9.13](.\images\Figure9.13.png)
<center>图 9.13 Failed updates don’t automatically roll back or pause; they just keep trying.</center>

the impact for your apps. DaemonSets and StatefulSets also have rolling update functionality, and because they have different ways of managing their Pods, they have different approaches to rollouts, too.

## 9.4 DaemonSets 和 StatefulSets 中的滚动更新

DaemonSets and StatefulSets have two update strategies available. The default is RollingUpdate, which we’ll explore in this section. The alternative is OnDelete, which is for situations when you need close control over when each Pod is updated. You deploy the update, and the controller watches Pods, but it doesn’t terminate any existing Pods. It waits until they are deleted by another process, and then it replaces them with Pods from the new spec.

​	This isn’t quite as pointless as it sounds, when you think about the use cases for these controllers. You may have a StatefulSet where each Pod needs to have flushed data to disk before it’s removed, and you can have an automated process to do that. You may have a DaemonSet where each Pod needs to be disconnected from a hardware component, so it’s free for the next Pod to use. These are rare cases, but the OnDelete strategy lets you take ownership of when Pods are deleted and still have Kubernetes automatically create replacements.

​	We’ll focus on rolling updates in this section, and for that we’ll deploy a version of the to-do list app, which runs the database in a StatefulSet, the web app in a Deployment, and a reverse proxy for the web app in a DaemonSet.

​	**TRY IT NOW	The to-do app runs across six Pods, so start by clearing the existing apps to make room. Then deploy the app, and test that it works correctly.**

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

![图9.14](.\images\Figure9.14.png)
<center>图 9.14	Running the to-do app with a gratuitous variety of controllers</center>

The first update is for the DaemonSet, where we’ll be rolling out a new version of the Nginx proxy image. DaemonSets run a single Pod on all (or some) of the nodes in the cluster, and with a rolling update, you have no surge option. During the update, nodes will never run two Pods, so this is always a delete-then-remove strategy. You can add the maxUnavailable setting to control how many nodes are updated in parallel, but if you take down multiple Pods, you’ll be running at reduced capacity until the replacements are ready.

​	We’ll update the proxy using a maxUnavailable setting of 1, and a minReadySeconds setting of 90. On a single-node lab cluster, the delay won’t have any effect— there’s only one Pod on one node to replace. On a larger cluster, it would mean replacing one Pod at a time and waiting 90 seconds for the Pod to prove it’s stable before moving on to the next.

​	**TRY IT NOW	Start the rolling update of the DaemonSet. On a single-node cluster, a short outage will occur while the replacement Pod starts.**

```
# deploy the DaemonSet update:
kubectl apply -f todo-list/proxy/update/nginx-rollingUpdate.yaml

# watch the Pod update:
kubectl get po -l app=todo-proxy --watch

# Press ctrl-c when the update completes
```

The watch flag in kubectl is useful for monitoring changes—it keeps looking at an object and prints an update line whenever the state changes. In this exercise you’ll see that the old Pod is terminated before the new one is created, which means the app has downtime while the new Pod starts up. Figure 9.15 shows I had one second of downtime in my release.

![图9.15](.\images\Figure9.15.png)
<center>图 9.15 DaemonSets update by removing the existing Pod before creating a replacement.</center>

A multinode cluster wouldn’t have any downtime because the Service sends traffic only to Pods that are ready, and only one Pod at a time gets updated, so the other Pods are always available. You will have reduced capacity, though, and if you tune a faster rollout with a higher maxUnavailable setting, that means a greater reduction in capacity as more Pods are updated in parallel. That’s the only setting you have for DaemonSets, so it’s a simple choice between manually controlling the update by deleting Pods or having Kubernetes roll out the update by a specified number of Pods in parallel.

​	StatefulSets are more interesting, although they have only one option to configure the rollout. Pods are managed in order by the StatefulSet, which also applies to updates—the rollout proceeds backward from the last Pod in the set down to the first. That’s especially useful for clustered applications where Pod 0 is the primary, because it validates the update on the secondaries first.

​	There is no maxSurge or maxUnavailable setting for StatefulSets. The update is always by one Pod at a time. Your configuration option is to define how many Pods should be updated in total, using the partition setting. This setting defines the cutoff point where the rollout stops, and it’s useful for performing a staged rollout of a stateful app. If you have five replicas in your set and your spec includes partition=3, then only Pod 4 and Pod 3 will be updated; Pods 0, 1, and 2 are left running the previous spec.

​	**TRY IT NOW	Deploy a partitioned update to the database image in the StatefulSet, which stops after Pod 1, so Pod 0 doesn’t get updated.**

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

![图9.16](.\images\Figure9.16.png)
<center>图 9.16 Partitioned updates to StatefulSets let you update secondaries and leave the primary unchanged.</center>

This rollout is complete, even though the Pods in the set are running from different specs. For a data-heavy application in a StatefulSet, you may have a suite of verification jobs that you need to run on each updated Pod before you’re happy to continue the rollout, and a partitioned update lets you do that. You can manually control the pace of the release by running successive updates with decreasing partition values, until you remove the partition altogether in the final update to finish the set.

​	**TRY IT NOW	Deploy the update to the database primary. This spec is the same as the previous exercise but with the partition setting removed.**

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

![图9.17](.\images\Figure9.17.png)
<center>图 9.17 Completing the StatefulSet rollout, with an update that is not partitioned</center>

Rolling updates are the default for Deployments, DaemonSets, and StatefulSets, and they all work in broadly the same way: gradually replacing Pods running the previous application spec with Pods running the new spec. The actual details differ because the controllers work in different ways and have different goals, but they impose the same requirement on your app: it needs to work correctly when multiple versions are live. That’s not always possible, and there are alternative ways to deploy app updates in Kubernetes.

## 9.5 理解发布策略

Take the example of a web application. A rolling update is great because it lets each Pod close gracefully when all its client requests are dealt with, and the rollout can be as fast or as conservative as you like. The practical side of the rollout is simple, but you have to consider the user experience (UX) side, too.

​	Application updates might well change the UX—with a different design, new features, or an updated workflow. Any changes like that will be pretty strange for the user if they see the new version during a rollout, then refresh and find themselves with the old version, because the requests have been served by Pods running different versions of the app.

​	The strategies to deal with that go beyond the RollingUpdate spec in your controllers. You can set cookies in your web app to link a client to a particular UX, and then use a more advanced traffic routing system to ensure users keep seeing the new version. When we cover that in chapter 15, you’ll see it introduces several more moving parts. For cases where that method is too complex or doesn’t solve the problem of dual running multiple versions, you can manage the release yourself with a blue-green deployment.

​	Blue-green deployments are a simple concept: you have both the old and new versions of your app deployed at the same time, but only one version is active. You can flip a switch to choose which version is the active one. In Kubernetes, you can do that by updating the label selector in a Service to send traffic to the Pods in a different Deployment, as shown in figure 9.18.

​	You need to have the capacity in your cluster to run two complete copies of your app. If it’s a web or API component, then the new version should be using minimal

![图9.18](.\images\Figure9.18.png)
<center>图 9.18 You run multiple versions of the app in a blue-green deployment, but only one is live.</center>

memory and CPU because it’s not receiving any traffic. You switch between versions by updating the label selector for the Service, so the update is practically instant because all the Pods are running and ready to receive traffic. You can flip back and forth easily, so you can roll back a problem release without waiting for ReplicaSets to scale up and down.

​	Blue-green deployments are less sophisticated than rolling updates, but they’re simpler because of that. They can be a better fit for organizations moving to Kubernetes who have a history of big-bang deployments, but they’re a compute-intensive approach that requires multiple steps and doesn’t preserve the rollout history of your app. You should look to rolling updates as your preferred deployment strategy, but blue-green deployments are a good stepping-stone to use while you gain confidence.

​	That’s all on rolling updates for now, but we will return to the concepts when we cover topics in production readiness, network ingress, and monitoring. We just need to tidy up the cluster now before going on to the lab.

​	**TRY IT NOW	Remove all the objects created for this chapter.**

```
kubectl delete all -l kiamol=ch09
kubectl delete cm -l kiamol=ch09
kubectl delete pvc -l kiamol=ch09
```

## 9.6 实验室

We learned the theory of blue-green deployments in the previous section, and now in the lab, you’re going to make it happen. Working through this lab will help make it clear how selectors relate Pods to other objects and give you experience working with the alternative to rolling updates.

- The starting point is version 1 of the web app, which you can deploy from the lab/v1 folder.
- You need to create a blue-green deployment for version 2 of the app. The spec will be similar to the version 1 spec but using the :v2 container image.
- When you deploy your update, you should be able to flip between the version 1 and version 2 release just by changing the Service and without any updates to Pods.

This is good practice in copying YAML files and trying to work out which fields you need to change. You can find my solution on GitHub: https://github.com/sixeyed/kiamol/blob/master/ch09/lab/README.md.