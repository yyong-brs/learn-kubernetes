# 第十四章 使用 Prometheus 监控应用程序和 Kubernetes

Monitoring is the companion to logging: your monitoring system tells you something is wrong, and then you can dig into the logs to find out the details. Like logging, you want to have a centralized system to collect and visualize metrics about all your application components. An established approach for monitoring in Kuber-netes uses another CNCF project: Prometheus, which is a server application that collects and stores metrics. In this chapter, you’ll learn how to deploy a shared monitoring system in Kubernetes with dashboards that show the health of individ-ual applications and the cluster as a whole.

Prometheus runs on many platforms, but it’s particularly well suited to Kuber-netes. You run Prometheus in a Pod that has access to the Kubernetes API server,and then Prometheus queries the API to find all the targets it needs to monitor.

When you deploy new apps, you don’t need to make any setup changes—Prometheus discovers them automatically and starts collecting metrics. Kubernetes apps are par-ticularly well suited to Prometheus, too. You’ll see in this chapter how to make good use of the sidecar pattern, so every app can provide some metrics to Prometheus, even if the application itself isn’t Prometheus-ready.

监控是日志记录的伙伴:监视系统告诉您某些地方出了问题，然后您可以深入日志以找出详细信息。与日志记录一样，您希望有一个集中的系统来收集和可视化关于所有应用程序组件的指标。Kuber-netes中已建立的监控方法使用另一个CNCF项目:Prometheus，这是一个收集和存储指标的服务器应用程序。在本章中，您将学习如何在Kubernetes中部署一个共享监控系统，使用指示板显示单个应用程序和整个集群的健康状况。

Prometheus在许多平台上运行，但它特别适合Kuber-netes。您可以在一个Pod中运行Prometheus，该Pod可以访问Kubernetes API服务器，然后Prometheus查询API以找到它需要监视的所有目标。

当你部署新的应用程序时，你不需要做任何设置更改——prometheus会自动发现它们并开始收集指标。Kubernetes的应用程序也特别适合Prometheus。在本章中，你将看到如何很好地利用sidecar模式，因此每个应用程序都可以为Prometheus提供一些指标，即使应用程序本身还没有准备好。

## 14.1 Prometheus 如何监控 Kubernetes 的工作负载

Metrics in Prometheus are completely generic: each component you want to moni-tor has an HTTP endpoint, which returns all the values that are important to that component. A web server includes metrics for the number of requests it serves, and a Kubernetes node includes metrics for how much memory is available. Pro-metheus doesn’t care what’s in the metrics; it just stores everything the component returns. What’s important to Prometheus is a list of targets it needs to collect from.

Figure 14.1 shows how that works in Kubernetes, using Prometheus’s built-in service discovery.

![图14.1](./images/Figure14.1.png)
<center>图 14.1 Prometheus uses a pull model to collect metrics, automatically finding targets </center>

The focus in this chapter is getting Prometheus working nicely with Kubernetes, to give you a dynamic monitoring system that keeps working as your cluster expands with more nodes running more applications. I won’t go into much detail on how you add monitoring to your applications or what metrics you should recordappendix B in the ebook is the chapter “Adding Observability with Containerized Monitoring” from Learn Docker in a Month of Lunches, which will give you that additional detail.

We’ll start by getting Prometheus up and running. The Prometheus server is a sin-gle component that takes care of service discovery and metrics collection and storage,and it has a basic web UI that you can use to check the status of the system and run simple queries.

TRY IT NOW
Deploy Prometheus in a dedicated monitoring namespace, config-ured to find apps in a test namespace (the test namespace doesn’t exist yet).

Prometheus 中的度量完全是通用的:您想要监视的每个组件都有一个HTTP端点，该端点返回对该组件重要的所有值。web服务器包含它所服务的请求数量的指标，Kubernetes节点包含可用内存数量的指标。Pro-metheus并不关心度量标准中的内容;它只存储组件返回的所有内容。对普罗米修斯来说，重要的是它需要收集的目标列表。

图14.1显示了如何使用Prometheus的内置服务发现在Kubernetes中工作。

![图14.1](./images/Figure14.1.png)
<center>图14.1普罗米修斯使用拉模型收集指标，自动找到目标</center>

本章的重点是让Prometheus与Kubernetes很好地合作，为您提供一个动态监控系统，当您的集群扩展到更多的节点，运行更多的应用程序时，该系统仍能保持工作。我不会详细介绍如何在应用程序中添加监控，或者应该记录哪些指标，电子书的附录B是《Learn Docker in a Month of lunch》中的“添加可观察性与容器化监控”一章，它将为您提供额外的细节。

我们将从启动普罗米修斯开始。Prometheus服务器是一个单独的组件，负责服务发现、指标收集和存储，它有一个基本的web UI，您可以使用它来检查系统的状态并运行简单的查询。

现在试试吧
将Prometheus部署在专用的监视名称空间中，配置为在测试名称空间中查找应用程序(测试名称空间还不存在)。

```
# switch to this chapter’s folder:
cd ch14

# create the Prometheus Deployment and ConfigMap:
kubectl apply -f prometheus/

# wait for Prometheus to start:
kubectl wait --for=condition=ContainersReady pod -l app=prometheus -n
kiamol-ch14-monitoring

# get the URL for the web UI:
kubectl get svc prometheus -o jsonpath='http://{.status.loadBalancer
.ingress[0].*}:9090' -n kiamol-ch14-monitoring
# browse to the UI, and look at the /targets page
```

Prometheus calls metrics collection scraping. When you browse to the Prometheus UI,you’ll see there are no scrape targets, although there is a category called testpods, which lists zero targets. Figure 14.2 shows my output. The test-pods name comes from the Prometheus configuration you deployed in a ConfigMap, which the Pod reads from.

![图14.2](./images/Figure14.2.png)
<center>Figure 14.2 No targets yet, but Prometheus will keep checking the Kubernetes API for new Pods. </center>

Configuring Prometheus to find targets in Kubernetes is fairly straightforward, although the terminology is confusing at first. Prometheus uses jobs to define a related set of targets to scrape, which could be multiple components of an application. The scrape configuration can be as simple as a static list of domain names, which Prometheus polls to grab the metrics, or it can use dynamic service discovery.Listing 14.1 shows Prometheus the beginning of the test-pods job configuration, which uses the Kubernetes API for service discovery.

普罗米修斯称度量收集为刮取。当您浏览到Prometheus UI时，您将看到没有抓取目标，尽管有一个名为testpods的类别，它列出了零目标。图14.2显示了我的输出。test-pods的名称来自您在ConfigMap中部署的Prometheus配置，Pod从中读取该配置。

![图14.2](./images/Figure14.2.png)
<center>图14.2目前还没有目标，但Prometheus将继续检查Kubernetes API以寻找新的pod </center>

配置Prometheus以在Kubernetes中寻找目标是相当简单的，尽管术语一开始令人困惑。Prometheus使用作业来定义一组相关的目标，这些目标可以是应用程序的多个组件。抓取配置可以简单到一个静态域名列表(Prometheus轮询该列表以获取指标)，也可以使用动态服务发现。清单14.1向Prometheus展示了test-pods作业配置的开头，它使用Kubernetes API进行服务发现。

