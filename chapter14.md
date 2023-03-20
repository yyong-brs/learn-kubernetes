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

## 14.2 Monitoring apps built with Prometheus client libraries

Appendix B in the ebook walks through adding metrics to an app that shows a picture from NASA’s Astronomy Photo of the Day (APOD) service. The components of that app are in Java, Go, and Node.js, and they each use a Prometheus client library to expose run-time and application metrics. This chapter includes Kubernetes manifests for the app that deploy to the test namespace, so all the application Pods will be discovered by Prometheus.

![图14.4](./images/Figure14.4.png)
<center>Figure 14.4 Every instance records its own metrics so you need to collect from every Pod. </center>

TRY IT NOW 
Deploy the APOD app to the test namespace, and confirm that the three components of the app are added as Prometheus targets.

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

> Listing 14.2 prometheus-config.yaml, using annotations to override default values

```
- source_labels: # This is a relabel configuration in the test-pods job.
  - __meta_kubernetes_pod_annotationpresent_prometheus_io_path
  - __meta_kubernetes_pod_annotation_prometheus_io_path
regex: true;(.*) # If the Pod has an annotation named prometheus.io/path . . .

target_label: __metrics_path__ # sets the target path from the annotation.
```

This is way less complicated than it looks. The rule says: if the Pod has an annotation called prometheus.io/path, then use the value of that annotation as the metrics path. Prometheus does it all with labels, so every Pod annotation becomes a label with the name meta_kubernetes_pod_annotation_<annotation-name>, and there’s an accompanying label called meta_kubernetes_pod_annotationpresent_<annotation-name>, which you can use to check if the annotation exists. Any apps that use a custom metrics path need to add the annotation. Listing 14.3 shows that for the APOD API.

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

## 14.3 Monitoring third-party apps with metrics exporters
Most applications record metrics in some way, but older apps won’t collect and expose them in Prometheus format. Exporters are separate applications that understand how the target app does its monitoring and can convert those metrics to Prometheus format. Kubernetes provides the perfect way to run an exporter alongside every instance of an application using a sidecar container. This is the adapter pattern we covered in chapter 7.

Nginx and Postgres both have exporters available that we can run as sidecars to improve the monitoring dashboard for the to-do app. The Nginx exporter reads from a status page on the Nginx server and converts the data to Prometheus format. Remember that all the containers in a Pod share the network namespace, so the exporter container can access the Nginx container at the localhost address. The exporter provides its own HTTP endpoint for metrics on a custom port, so the full Pod spec includes the sidecar container and an annotation to specify the metrics port. Listing 14.5 shows the key parts.

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

## 14.4 Monitoring containers and Kubernetes objects
Prometheus integrates with Kubernetes for service discovery, but it doesn’t collect any metrics from the API. You can get metrics about Kubernetes objects and container activity from two additional components: cAdvisor, a Google open source project, and kube-state-metrics, which is part of the wider Kubernetes organization on GitHub. Both run as containers in the cluster, but they collect data from different sources. cAdvisor collects metrics from the container runtime, so it runs as a DaemonSet with a Pod on each node to report on that node’s containers. kube-state-metrics queries the Kubernetes API so it can run as a Deployment with a single replica on any node.

TRY IT NOW 
Deploy the metric collectors for cAdvisor and kube-state-metrics,and update the Prometheus configuration to include them as scrape targets.

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