> Listing 14.1 prometheus-config.yaml, scrape configuration with Kubernetes

```
scrape_configs: # This is the YAML inside the ConfigMap.
   - job_name: 'test-pods' # Used for test apps
     kubernetes_sd_configs: # Finds targets from the Kubernetes API
     - role: pod # Searches for Pods
     relabel_configs: # Applies these filtering rules
     - source_labels:
       - __meta_kubernetes_namespace
       action: keep # Includes Pods only where the namespace
       regex: kiamol-ch14-test # is the test namespace for this chapter
```

It’s the relabel_configs section that needs explanation. Prometheus stores metrics with labels, which are key-value pairs that identify the source system and other relevant information. You’ll use labels in queries to select or aggregate metrics, and you can also use them to filter or modify metrics before they are stored in Prometheus. This is relabeling, and conceptually, it’s similar to the data pipeline in Fluent Bit—it’s your chance to discard data you don’t want and reshape the data you do want. Regular expressions rear their unnecessarily complicated heads in Prometheus, too, but it’s rare that you need to make changes. The pipeline you set up in the relabeling phase should be generic enough to work for all your apps. The full pipeline in the configuration file applies the following rules:

- Include Pods only from the namespace kiamol-ch14-test.
- Use the Pod name as the value of the Prometheus instance label.
- Use the app label in the Pod metadata as the value of the Prometheus job label.
- Use optional annotations in the Pod metadata to configure the scrape target.

This approach is convention-driven—as long as your apps are modeled to suit the rules, they’ll automatically be picked up as monitoring targets. Prometheus uses the rules to find Pods that match, and for each target, it collects metrics by making an HTTP GET request to the /metrics path. Prometheus needs to know which network port to use, so the Pod spec needs to explicitly include the container port. That’s a good practice anyway because it helps to document your application’s setup. Let’s deploy a simple app to the test namespace and see what Prometheus does with it.

TRY IT NOW 
Deploy the timecheck application to the test namespace. The spec matches all the Prometheus scrape rules, so the new Pod should be found and added as a scrape target.

需要解释的是relabel_configs部分。Prometheus使用标签存储度量，标签是标识源系统和其他相关信息的键值对。您将在查询中使用标签来选择或聚合指标，还可以在将指标存储到Prometheus之前使用它们来过滤或修改指标。这是重新标签，从概念上讲，它类似于Fluent bit中的数据管道——您有机会丢弃不想要的数据并重新塑造您想要的数据。正则表达式在Prometheus中也出现了不必要的复杂问题，但很少需要进行更改。你在重新标签阶段设置的管道应该足够通用，适用于所有应用程序。配置文件中的全管道应用如下规则:

- 只包含命名空间kiamol-ch14-test中的Pods。
- 使用Pod名称作为Prometheus实例标签的值。
- 使用Pod元数据中的app标签作为Prometheus作业标签的值。
- 在Pod元数据中使用可选注解配置抓取目标。

这种方法是由约定驱动的——只要您的应用程序被建模以适应规则，它们就会自动被选为监视目标。Prometheus使用规则来查找匹配的Pods，对于每个目标，它通过向/metrics路径发出HTTP GET请求来收集指标。Prometheus需要知道使用哪个网络端口，因此Pod规范需要显式地包括容器端口。这是一个很好的实践，因为它有助于记录应用程序的设置。让我们将一个简单的应用程序部署到test名称空间，看看Prometheus用它做了什么。

现在试试吧
将时间检查应用程序部署到测试名称空间。该规格匹配所有的普罗米修斯刮擦规则，所以新的Pod应该被找到并添加为刮擦目标。

```
# create the test namespace and the timecheck Deployment:
kubectl apply -f timecheck/
# wait for the app to start:
kubectl wait --for=condition=ContainersReady pod -l app=timecheck -n
kiamol-ch14-test
# refresh the target list in the Prometheus UI, and confirm the
# timecheck Pod is listed, then browse to the /graph page, select
# timecheck_total from the dropdown list, and click Execute
```

My output is shown in figure 14.3, where I’ve opened two browser windows so you can see what happened when the app was deployed. Prometheus saw the timecheck Pod being created, and it matched all the rules in the relabel stage, so it was added as a target. The Prometheus configuration is set to scrape targets every 30 seconds. The timecheck app has a /metrics endpoint,which returns a count for how many timecheck logs it has written. When I queried that metric in Prometheus, the app had written 22 log entries.

![图14.3](./images/Figure14.3.png)
<center>Figure 14.3 Deploying an app to the test namespace—Prometheus finds it and starts collecting metrics. </center>

You should realize two important things here: the application itself needs to provide the metrics because Prometheus is just a collector, and those metrics represent the activity for one instance of the application. The timecheck app isn’t a web application—it’s just a background process—so there’s no Service directing traffic to it. Prometheus gets the Pod IP address when it queries the Kubernetes API, and it makes the HTTP request directly to the Pod. You can configure Prometheus to query Services, too, but then you’d get a target that is a load balancer across multiple Pods, and you want Prometheus to scrape each Pod independently.

You’ll use the metrics in Prometheus to power dashboards showing the overall health of your apps, and you may aggregate across all the Pods to get the headline values. You need to be able to drill down, too, to see if there are differences between the Pods. That will help you identify if some instances are performing badly, and that will feed back into your health checks.We can scale up the timecheck app to see the importance of collecting at the individual Pod level.

TRY IT NOW 
Add another replica to the timecheck app. It’s a new Pod that matches the Prometheus rules, so it will be discovered and added as another scrape target.

我的输出如图14.3所示，其中我打开了两个浏览器窗口，以便您可以看到部署应用程序时发生了什么。普罗米修斯看到时间检查Pod被创建，它符合重新标记阶段的所有规则，所以它被添加为目标。普罗米修斯配置设置为每30秒刮一次目标。时间检查应用程序有一个/metrics端点，它返回它写了多少时间检查日志的计数。当我在Prometheus中查询该指标时，应用程序已经写入了22个日志条目。

![图14.3](./images/Figure14.3.png)
<center>图14.3将应用程序部署到测试名称空间- prometheus找到它并开始收集指标. </center>

这里您应该认识到两个重要的事情:应用程序本身需要提供度量，因为Prometheus只是一个收集器，而那些度量代表应用程序实例的活动。时间检查应用程序不是一个web应用程序——它只是一个后台进程——所以没有服务将流量导向它。普罗米修斯在查询Kubernetes API时获得Pod的IP地址，并直接向Pod发出HTTP请求。您也可以配置Prometheus来查询Services，但是这样您就会得到一个目标，它是跨多个Pod的负载均衡器，并且您希望Prometheus独立地抓取每个Pod。

你可以使用Prometheus中的指标来增强仪表板，显示应用程序的整体健康状况，你可以汇总所有pod来获得标题值。您还需要能够向下钻取，以查看pod之间是否有差异。这将帮助您确定某些实例是否执行不良，并将反馈到您的运行状况检查中。我们可以放大时间检查应用程序，看看在单个Pod级别收集的重要性。

现在试试吧
添加另一个副本到时间检查应用程序。这是一个新的Pod，符合普罗米修斯规则，所以它将被发现并添加为另一个刮擦目标。

```
# scale the Deployment to add another Pod:
kubectl scale deploy/timecheck --replicas 2 -n kiamol-ch14-test
# wait for the new Pod to spin up:
kubectl wait --for=condition=ContainersReady pod -l app=timecheck -n
kiamol-ch14-test
# back in Prometheus, check the target list, and in the graph page,
# execute queries for timecheck_total and dotnet_total_memory_bytes
```

## 14.2 监视使用 Prometheus 客户端库构建的应用程序

Appendix B in the ebook walks through adding metrics to an app that shows a picture from NASA’s Astronomy Photo of the Day (APOD) service. The components of that app are in Java, Go, and Node.js, and they each use a Prometheus client library to expose run-time and application metrics. This chapter includes Kubernetes manifests for the app that deploy to the test namespace, so all the application Pods will be discovered by Prometheus.

![图14.4](./images/Figure14.4.png)
<center>Figure 14.4 Every instance records its own metrics so you need to collect from every Pod. </center>

TRY IT NOW 
Deploy the APOD app to the test namespace, and confirm that the three components of the app are added as Prometheus targets.

电子书的附录B介绍了如何向一个应用程序添加指标，该应用程序显示了来自NASA“每日天文照片”(APOD)服务的图片。该应用程序的组件在Java、Go和Node.js中，它们都使用Prometheus客户端库来公开运行时和应用程序指标。本章包括部署到test名称空间的应用程序的Kubernetes清单，因此所有应用程序Pods将被Prometheus发现。

![图14.4](./images/Figure14.4.png)
<center>图14.4每个实例都记录自己的度量，因此您需要从每个Pod收集。</center>

现在试试吧
将APOD应用程序部署到测试名称空间，并确认应用程序的三个组件被添加为Prometheus目标。

```
# deploy the app:
kubectl apply -f apod/
# wait for the main component to start:
kubectl wait --for=condition=ContainersReady pod -l app=apod-api -n
kiamol-ch14-test
# get the app URL:
kubectl get svc apod-web -o
jsonpath='http://{.status.loadBalancer.ingress[0].*}:8014' -n
kiamol-ch14-test
# browse to the app, and then refresh the Prometheus targets
```

You can see my output in figure 14.5, with a very pleasant image of something called Lynds Dark Nebula 1251. The application is working as expected, and Prometheushas discovered all of the new Pods. Within 30 seconds of deploying the app, you

![图14.5](./images/Figure14.5.png)
<center>Figure 14.5 The APOD components all have Services, but they are still scraped at the Pod level. </center>

should see that the state of all of the new targets is up, which means Prometheus has successfully scraped them.

I have two additional important things to point out in this exercise. First, the Pod specs all include a container port, which states that the application container is listening on port 80, and that’s how Prometheus finds the target to scrape. The Service for the web UI actually listens on port 8014, but Prometheus goes directly to the Pod port. Second, the API target isn’t using the standard /metrics path, because the Java client library uses a different path. I’ve used an annotation in the Pod spec to state the correct path.

Convention-based discovery is great because it removes a lot of repetitive configuration and the potential for mistakes, but not every app will fit with the conventions.The relabeling pipeline we’re using in Prometheus gives us a nice balance. The default values will work for any apps that meet the convention, but any that don’t can override the defaults with annotations. Listing 14.2 shows how the override is configured to set the path to the metrics.

你可以在图14.5中看到我的输出，其中有一个非常令人愉快的图像，叫做林德斯暗星云1251。应用程序如预期的那样运行，普罗米修斯已经发现了所有新的pod。在部署应用程序的30秒内，你

![图14.5](./images/Figure14.5.png)
<center>图14.5 APOD组件都有服务，但它们仍然在Pod级别被抓取</center>

应该会看到所有新目标的状态都是正常的，这意味着普罗米修斯已经成功地抓取了它们。

在这个练习中，我还有两件重要的事情要指出。首先，Pod规范都包括一个容器端口，它说明应用程序容器正在监听端口80，这就是Prometheus找到要抓取的目标的方法。web UI的Service实际上监听端口8014，但是Prometheus直接访问Pod端口。其次，API目标没有使用标准/度量路径，因为Java客户机库使用不同的路径。我在Pod规范中使用了注释来说明正确的路径。

基于约定的发现很棒，因为它消除了大量重复的配置和潜在的错误，但并不是每个应用都符合约定。我们在《普罗米修斯》中使用的重标签管道为我们提供了一个很好的平衡。默认值适用于任何符合约定的应用程序，但任何不符合约定的应用程序都可以用注释覆盖默认值。清单14.2显示了如何配置覆盖以设置度量的路径。

> Listing 14.2 prometheus-config.yaml, using annotations to override default values

```
- source_labels: # This is a relabel configuration in the test-pods job.
  - __meta_kubernetes_pod_annotationpresent_prometheus_io_path
  - __meta_kubernetes_pod_annotation_prometheus_io_path
regex: true;(.*) # If the Pod has an annotation named prometheus.io/path . . .

target_label: __metrics_path__ # sets the target path from the annotation.
```

This is way less complicated than it looks. The rule says: if the Pod has an annotation called prometheus.io/path, then use the value of that annotation as the metrics path. Prometheus does it all with labels, so every Pod annotation becomes a label with the name meta_kubernetes_pod_annotation_<annotation-name>, and there’s an accompanying label called meta_kubernetes_pod_annotationpresent_<annotation-name>, which you can use to check if the annotation exists. Any apps that use a custom metrics path need to add the annotation. Listing 14.3 shows that for the APOD API.

这远没有看起来那么复杂。规则是这样的:如果豆荚有一个叫普罗米修斯的注释。Io /path，然后使用该注释的值作为度量路径。Prometheus使用标签来完成这一切，因此每个Pod注释都成为一个名称为meta_kubernetes_pod_annotation_<annotation-name>的标签，并且有一个附带的标签名为meta_kubernetes_pod_annotationpresent_<annotation-name>，您可以使用它来检查注释是否存在。任何使用自定义指标路径的应用程序都需要添加注释。清单14.3显示了APOD API。

> Listing 14.3 api.yaml, the path annotation in the API spec

```
template: # This is the pod spec in the Deployment.
   metadata:
     labels:
       app: apod-api # Used as the job label in Prometheus
     annotations:
       prometheus.io/path: "/actuator/prometheus" # Sets the metrics path
```
The complexity is centralized in the Prometheus configuration, and it’s really easy for application manifests to specify overrides. The relabeling rules aren’t so complex when you work with them a little more, and you’re usually following exactly the same pattern. The full Prometheus configuration includes similar rules for apps to override the metrics port and to opt out of scraping altogether.

While you’ve been reading this, Prometheus has been busily scraping the timecheck and APOD apps. Take a look at the metrics on the Graph page of the Prometheus UI to see around 200 metrics being collected. The UI is great for running queries and quickly seeing the results, but you can’t use it to build a dashboard showing all the key metrics for your app in a single screen. For that you can use Grafana, another open source project in the container ecosystem, which comes recommended by the Prometheus team.

TRY IT NOW 
Deploy Grafana with ConfigMaps that set up the connection to Prometheus, and include a dashboard for the APOD app.

复杂性集中在Prometheus配置中，应用程序清单非常容易指定覆盖。当您多使用一些重新标记规则时，它们就不那么复杂了，并且通常遵循完全相同的模式。完整的Prometheus配置包括类似的规则，应用程序可以覆盖指标端口，并选择完全退出抓取。

当你在读这篇文章的时候，普罗米修斯一直在忙着抓取时间支票和APOD应用程序。看看Prometheus UI的Graph页面上的指标，可以看到大约有200个指标正在被收集。UI非常适合运行查询并快速查看结果，但你不能用它来构建一个仪表板，在一个屏幕上显示应用程序的所有关键指标。为此，您可以使用Grafana，它是容器生态系统中的另一个开源项目，由Prometheus团队推荐。

现在试试吧
使用ConfigMaps部署Grafana，它建立了与Prometheus的连接，并包括APOD应用程序的仪表板。

```
# deploy Grafana in the monitoring namespace:
kubectl apply -f grafana/
# wait for it to start up:
kubectl wait --for=condition=ContainersReady pod -l app=grafana -n
kiamol-ch14-monitoring
# get the URL for the dashboard:
kubectl get svc grafana -o
jsonpath='http://{.status.loadBalancer.ingress[0].*}:3000/d/kb5nhJA
Zk' -n kiamol-ch14-monitoring
# browse to the URL; log in with username kiamol and password kiamol
```

The dashboard shown in figure 14.6 is tiny, but it gives you an idea of how you can transform raw metrics into an informative view of system activity. Each visualization in

![图14.6](./images/Figure14.6.png)
<center>Figure14.6 Application dashboards give a quick insight into performance. The graphs are all powered from Prometheus metrics. </center>

the dashboard is powered by a Prometheus query, which Grafana runs in the background. There’s a row for each component, and that includes a mixture of run-time metrics—processor and memory usage—and application metrics—HTTP requests and cache usage.

Dashboards like this will be a joint effort that cuts across the organization. The support team will set the requirements for what they need to see, and the application development and operations teams ensure the app captures the data and the dashboard shows it. Just like the logging system we looked at in chapter 13, this is a solution built from lightweight open source components, so developers can run the same monitoring system on their laptops that runs in production. That helps with performance testing and debugging in development and test.

Moving to centralized monitoring with Prometheus will require development effort,but it can be an incremental process where you start with basic metrics and add to them as teams start to come up with more requirements. I’ve added Prometheus support to the to-do list app for this chapter, and it took about a dozen lines of code. There’s a simple dashboard for the app ready to use in Grafana, so when you deploy the app, you’ll be able to see the starting point for a dashboard that will improve with future releases.

TRY IT NOW 
Run the to-do list app with metrics enabled, and use the app to produce some metrics. There’s already a dashboard in Grafana to visualize the metrics.

图14.6所示的仪表板很小，但它让您了解了如何将原始指标转换为系统活动的信息视图。中的每个可视化

![图14.6](./images/Figure14.6.png)
<center>图14.6应用程序仪表板提供了对性能的快速洞察。这些图表都是由Prometheus metrics提供的</center>

仪表板由Prometheus查询驱动，Grafana在后台运行该查询。每个组件都有一行，其中包括运行时指标(处理器和内存使用)和应用程序指标(http请求和缓存使用)的混合。

像这样的仪表板将是贯穿整个组织的共同努力。支持团队将设置他们需要看到的需求，应用程序开发和运营团队确保应用程序捕获数据并在仪表板上显示它。就像我们在第13章中看到的日志系统一样，这是一个由轻量级开源组件构建的解决方案，因此开发人员可以在他们的笔记本电脑上运行与在生产环境中运行的相同的监控系统。这有助于在开发和测试中进行性能测试和调试。

使用Prometheus转移到集中监视将需要开发工作，但是它可以是一个增量过程，您从基本的度量开始，并随着团队开始提出更多的需求而添加它们。我在本章的待办事项列表应用程序中添加了对普罗米修斯的支持，这大约花了十几行代码。在Grafana中有一个简单的应用程序仪表板，所以当你部署应用程序时，你将能够看到一个仪表板的起点，它将在未来的版本中得到改进。

现在试试吧
运行启用指标的待办事项列表应用程序，并使用该应用程序生成一些指标。在Grafana中已经有一个仪表盘来可视化指标。

```
# deploy the app:
kubectl apply -f todo-list/
# wait for it to start:
kubectl wait --for=condition=ContainersReady pod -l app=todo-web -n
kiamol-ch14-test
# browse to the app, and insert an item
# then run some load in with a script - on Windows:
.\loadgen.ps1
# OR on macOS/Linux:
chmod +x ./loadgen.sh && ./loadgen.sh
# get the URL for the new dashboard:
kubectl get svc grafana -o jsonpath='http://{.status.loadBalancer
.ingress[0].*}:3000/d/Eh0VF3iGz' -n kiamol-ch14-monitoring
# browse to the dashboard
```

There’s not much in that dashboard, but it’s a lot more information than no dashboard at all. It tells you how much CPU and memory the app is using inside the container, the rate at which tasks are being created, and the average response time for HTTP requests. You can see my output in figure 14.7 where I’ve added some tasks and sent some traffic in with the load generation script.

![图14.7](./images/Figure14.7.png)
<center>Figure14.7 A simple dashboard powered by the Prometheus client library and a few lines of code </center>

All of those metrics are coming from the to-do application Pod. There are two other components to the app in this release: a Postgres database for storage and an Nginx proxy. Neither of those components has native support for Prometheus, so they’re excluded from the target list. Otherwise, Prometheus would keep trying to scrape metrics and failing. It’s the job of whoever models the application to know that a component doesn’t expose metrics and to specify that it should be excluded. Listing 14.4 shows that done with a simple annotation.

仪表板上没有太多东西，但它比没有仪表板的信息要多得多。它告诉你应用程序在容器内使用了多少CPU和内存，创建任务的速率，以及HTTP请求的平均响应时间。您可以在图14.7中看到我的输出，其中我添加了一些任务，并使用负载生成脚本发送了一些流量。

![图14.7](./images/Figure14.7.png)
<center>图14.7一个由Prometheus客户端库和几行代码驱动的简单仪表板 </center>

所有这些指标都来自待办事项应用程序Pod。在这个版本中，应用程序还有另外两个组件:一个用于存储的Postgres数据库和一个Nginx代理。这两个组件都没有对Prometheus的本地支持，因此它们被排除在目标列表之外。否则，普罗米修斯将继续尝试着获取度量标准并失败。对应用程序进行建模的人员的工作是了解一个组件不公开指标，并指定应该排除它。清单14.4显示了使用一个简单注释完成的操作。

> Listing 14.4 proxy.yaml, a Pod spec that excludes itself from monitoring

```
template: # This is the Pod spec in the Deployment.
   metadata:
     labels:
       app: todo-proxy
     annotations: # Excludes the target in Prometheus
       prometheus.io/scrape: "false"
```
Components don’t need to have native support for Prometheus and provide their own metrics endpoint to be included in your monitoring system. Prometheus has its own ecosystem—in addition to client libraries that you can use to add metrics to your own applications, a whole set of exporters can extract and publish metrics for third-party applications. We can use exporters to add the missing metrics for the proxy and database components.

组件不需要对Prometheus提供本地支持，并提供自己的度量端点以包含在监视系统中。Prometheus有自己的生态系统——除了可以用来向自己的应用程序添加指标的客户端库之外，一整套导出器可以为第三方应用程序提取和发布指标。我们可以使用导出器为代理和数据库组件添加缺少的指标。

## 14.3 通过 metrics exporters 来监控第三方应用

Most applications record metrics in some way, but older apps won’t collect and expose them in Prometheus format. Exporters are separate applications that understand how the target app does its monitoring and can convert those metrics to Prometheus format. Kubernetes provides the perfect way to run an exporter alongside every instance of an application using a sidecar container. This is the adapter pattern we covered in chapter 7.

Nginx and Postgres both have exporters available that we can run as sidecars to improve the monitoring dashboard for the to-do app. The Nginx exporter reads from a status page on the Nginx server and converts the data to Prometheus format. Remember that all the containers in a Pod share the network namespace, so the exporter container can access the Nginx container at the localhost address. The exporter provides its own HTTP endpoint for metrics on a custom port, so the full Pod spec includes the sidecar container and an annotation to specify the metrics port. Listing 14.5 shows the key parts.

大多数应用程序都以某种方式记录指标，但较老的应用程序不会以Prometheus格式收集和公开它们。出口商是独立的应用程序，了解目标应用程序如何进行监视，并可以将这些指标转换为Prometheus格式。Kubernetes提供了完美的方式来运行一个出口国与应用程序的每个实例使用双轮马车容器。这就是我们在第7章中介绍的适配器模式。

Nginx和Postgres都有可用的导出器，我们可以作为sidecars来运行，以改善待办应用程序的监控仪表板。Nginx导出器从Nginx服务器上的状态页面读取数据，并将数据转换为Prometheus格式。记住，Pod中的所有容器都共享网络命名空间，因此导出容器可以在本地主机地址访问Nginx容器。导出器为自定义端口上的指标提供了自己的HTTP端点，因此完整的Pod规范包括sidecar容器和指定指标端口的注释。清单14.5显示了关键部分。

> Listing 14.5 proxy-with-exporter.yaml, adding a metrics exporter container

```
template: # Pod spec in the Deployment
   metadata:
      labels:
         app: todo-proxy
      annotations: # The exclusion annotation is gone.
         prometheus.io/port: "9113" # Specifies the metrics port
   spec:
      containers:
         - name: nginx
           # ... nginx spec is unchanged
         - name: exporter # The exporter is a sidecar.
           image: nginx/nginx-prometheus-exporter:0.8.0
           ports:
             - name: metrics
               containerPort: 9113 # Specifies the metrics port
           args: # and loads metrics from Nginx
             - -nginx.scrape-uri=http://localhost/stub_status
```

The scrape exclusion has been removed, so when you deploy this update, Prometheus will scrape the Nginx Pod on port 9113, where the exporter is listening. All the Nginx metrics will bestored by Prometheus, and the Grafana dashboard can be updated to add a row for the proxy. We’re not going to get into the Prometheus query language (PromQL) or building Grafana dashboards in this chapter—dashboards can be imported from JSON files, and there’s an updated dashboard ready to be deployed.

TRY IT NOW 
Update the proxy Deployment to add the exporter sidecar, and load an updated dashboard into the Grafana ConfigMap.

排除刮擦已经被移除，所以当你部署这个更新，普罗米修斯将刮擦端口9113上的Nginx Pod，在那里出口商正在监听。所有的Nginx指标将由Prometheus存储，Grafana仪表板可以更新为代理添加一行。在本章中，我们不打算讨论Prometheus查询语言(PromQL)或构建Grafana仪表板——仪表板可以从JSON文件导入，并且有一个更新的仪表板可以部署。

现在试试吧
更新代理部署以添加导出器侧车，并将更新后的仪表板加载到Grafana ConfigMap中。

```
# add the proxy sidecar:
kubectl apply -f todo-list/update/proxy-with-exporter.yaml
# wait for it to spin up:
kubectl wait --for=condition=ContainersReady pod -l app=todo-proxy -n
kiamol-ch14-test
# print the logs of the exporter:
kubectl logs -l app=todo-proxy -n kiamol-ch14-test -c exporter
# update the app dashboard:
kubectl apply -f grafana/update/grafana-dashboard-todo-list-v2.yaml
# restart Grafana to load the new dashboard:
kubectl rollout restart deploy grafana -n kiamol-ch14-monitoring
# refresh the dashboard, and log in with kiamol/kiamol again
```

The Nginx exporter doesn’t provide a huge amount of information, but the basic details are there. You can see in figure 14.8 that we get the number of HTTP requests and a lower-level breakdown of how Nginx handles connection requests. Even with this simple dashboard, you can see a correlation between the traffic Nginx is handling and the traffic the web app is handling, which suggests the proxy isn’t caching responses and is calling the web app for every request.

It would be nice to get a bit more information from Nginx—like the breakdown of HTTP status codes in the response—but exporters can relay only the information available from the source system, which isn’t much for Nginx. Other exporters provide far more detail, but you need to focus your dashboard so it shows key indicators. More than a dozen or so visualizations and the dashboard becomes overwhelming, and, if it doesn’t convey useful information at a glance, then it’s not doing a very good job.

There’s one more component to add to the to-do list dashboard: the Postgres database. Postgres stores all sorts of useful information in tables and functions inside the database, and the exporter runs queries to power its metrics endpoint. The setup for the Postgres exporter follows the same pattern we’ve seen in Nginx. In this case, the sidecar is configured to access Postgres on localhost, using the same Kubernetes Secret that the Postgres container uses for the admin password. We’ll make a final update to the application dashboard to show the key database metrics from the exporter.

![图14.8](./images/Figure14.8.png)
<center>Figure14.8 Collecting proxy metrics with an exporter adds another level of detail to the dashboard. </center>

TRY IT NOW 
Update the database Deployment spec, adding the Postgres exporter as a sidecar container. Then update the to-do list dashboard with a new row to show database performance.

Nginx导出器没有提供大量的信息，但基本的细节都在那里。你可以在图14.8中看到，我们得到了HTTP请求的数量，以及Nginx如何处理连接请求的底层分解。即使使用这个简单的仪表板，你也可以看到Nginx正在处理的流量和web应用程序正在处理的流量之间的相关性，这表明代理没有缓存响应，而是对每个请求都调用web应用程序。

如果能从Nginx得到更多的信息就好了——比如响应中HTTP状态码的分解——但是导出者只能从源系统中转可用的信息，这对Nginx来说并不多。其他导出器提供更多细节，但您需要集中您的仪表板，以便显示关键指标。超过12个左右的可视化和仪表板变得势不可挡，而且，如果它不能一眼传达有用的信息，那么它就没有做得很好。

还有一个组件要添加到待办事项列表仪表板中:Postgres数据库。Postgres将各种有用的信息存储在数据库中的表和函数中，导出器运行查询来支持其metrics端点。Postgres导出器的设置遵循我们在Nginx中看到的相同模式。在这种情况下，sidecar被配置为访问本地主机上的Postgres，使用与Postgres容器用于admin密码相同的Kubernetes Secret。我们将对应用程序仪表板进行最后的更新，以显示来自导出器的关键数据库指标。

![图14.8](./images/Figure14.8.png)
<center>图14.8使用导出器收集代理指标为仪表板添加了另一层细节. </center>

现在试试吧
更新数据库部署规范，添加Postgres导出器作为侧车容器。然后用新行更新待办事项列表仪表板以显示数据库性能。

```
# add the exporter sidecar to Postgres:
kubectl apply -f todo-list/update/db-with-exporter.yaml
# wait for the new Pod to start:
kubectl wait --for=condition=ContainersReady pod -l app=todo-db -n
kiamol-ch14-test
# print the logs from the exporter:
kubectl logs -l app=todo-db -n kiamol-ch14-test -c exporter
# update the dashboard and restart Grafana:
kubectl apply -f grafana/update/grafana-dashboard-todo-list-v3.yaml
kubectl rollout restart deploy grafana -n kiamol-ch14-monitoring
```
I’ve zoomed out and scrolled down in figure 14.9 so you can see the new visualizations, but the whole dashboard is a joy to behold in full-screen mode. A single page shows you how much traffic is coming to the proxy, how hard the app is working and what users are actually doing, and what’s happening inside the database. You can get the same level of detail in your own apps with client libraries and exporters, and you’re looking at just a few days’ effort.

![图14.9](./images/Figure14.9.png)
<center>Figure14.9 The database exporter records metrics about data activity, which add detail to the dashboard. </center>

Exporters are there to add metrics to apps that don’t have Prometheus support. If your goal is to move a set of existing applications onto Kubernetes, then you may not have the luxury of a development team to add custom metrics. For those apps, you can use the Prometheus blackbox exporter, taking to the extreme the approach that some monitoring is better than none.

The blackbox exporter can run in a sidecar and make TCP or HTTP requests to your application container as well as provide a basic metrics endpoint to say whether the application is up. This approach is similar to adding container probes to your Pod spec, except that the blackbox exporter is for information only. You can run a dashboard to show the status of an app if it isn’t a good fit for Kubernetes’s self-healing mechanisms, like the random-number API we’ve used in this book.

TRY IT NOW 
Deploy the random-number API with a blackbox exporter and the simplest possible Grafana dashboard. You can break the API by using it repeatedly and then reset it so it works again, and the dashboard tracks the status.

在图14.9中，我缩小并向下滚动，这样您就可以看到新的可视化效果，但是在全屏模式下，整个仪表板都是赏心悦目的。一个页面显示了代理的流量，应用程序的工作强度，用户实际在做什么，以及数据库内部发生了什么。您可以在自己的应用程序中使用客户端库和导出器获得相同级别的细节，而这只需要几天的努力。

![图14.9](./images/Figure14.9.png)
<center> Figure14.9 数据库导出器记录有关数据活动的度量，这将向仪表板添加详细信息. </center>

出口商在那里为没有普罗米修斯支持的应用程序添加指标。如果您的目标是将一组现有的应用程序转移到Kubernetes上，那么您可能没有一个奢侈的开发团队来添加自定义指标。对于这些应用程序，您可以使用Prometheus黑盒导出器，这是一种极端的方法，即有监控总比没有好。

黑盒导出器可以在sidecar中运行，向应用程序容器发出TCP或HTTP请求，并提供一个基本的度量端点来说明应用程序是否启动。这种方法类似于在Pod规范中添加容器探测，除了黑盒导出器仅供参考。如果应用程序不适合Kubernetes的自我修复机制，你可以运行一个仪表板来显示应用程序的状态，比如我们在本书中使用的随机数API。

现在试试吧
使用黑盒导出器和最简单的Grafana仪表板部署随机数API。您可以通过重复使用API来破坏它，然后重置它，使其重新工作，仪表板跟踪状态。

```
# deploy the API to the test namespace:
kubectl apply -f numbers/
# add the new dashboard to Grafana:
kubectl apply -f grafana/update/numbers-api/
# get the URL for the API:
kubectl get svc numbers-api -o jsonpath='#app - http://{.status
.loadBalancer.ingress[0].*}:8016/rng' -n kiamol-ch14-test
# use the API by visiting the /rng URL
# it will break after three calls;
# then visit /reset to fix it
# get the dashboard URL, and load it in Grafana:
kubectl get svc grafana -o jsonpath='# dashboard - http://{.status
.loadBalancer.ingress[0].*}:3000/d/Tb6isdMMk' -n kiamol-ch14-
monitoring
```

The random-number API doesn’t have Prometheus support, but running the black- box exporter as a sidecar container gives basic insight into the application status. Figure 14.10 shows a dashboard that is mostly empty, but the two visualizations show whether the app is healthy and the historical trend of the status as the app flips between unhealthy and being reset.

The Pod spec for the random-number API follows a similar pattern to Nginx and Postgres in the to-do app: the blackbox exporter is configured as an additional container and specifies the port where metrics are exposed. The Pod annotations customize the path to the metrics URL, so when Prometheus scrapes metrics from the sidecar, it calls the blackbox exporter, which checks that the API is responding to HTTP requests.

Now we have dashboards for three different apps that have different levels of detail, because the application components aren’t consistent with the data they collect. But all the components have something in common: they’re all running in containers

![图14.10](./images/Figure14.10.png)
<center>Figure14.10 Even a simple dashboard is useful. This shows the current and historical status of the API. </center>

随机数API不支持Prometheus，但是运行黑盒导出器作为sidecar容器可以基本了解应用程序状态。图14.10显示了一个大部分为空的仪表板，但这两个可视化显示了应用程序是否健康，以及应用程序在不健康和被重置之间切换时状态的历史趋势。

随机数API的Pod规范遵循与待办应用中的Nginx和Postgres类似的模式:黑盒导出器被配置为一个额外的容器，并指定暴露指标的端口。Pod注释自定义度量URL的路径，因此当Prometheus从sidecar中抓取度量时，它调用黑盒导出器，该导出器检查API是否响应HTTP请求。

现在我们有三个不同的应用程序的仪表板，它们有不同的细节级别，因为应用程序组件与它们收集的数据不一致。但是所有的组件都有一个共同点:它们都在容器中运行

![图14.10](./images/Figure14.10.png)
<center>图14.10即使是一个简单的仪表盘也是有用的。这显示了API的当前和历史状态。 </center>

## 14.4 监控容器以及 kubernetes 对象

Prometheus integrates with Kubernetes for service discovery, but it doesn’t collect any metrics from the API. You can get metrics about Kubernetes objects and container activity from two additional components: cAdvisor, a Google open source project, and kube-state-metrics, which is part of the wider Kubernetes organization on GitHub. Both run as containers in the cluster, but they collect data from different sources. cAdvisor collects metrics from the container runtime, so it runs as a DaemonSet with a Pod on each node to report on that node’s containers. kube-state-metrics queries the Kubernetes API so it can run as a Deployment with a single replica on any node.

TRY IT NOW 
Deploy the metric collectors for cAdvisor and kube-state-metrics,and update the Prometheus configuration to include them as scrape targets.

Prometheus与Kubernetes集成用于服务发现，但它不从API收集任何指标。你可以从另外两个组件获得关于Kubernetes对象和容器活动的指标:cAdvisor(谷歌开源项目)和kube-state-metrics (Kubernetes组织的一部分)。两者都作为集群中的容器运行，但它们从不同的来源收集数据。cAdvisor从容器运行时收集度量，因此它作为一个DaemonSet运行，每个节点上都有一个Pod，以报告该节点的容器。kube-state-metrics查询Kubernetes API，因此它可以作为部署运行，在任何节点上都有一个副本。

现在试试吧
为cAdvisor和kube-state-metrics部署度量收集器，并更新Prometheus配置以将它们包括为抓取目标。

```
# deploy cAdvisor and kube-state-metrics:
kubectl apply -f kube/
# wait for cAdvisor to start:
kubectl wait --for=condition=ContainersReady pod -l app=cadvisor -n
kube-system
# update the Prometheus config:
kubectl apply -f prometheus/update/prometheus-config-kube.yaml
# wait for the ConfigMap to update in the Pod:
sleep 30
# use an HTTP POST to reload the Prometheus configuration:
curl -X POST $(kubectl get svc prometheus -o
jsonpath='http://{.status.loadBalancer.ingress[0].*}:9090/-/reload'
-n kiamol-ch14-monitoring)
# browse to the Prometheus UI—in the Graph page you’ll see
# metrics listed covering containers and Kubernetes objects
```

In this exercise, you’ll see that Prometheus is collecting thousands of new metrics. The raw data includes the compute resources used by every container and the status of every Pod. My output is shown in figure 14.11. When you run this exercise, you can check the Targets page in the Prometheus UI to confirm that the new targets are being scraped. Prometheus doesn’t automatically reload configuration, so in the exercise, there’s a delay to give Kubernetes time to propagate the ConfigMap update, and the curl command forces a configuration reload in Prometheus.

The updated Prometheus configuration you just deployed includes two new job definitions, shown in listing 14.6. kube-state-metrics is specified as a static target using the full DNS name of the Service. A single Pod collects all of the metrics so there’s no loadbalancing issue here. cAdvisor uses Kubernetes service discovery to find every Pod in the DaemonSet, which would present one target for each node in a multinode cluster.

在本练习中，您将看到Prometheus正在收集数以千计的新度量。原始数据包括每个容器使用的计算资源和每个Pod的状态。我的输出如图14.11所示。当您运行这个练习时，您可以检查Prometheus UI中的Targets页面，以确认新的目标正在被抓取。Prometheus不会自动重新加载配置，因此在练习中，会有一个延迟，让Kubernetes有时间传播ConfigMap更新，并且curl命令强制Prometheus重新加载配置。

刚刚部署的更新后的Prometheus配置包括两个新的作业定义，如清单14.6所示。kube-state-metrics使用服务的完整DNS名称指定为静态目标。一个Pod收集所有的指标，所以这里不存在负载平衡问题。cAdvisor使用Kubernetes服务发现来查找DaemonSet中的每个Pod，这将为多节点集群中的每个节点提供一个目标。

> Listing 14.6 prometheus-config-kube.yaml, new scrape targets in Prometheus

```
- job_name: 'kube-state-metrics' # Kubernetes metrics use a
static_configs: # static configuration with DNS.
- targets:
- kube-state-metrics.kube-system.svc.cluster.local:8080
- kube-state-metrics.kube-system.svc.cluster.local:8081
- job_name: 'cadvisor' # Container metrics use
kubernetes_sd_configs: # Kubernetes service discovery
- role: pod # to find all the DaemonSet
relabel_configs: # Pods, by namespace and label.
- source_labels:
- __meta_kubernetes_namespace
- __meta_kubernetes_pod_labelpresent_app
- __meta_kubernetes_pod_label_app
action: keep
regex: kube-system;true;cadvisor
```
![图14.11](./images/Figure14.11.png)
<center>Figure14.11 New metrics show activity at the cluster and container levels. </center>

Now we have the opposite problem from the random-number dashboard: there’s far too much information in the new metrics, so the platform dashboard will need to be highly selective if it’s going to be useful. I have a sample dashboard prepared that is a good starter. It includes current resource usage and all available resource quantities for the cluster, together with some high-level breakdowns by namespace and warning indicators for the health of the nodes.

TRY IT NOW 
Deploy a dashboard for key cluster metrics and with an update to Grafana so it loads the new dashboard.

![图14.11](./images/Figure14.11.png)
<center>图14.11新指标显示集群和容器级别的活动. </center>

现在我们遇到了与随机数仪表板相反的问题:新指标中有太多信息，所以平台仪表板需要高度选择性，如果它想要有用的话。我准备了一个示例仪表板，这是一个很好的开始。它包括集群的当前资源使用情况和所有可用资源数量，以及按名称空间划分的一些高级分解和节点运行状况的警告指示器。

现在试试吧
为关键集群指标部署一个仪表板，并对Grafana进行更新，以便它加载新的仪表板。

```
# create the dashboard ConfigMap and update Grafana:
kubectl apply -f grafana/update/kube/
# wait for Grafana to load:
kubectl wait --for=condition=ContainersReady pod -l app=grafana -n
kiamol-ch14-monitoring
# get the URL for the new dashboard:
kubectl get svc grafana -o
jsonpath='http://{.status.loadBalancer.ingress[0].*}:3000/d/oWe9aYx
mk' -n kiamol-ch14-monitoring
# browse to the dashboard
```

This is another dashboard that is meant for the big screen, so the screenshot in figure 14.12 doesn’t do it justice. When you run the exercise, you can examine it more closely. The top row shows memory usage, the middle row displays CPU usage, and the bottom row shows the status of Pod containers.

![图14.12](./images/Figure14.12.png)
<center>Figure14.12 Another tiny screenshot—run the exercise in your own cluster to see it full size. </center>

A platform dashboard like this is pretty low level—it’s really just showing you if your cluster is near its saturation point. The queries that power this dashboard will be more useful as alerts, warning you if resource usage is getting out of hand. Kubernetes has pressure indicators that are useful there. The memory pressure and process pressure values are shown in the dashboard, as well as a disk pressure indicator. Those values are significant because if a node comes under compute pressure, it can terminate Pod containers. Those would be good metrics to alert on because if you reach that stage, you probably need to page someone to come and nurse the cluster back to health.

Platform metrics have another use: adding detail to application dashboards where the app itself doesn’t provide detailed enough metrics. The platform dashboard shows compute resource usage aggregated across the whole cluster, but cAdvisor collects it at the container level. It’s the same with kube-state-metrics, where you can filter metrics for a specific workload to add platform information to the application dashboard. We’ll make a final dashboard update in this chapter, adding details from the platform to the random-number app.

TRY IT NOW 
Update the dashboard for the random-number API to add metrics from the platform. This is just a Grafana update; there are no changes to the app itself or to Prometheus.

这是另一个用于大屏幕的仪表板，因此图14.12中的屏幕截图并不能准确地显示它。当您运行这个练习时，您可以更仔细地检查它。上面一行显示内存使用情况，中间一行显示CPU使用情况，下面一行显示Pod容器的状态。

![图14.12](./images/Figure14.12.png)
<center>图14.12另一个小截图—在您自己的集群中运行练习以查看完整的大小. </center>

像这样的平台指示板级别相当低——它实际上只是显示您的集群是否接近饱和点。支持这个仪表板的查询将更有用，可以作为警报，在资源使用失控时发出警告。Kubernetes的压力指示器在那里很有用。内存压力和进程压力值显示在仪表板中，还有一个磁盘压力指示器。这些值很重要，因为如果节点受到计算压力，它可以终止Pod容器。这些都是值得注意的良好指标，因为如果您达到了这个阶段，您可能需要呼叫某人来看护集群恢复正常。

平台指标还有另一个用途:在应用程序本身无法提供足够详细指标的情况下，为应用程序仪表板添加细节。平台仪表板显示整个集群中聚合的计算资源使用情况，但cAdvisor在容器级别上收集它。kube-state-metrics也是如此，您可以过滤特定工作负载的指标，以便向应用程序仪表板添加平台信息。我们将在本章中进行最后一次仪表板更新，将平台的详细信息添加到随机数应用程序中。

现在试试吧
为随机数API更新仪表板，以添加来自平台的指标。这只是一个Grafana更新;应用程序本身和普罗米修斯都没有变化。

```
# update the dashboard:
kubectl apply -f grafana/update/grafana-dashboard-numbers-api-v2.yaml
# restart Grafana so it reloads the dashboard:
kubectl rollout restart deploy grafana -n kiamol-ch14-monitoring
# wait for the new Pod to start:
kubectl wait --for=condition=ContainersReady pod -l app=grafana -n
kiamol-ch14-monitoring
# browse back to the random-number API dashboard
```

As shown in figure 14.13, the dashboard is still basic, but at least we now have some detail that could help correlate any issues. If the HTTP status code shows as 503, we can quickly see if the CPU is spiking, too. If the Pod labels contain an application version (which they should), we could identify which release of the app was experiencing the problem.

There’s a lot more to monitoring that I won’t cover here, but now you have a solid grounding in how Kubernetes and Prometheus work together. The main pieces you’re missing are collecting metrics at the server level and configuring alerts. Server metrics supply data like disk and network usage. You collect them by running exporters directly on the nodes (using the Node Exporter for Linux servers and the Windows Exporter for Windows servers), and you use service discovery to add the nodes as scrape targets. Prometheus has a sophisticated alerting system that uses PromQL queries to define alerting rules. You configure alerts so that when rules are triggered, Prometheus will send emails, create Slack messages, or send a notification through PagerDuty.

![图14.13](./images/Figure14.13.png)
<center>Figure14.13 Augmenting basic health stats with container and Pod metrics adds correlation. </center>

We’ll wrap up the chapter by looking at the full architecture of Prometheus in Kubernetes and digging into which pieces need custom work and where the effort needs to go.

如图14.13所示，仪表板仍然是基本的，但至少我们现在有了一些细节，可以帮助关联任何问题。如果HTTP状态代码显示为503，我们可以快速查看CPU是否也处于峰值状态。如果Pod标签包含一个应用程序版本(这是应该的)，我们可以确定哪个版本的应用程序遇到了问题。

关于监控，还有很多我不会在这里介绍的内容，但是现在您已经对Kubernetes和Prometheus如何协同工作有了一个坚实的基础。您缺少的主要部分是在服务器级收集指标和配置警报。服务器指标提供磁盘和网络使用情况等数据。您可以通过直接在节点上运行导出器来收集它们(对于Linux服务器使用Node export，对于Windows服务器使用Windows export)，并使用服务发现将节点添加为抓取目标。Prometheus有一个复杂的警报系统，它使用PromQL查询定义警报规则。您可以配置警报，以便当规则被触发时，Prometheus将发送电子邮件、创建Slack消息或通过PagerDuty发送通知。

![图14.13](./images/Figure14.13.png)
<center>图14.13使用容器和Pod指标增强基本运行状况统计可以增加相关性. </center>

我们将通过查看《Kubernetes》中普罗米修斯的完整架构来结束本章，并深入研究哪些部分需要定制工作以及需要在哪里努力。

## 14.5 Understanding the investment you make in monitoring
When you step outside of core Kubernetes and into the ecosystem, you need to under- stand whether the project you take a dependency on will still exist in five years, or one year, or by the time the chapter you’re writing makes it to the printing press. I’ve been careful in this book to include only those ecosystem components that are open source, are heavily used, and have an established history and governance model. The monitoring architecture in figure 14.14 uses components that all meet those criteria.

I make that point because the move to Prometheus will involve development work. You need to record interesting metrics for your applications to make your dashboards truly useful. You should feel confident about making that investment because Prometheus is the most popular tool for monitoring containerized applications, and the project was the second to graduate in the CNCF—after Kubernetes itself. There’s also work underway to take the Prometheus metric format into an open standard (called

![图14.14](./images/Figure14.14.png)
<center>Figure14.14 Monitoring doesn’t come for free—it needs development and dependencies on opensource projects. </center>

OpenMetrics), so other tools will be able to read application metrics exposed in the Prometheus format.

What you include in those metrics will depend on the nature of your applications, but a good general approach is to follow the guidelines from Google’s Site Reliability Engineering practice. It’s usually pretty simple to add the four golden signals to your app metrics: latency, traffic, errors, and saturation. (Appendix B in the ebook walks through how those look in Prometheus.) But the real value comes when you think about application performance from the user experience perspective. A graph that shows heavy disk usage in your database doesn’t tell you much, but if you can see that a high percentage of users don’t complete a purchase because your website’s check- out page takes too long to load, that’s worth knowing.

That’s all for monitoring now, so we can clear down the cluster to get ready for the lab.

TRY IT NOW 
Delete the namespaces for this chapter, and the objects created in the system namespace.

```
kubectl delete ns -l kiamol=ch14
kubectl delete all -n kube-system -l kiamol=ch14
```

## 14.6 实验室
Another investigative lab for this chapter. In the lab folder, there’s a set of manifests for a slightly simpler deployment of Prometheus and a basic deployment of Elasticsearch. The goal is to run Elasticsearch with metrics flowing into Prometheus. Here are the details:

- Elasticsearch doesn’t provide its own metrics, so you’ll need to find a component that does that for you.
- The Prometheus configuration will tell you which namespace you need to use for Elasticsearch and the annotation you need for the metrics path.
- You should include a version label in your Elasticsearch Pod spec, so Prometheus will pick that up and add it to the metric labels.
You’ll need to hunt around the documentation for Prometheus to get started, and that should show you the way. My solution is on GitHub for you to check in the usual place: <https://github.com/sixeyed/kiamol/blob/master/ch14/lab/README.md>.
