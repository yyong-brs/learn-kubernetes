# 第十章 通过 Helm 打包并管理应用

尽管 Kubernetes 规模庞大，但它本身并不能解决所有问题;一个庞大的生态系统填补了这些空白。其中一个差距就是应用的包装和分发，而 Helm 就是解决方案。您可以使用 Helm 将一组 Kubernetes YAML 文件分组到一个工件中，并在公共或私有存储库中共享该工件。任何可以访问存储库的人都可以通过一个 Helm 命令安装应用程序。该命令可能部署一整套相关的 Kubernetes 资源，包括 ConfigMaps、Deployments 和 Services，您可以自定义配置作为安装的一部分。

人们使用 Helm 的方式各不相同。有些团队只使用 Helm 来安装和管理来自公共存储库的第三方应用程序。其他团队将 Helm 用于他们自己的应用程序，打包并发布到私有存储库。在本章中，您将学习如何做到这两点，并且您将带着自己的想法离开，了解 Helm 如何适合您的组织。你不需要学习 Helm 来有效地使用 Kubernetes，但它被广泛使用，所以你应该熟悉它。该项目由云原生计算基金会(CNCF)管理，与管理 kubernetes 的基金会相同，这是成熟度和寿命的可靠指标。

## 10.1	Helm 给 Kubernetes 带来了什么

Kubernetes 应用程序在设计时是在 YAML 文件的扩展中建模的，在运行时使用一组标签进行管理。Kubernetes 中没有原生的“应用程序”概念，这显然是将一组相关资源组合在一起的，这是 Helm 解决的问题之一。它是一个命令行工具，用于与存储库服务器交互，以查找和下载应用程序包，并与 Kubernetes 集群一起安装和管理应用程序。

Helm 是另一个抽象层，这次是在应用程序级别。当您安装带有 Helm 的应用程序时，它会在 Kubernetes 集群中创建一组资源——它们是标准的 Kubernetes 资源。Helm 打包格式扩展了 Kubernetes YAML文件，因此 Helm 包实际上只是一组 Kubernetes 清单和一些元数据。我们将首先使用 Helm 部署前面章节中的一个示例应用程序，但首先我们需要安装 Helm。

​<b>现在就试试</b>	helm 是一个跨平台的工具，可以在Windows、macOS和Linux上运行。您可以在这里找到最新的安装说明:https://helm.sh/docs/intro/install。本练习假设您已经安装了 Homebrew 或 Chocolatey 这样的包管理器。如果没有，你需要参考Helm网站的完整安装说明。

```
# 在 Windows, 使用 Chocolatey:
choco install -y kubernetes-helm

# 在 Mac, 使用 Homebrew:
brew install helm

# 在 Linux, 使用 Helm install script:
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-
  helm-3 | bash
  
# 检查成功安装:
helm version
```

本练习中的安装步骤可能在您的系统上不起作用，在这种情况下，您需要在这里停下来，转向 Helm 安装文档。在安装 Helm 并看到 version 命令成功输出(如图10.1所示)之前，我们不能继续深入。

![图10.1](.\images\Figure10.1.png)
<center>图 10.1 有很多安装 Helm 的选项;使用包管理器是最简单的</center>

Helm 是一个客户端工具。以前版本的 Helm 需要在 Kubernetes 集群中部署服务器组件，但在 Helm 3 的主要更新中发生了变化。Helm CLI 使用与 kubectl 连接到 Kubernetes 集群相同的连接信息，因此安装应用程序不需要任何额外配置。然而，您需要配置一个包存储库。Helm 存储库类似于 Docker Hub 这样的容器镜像仓库，但服务器发布所有可用包的索引;Helm 缓存存储库索引的本地副本，您可以使用它来搜索包。

​	<b>现在就试试</b>	添加Helm存储库，同步它，并搜索一个应用程序。

```
# 添加一个仓库, 使用本地名称代替远程服务:
helm repo add kiamol https://kiamol.net

# 更新本地仓库缓存:
helm repo update

# 在仓库缓存中搜索应用:
helm search repo vweb --versions
```

Kiamol 存储库是一个公共服务器，在本练习中您可以看到这个名为 vweb 的包有两个版本。我的输出如图10.2所示。

![图10.2](.\images\Figure10.2.png)
<center>图 10.2 同步Kiamol Helm存储库的本地副本并搜索包</center>

你对 helm 有了一些了解，但现在是时候介绍一些理论了这样我们就可以在进一步讨论之前使用正确的概念和名称。Helm 的应用程序包被称为 chart;可以在本地开发和部署 chart，也可以将 chart 发布到存储库。当你安装一个 chart 时，这被称为发布;每个版本都有一个名称，您可以在集群中安装同一 chart 的多个实例作为独立的、命名的版本,被称作 release。

Charts 包含 Kubernetes YAML 清单，清单通常包含参数化值，因此用户可以使用不同的配置设置安装相同的 Chart——要运行的副本数量或应用程序日志级别可以是参数值。每个 Chart 还包含一组默认值，可以使用命令行检查这些值。图10.3显示了 Helm Chart 的文件结构。

![图10.3](.\images\Figure10.3.png)
<center>图 10.3 Helm Chart 包含应用程序的所有 Kubernetes YAML，加上一些元数据</center>

vweb charts 包包含了我们在第9章中用来演示更新和回滚的简单 web 应用程序。每个 Chart 都包含一个 Service 和 Deployment 的 spec，以及一些参数化值和默认设置。您可以在安装 Chart 之前使用 Helm 命令行检查所有可用值，然后在安装版本时使用自定义值覆盖默认值。

​	<b>现在就试试</b> 检查 vweb Chart 版本 1 中可用的值，然后使用自定义值安装一个 release

```
# 查看 chart 中默认参数信息:
helm show values kiamol/vweb --version 1.0.0

# 安装 chart, 覆盖默认参数值:
helm install --set servicePort=8010 --set replicaCount=1 ch10-vweb
  kiamol/vweb --version 1.0.0

# 检查你安装的 release:
helm ls
```

在本练习中，您可以看到 Chart 中 Service 端口和 Deployment 中的副本数量具有默认值。我的输出如图10.4所示。你使用 helm install 的 set 参数来指定你自己的值，当安装完成时，你有一个应用程序在 Kubernetes 中运行，不使用kubectl，也不直接使用 YAML 清单。

![图10.4](.\images\Figure10.4.png)
<center>图 10.4 使用helm安装应用程序-这将创建Kubernetes资源，而不使用kubectl</center>

Helm 提供了一组用于使用存储库和 Chart 以及安装、更新和回滚版本的特性，但它不适用于应用程序的持续管理。Helm 命令行并不是 kubectl 的替代品—您可以同时使用它们。现在已经安装了 release，您可以以通常的方式使用 Kubernetes 资源，如果需要修改设置，还可以返回 Helm 操作。

​	<b>现在就试试</b> 使用 kubectl 检查 Helm 部署的资源，然后返回Helm 扩容部署并检查应用程序是否正常工作

```
# 查看 Deployment:
kubectl get deploy -l app.kubernetes.io/instance=ch10-vweb --show-
  labels
  
# 更新 release 增加副本数量:
helm upgrade --set servicePort=8010 --set replicaCount=3 ch10-vweb
  kiamol/vweb --version 1.0.0

# 检查 ReplicaSet:
kubectl get rs -l app.kubernetes.io/instance=ch10-vweb

# 获取访问 url:
kubectl get svc ch10-vweb -o
  jsonpath='http://{.status.loadBalancer.ingress[0].*}:8010'

# 浏览器访问 url
```

让我们从这个练习中看看一些东西。首先，标签比标准的 “app” 和“version” 标签要冗长得多。这是因为这是一个公共存储库上的公共 Chart ，所以我使用Kubernetes 配置最佳实践指南中推荐的标签名称——这是我的选择，而不是 Helm 的要求。其次，Helm 升级命令再次指定了Service端口，尽管我想修改的只是副本计数。这是因为 Helm 使用默认值，除非您指定它们，所以如果端口没有包含在升级命令中，它将被更改为默认值。可以在图10.5中看到我的输出。

![图10.5](.\images\Figure10.5.png)
<center>图 10.5 你不使用 Helm 来管理应用程序，但你可以用它来更新配置</center>

这是 Helm 工作流的消费者端。您可以从 Helm 命令行搜索应用程序的存储库，发现应用程序可用的配置值，然后安装和升级应用程序。它是在Kubernetes中运行的应用程序的包管理。在下一节中，您将学习如何打包和发布您自己的应用程序，这是工作流的生产者方面。

## 10.2	使用 Helm 打包你自己的应用

Helm charts are folders or zipped archives that contain Kubernetes manifests. You create your own charts by taking your application manifests, identifying any values you want to be parameterized, and replacing the actual value with a templated variable.


Listing 10.1 shows the beginning of a templated Deployment spec, which uses values set by Helm for the resource name and the label value.

**Listing 10.1	web-ping-deployment.yaml, a templated Kubernetes manifest**

```
apiVersion: apps/v1
kind: Deployment            # This much is standard Kubernetes YAML.

metadata:
  name: {{ .Release.Name }}             # Contains the name of the release
  labels:
    kiamol: {{ .Values.kiamolChapter }} # Contains the “kiamolChapter”
                                        # value
```

The double-brace syntax is for templated values—everything from the opening {{ to the closing }} is replaced at install time, and Helm sends the processed YAML to Kubernetes. Multiple sources can be used as the input to replace templated values. The snippet in listing 10.1 uses the Release object to get the name of the release and the Values object to get a parameter value called kiamolChapter. The Release object is populated with information from the install or upgrade command, and the Values object is populated from defaults in the chart and any settings the user has overridden. Templates can also access static details about the chart and runtime details about the capabilities of the Kubernetes cluster.

​	Helm is very particular about the file structure in a chart. You can use the helm create command to generate the boilerplate structure for a new chart. The top level is a folder whose name has to match the chart name you want to use, and that folder must have at least the following three items:

- A Chart.yaml file that specifies the chart metadata, including the name and version
- A values.yaml file that sets the default values for parameters
- A templates folder that contains the templated Kubernetes manifests

Listing 10.1 is from a file called web-ping-deployment.yaml in the web-ping/templates folder in this chapter’s source. The web-ping folder contains all the files needed for a valid chart, and Helm can validate the chart contents and install a release from the chart folder.

​	<b>现在就试试</b>	When you’re developing charts, you don’t need to package them in zip archives; you can work with the chart folder.**

```
# switch to this chapter’s source:
cd ch10

# validate the chart contents:
helm lint web-ping

# install a release from the chart folder:
helm install wp1 web-ping/

# check the installed releases:
helm ls
```

The lint command is only for working with local charts, but the install command is the same for local charts and for charts stored in a repository. Local charts can be folders or zipped archives, and you’ll see in this exercise that installing a release from a local chart is the same experience as installing from a repository. My output in figure 10.6 shows I now have two releases installed: one from the vweb chart and one from the web-ping chart.

​	The web-ping application is a basic utility that checks whether a website is up by making HTTP requests to a domain name on a regular schedule. Right now, you have

![图10.6](.\images\Figure10.6.png)

​									<center>图 10.6 Installing and upgrading from a local folder lets you iterate quickly on chart development</center>

a Pod running, which is sending requests to my blog every 30 seconds. My blog runs on Kubernetes, so I’m sure it will be able to handle that. The app uses environment variables to configure the URL to use and the schedule interval, and those are templated in the manifest for Helm. Listing 10.2 shows the Pod spec with the templated variables.

**Listing 10.2	web-ping-deployment.yaml, templated container environment**

```
spec:
  containers:
    - name: app
      image: kiamol/ch10-web-ping
      env:
        - name: TARGET
          value: {{ .Values.targetUrl }}
        - name: INTERVAL
          value: {{ .Values.pingIntervalMilliseconds | quote }}
```

Helm has a rich set of templating functions you can use to manipulate the values that get set in the YAML. The quote function in listing 10.2 wraps the provided value in quotation marks, if it doesn’t already have them. You can include looping and branching logic in your templates, calculate strings and numbers, and even query the Kubernetes API to find details from other objects. We won’t get into that much detail, but it’s important to remember that Helm lets you generate sophisticated templates that can do pretty much anything.

​	You need to think carefully about the parts of your spec that need to be templated. One of the big benefits of Helm over standard manifest deployments is that you can run multiple instances of the same app from a single chart. You can’t do that with kubectl because the manifests contain resource names that need to be unique. If you deploy the same set of YAML multiple times, Kubernetes will just update the same resources. If you template all the unique parts of the spec—like resource names and label selectors—then you can run many copies of the same app with Helm.

​	<b>现在就试试</b>	Deploy a second release of the web-ping app, using the samechart folder but specifying a different URL to ping.**

```
# check the available settings for the chart:
helm show values web-ping/

# install a new release named wp2, using a different target:
helm install --set targetUrl=kiamol.net wp2 web-ping/

# wait a minute or so for the pings to fire, then check the logs:
kubectl logs -l app=web-ping --tail 1
```

You’ll see in this exercise that I need to do some optimization on my blog—it returns in around 500 ms whereas the Kiamol website returns in 100 ms. More important, you can see two instances of the app running: two Deployments managing two sets of Pods with different container specs. My output is shown in figure 10.7.

​	It should be clear now that the Helm workflow for installing and managing apps is different from the kubectl workflow, but you also need to understand that the two are incompatible. You can’t deploy the app by running kubectl apply in the templates folder for the chart, because the templated variables are not valid YAML, and the command will fail. If you adopt Helm, you need to choose between using Helm for every environment, which is likely to slow down the developer workflow, or using plain Kubernetes manifests for development and Helm for other environments, which means you’ll have multiple copies of your YAML.

​	Remember that Helm is about distribution and discovery as much as it’s about installation. The additional friction that Helm brings is the price of being able to simplify complex applications down to a few variables and share them on a repository. A repository is really just an index file with a list of chart versions that can be stored on any web server (the Kiamol repository uses GitHub pages, and you can see the whole contents at https://kiamol.net/index.yaml).

![图10.7](.\images\Figure10.7.png)

​									<center>图 10.7 You can’t install multiple instances of an app with plain manifests, but you can with Helm</center>

You can use any server technology to host your repository, but for the rest of this section, we’ll use a dedicated repository server called ChartMuseum, which is a popular open source option. You can run ChartMuseum as a private Helm repository in your own organization, and it’s easy to set up because you can install it with a Helm chart.

​	<b>现在就试试</b>	The ChartMuseum chart is on the official Helm repository, conventionally called “stable.” Add that repository, and you can install a release to run your own repository locally.**

```
# add the official Helm repository:
helm repo add stable https://kubernetes-charts.storage.googleapis.com

# install ChartMuseum—the repo flag fetches details from
# the repository so you don’t need to update your local cache:

helm install --set service.type=LoadBalancer --set
  service.externalPort=8008 --set env.open.DISABLE_API=false repo
  stable/chartmuseum --version 2.13.0 --wait

# get the URL for your local ChartMuseum app:
kubectl get svc repo-chartmuseum -o
  jsonpath='http://{.status.loadBalancer.ingress[0].*}:8008'

# add it as a repository called local:
helm repo add local $(kubectl get svc repo-chartmuseum -o
  jsonpath='http://{.status.loadBalancer.ingress[0].*}:8008')
```

Now you have three repositories registered with Helm: the Kiamol repository, the stable Kubernetes repository (which is a curated set of charts, similar to the official images in Docker Hub), and your own local repository. You can see my output in figure 10.8, which is abridged to reduce the output from the Helm install command.

![图10.8](.\images\Figure10.8.png)

​								<center>图 10.8 Running your own Helm repository is as simple as installing a chart from a Helmrepository</center>

Charts need to be packaged before they can be published to a repository, and publishing is usually a three-stage process: package the chart into a zip archive, upload the archive to a server, and update the repository index to add the new chart. ChartMuseum takes care of the last step for you, so you just need to package and upload the chart for the repository index to be automatically updated.

​	<b>现在就试试</b>	Use Helm to create the zip archive for the chart, and use curl to upload it to your ChartMuseum repository. Check the repository—you’ll see your chart has been indexed.**

```
# package the local chart:
helm package web-ping

# *on Windows 10* remove the PowerShell alias to use the real curl:
Remove-Item Alias:curl -ErrorAction Ignore

# upload the chart zip archive to ChartMuseum:
curl --data-binary "@web-ping-0.1.0.tgz" $(kubectl get svc repo-
  chartmuseum -o
  jsonpath='http://{.status.loadBalancer.ingress[0].*}:8008/api/chart
  s')

# check that ChartMuseum has updated its index:
curl $(kubectl get svc repo-chartmuseum -o jsonpath='http://{.status
  .loadBalancer.ingress[0].*}:8008/index.yaml')
```

Helm uses compressed archives to make charts easy to distribute, and the files are tiny—they contain the Kubernetes manifests and the metadata and values, but they don’t contain any large binaries. Pod specs in the chart specify container images to use, but the images themselves are not part of the chart—they’re pulled from Docker Hub or your own image registry when you install a release. You can see in figure 10.9 that ChartMusem generates the repository index when you upload a chart and adds the new chart details.

​	You can use ChartMuseum or another repository server in your organization to share internal applications or to push charts as part of your continuous integration process before making release candidates available on your public repository. The local repository you have is running only in your lab environment, but it’s published using a LoadBalancer Service, so anyone with network access can install the web-ping app from it.

​	<b>现在就试试</b>	Install yet another version of the web-ping app, this time using the chart from your local repository and providing a values file instead of specifying each setting in the install command.**

```
# update your repository cache:
helm repo update

# verify that Helm can find your chart:
helm search repo web-ping

# check the local values file:
cat web-ping-values.yaml

# install from the repository using the values file:
helm install -f web-ping-values.yaml wp3 local/web-ping

# list all the Pods running the web-ping apps:
kubectl get pod -l app=web-ping -o custom-
  columns='NAME:.metadata.name,ENV:.spec.containers[0].env[*].value'
```

![图10.9](.\images\Figure10.9.png)

​									<center>图 10.9 You can run ChartMuseum as a private repository to easily share charts between teams</center>

In this exercise, you saw another way to install a Helm release with custom settings—using a local values file. That’s a good practice, because you can store the settings for different environments in different files, and you mitigate the risk that an update reverts back to a default value when a setting isn’t provided. My output is shown in figure 10.10.

​	You also saw in the previous exercise that you can install a chart from a repository without specifying a version. That’s not such a good practice, because it installs the latest version, which is a moving target. It’s better to always explicitly state the chart version. Helm requires you to use semantic versioning so chart consumers

![图10.10](.\images\Figure10.10.png)

​										<center>图 10.10 Installing charts from your local repository is the same as installing from any remote repository</center>

know whether the package they’re about to upgrade is a beta release or if it has breaking changes.

​	You can do far more with charts than I’m going to cover here. They can include tests, which are Kubernetes Job specs that run after installation to verify the deployment; they can have hooks, which let you run Jobs at specific points in the installation workflow; and they can be signed and shipped with a signature for provenance. In the next section, I’m going to cover one more feature you use in authoring templates, and it’s an important one—building charts that are dependent on other charts.

## 10.3	charts 中的模块依赖

Helm lets you design your app so it works in different environments, and that raises an interesting problem for dependencies. A dependency might be required in some environments but not in others. Maybe you have a web app that really needs a caching reverse proxy to improve performance. In some environments, you’ll want to deploy the proxy along with the app, and in others, you’ll already have a shared proxy so you just want to deploy the web app itself. Helm supports these with conditional dependencies.

​	Listing 10.3 shows a chart manifest for the Pi web application we’ve been using since chapter 5. It has two dependencies—one from the Kiamol repository, and one from the local filesystem—and they are separate charts.

**Listing 10.3	chart.yaml, a chart that includes optional dependencies**

```
apiVersion: v2 # The version of the Helm spec
name: pi # Chart name
version: 0.1.0 # Chart version
dependencies: # Other charts this chart is dependent on
  - name: vweb
    version: 2.0.0
    repository: https://kiamol.net # A dependency from a repository
    condition: vweb.enabled # Installed only if required
  - name: proxy
    version: 0.1.0
    repository: file://../proxy # A dependency from a local folder
    condition: proxy.enabled # Installed only if required
```

You need to keep your charts flexible when you model dependencies. The parent chart (the Pi app, in this case) may require the subchart (the proxy and vweb charts), but subcharts themselves need to be standalone. You should template the Kubernetes manifests in a subchart to make it generically useful. If it’s something that is useful in only one application, then it should be part of that application chart and not a subchart.

My proxy is generically useful; it’s just a caching reverse proxy, which can use any HTTP server as the content source. The chart uses a templated value for the name of the server to proxy, so although it’s primarily intended for the Pi app, it can be used to proxy any Kubernetes Service. We can verify that by installing a release that proxies an existing app in the cluster.

​	<b>现在就试试</b>	Install the proxy chart on its own, using it as a reverse proxy for the vweb app we installed earlier in the chapter.**

```
# install a release from the local chart folder:
helm install --set upstreamToProxy=ch10-vweb:8010 vweb-proxy proxy/

# get the URL for the new proxy service:
kubectl get svc vweb-proxy-proxy -o
  jsonpath='http://{.status.loadBalancer.ingress[0].*}:8080'
  
# browse to the URL
```

The proxy chart in that exercise is completely independent of the Pi app; it’s being used to proxy the web app I deployed with Helm from the Kiamol repository. You can see in figure 10.11 that it works as a caching proxy for any HTTP server.

![图10.11](.\images\Figure10.11.png)

​										<center>图 10.11 The proxy subchart is built to be useful as a chart in its own right—it can proxy any app</center>

To use the proxy as a dependency, you need to add it in the dependency list in a parent chart, so it becomes a subchart. Then you can specify values for the subchart settings in the parent chart, by prefixing the setting name with the dependency name— the setting upstreamToProxy in the proxy chart is referenced as proxy.upstreamToProxy in the Pi chart. Listing 10.4 shows the default values file for the Pi app, which includes settings for the app itself and for the proxy dependency.

**Listing 10.4	values.yaml, the default settings for the Pi chart**

```
replicaCount: 2 # Number of app Pods to run
serviceType: LoadBalancer # Type of the Pi Service

proxy: # Settings for the reverse proxy
  enabled: false # Whether to deploy the proxy
  upstreamToProxy: "{{ .Release.Name }}-web" # Server to proxy
  servicePort: 8030 # Port of the proxy Service
  replicaCount: 2 # Number of proxy Pods to run
```

These values deploy the app itself without the proxy, using a LoadBalancer Service for the Pi Pods. The setting proxy.enabled is specified as the condition for the proxy dependency in the Pi chart, so the entire subchart is skipped unless the install settings override the default. The full values file also sets the vweb.enabled value to false— that dependency is there only to demonstrate that subcharts can be sourced from repositories, so the default is not to deploy that chart, either.

​	There’s one extra detail to call out here. The name of the Service for the Pi app is templated in the chart, using the release name. That’s important to enable multiple installs of the same chart, but it adds complexity to the default values for the proxy subchart. The name of the server to proxy needs to match the Pi Service name, so the values file uses the same templated value as the Service name, and that links the proxy to the Service in the same release.

​	Charts need to have their dependencies available before you can install or package them, and you use the Helm command line to do that. Building dependencies will populate them into the chart’s charts folder, either by downloading the archive from a repository or packaging a local folder into an archive.

​	<b>现在就试试</b>	Build the dependencies for the Pi chart, which downloads the remote chart, packages the local chart, and adds them to the chart folder.**

```
# build dependencies:
helm dependency build pi

# check that the dependencies have been downloaded:
ls ./pi/charts
```

Figure 10.12 shows why versioning is so important for Helm charts. Chart packages are versioned using the version number in the chart metadata. Parent charts are packaged with their dependencies, at the specified version. If I update the proxy chart without updating the version number, my Pi chart will be out of sync because version 0.1.0 of the proxy chart in the Pi package is different from the latest version 0.1.0. You should consider Helm charts to be immutable and always publish changes by publishing a new package version.

​	This principle of conditional dependencies is how you could manage a much more complex application like the to-do app from chapter 8. The Postgres database deployment would be a subchart, which users could skip altogether for environments where they want to use an external database. Or you could even have multiple conditional dependencies, allowing users to deploy a simple Postgres Deployment for dev environments, use a highly available StatefulSet for test environments, and plug into a managed Postgres service in production.

​	The Pi app is simpler than that, and we can choose whether to deploy it on its own or with a proxy. This chart uses a templated value for the type of the Pi Service, but that could be computed in the template instead by setting it to LoadBalancer if the proxy is not deployed and ClusterIP if the proxy is deployed.

![图10.12](.\images\Figure10.12.png)

​										<center>图 10.12 Helm bundles dependencies into the parent chart, and they are distributed as one package</center>

​	<b>现在就试试</b>	Deploy the Pi app with the proxy subchart enabled. Use Helm’s dry-run feature to check the default deployment, and then use custom settings for the actual install.**

```
# print the YAML Helm would deploy with default values:
helm install pi1 ./pi --dry-run

# install with custom settings to add the proxy:
helm install --set serviceType=ClusterIP --set proxy.enabled=true pi2
./pi

# get the URL for the proxied app:
kubectl get svc pi2-proxy -o
  jsonpath='http://{.status.loadBalancer.ingress[0].*}:8030'
  
# browse to it
```

You’ll see in this exercise that the dry-run flag is quite useful: it applies values to the templates and writes out all the YAML for the resources it would install, without deploying anything. Then in the actual installation, setting a couple of flags deploys an additional chart that is integrated with the main chart, so the application works as a single unit. My Pi calculation appears in figure 10.13.

 

![图10.13](.\images\Figure10.13.png)

​													<center>图 10.13 Installing a chart with an optional subchart by overriding default settings</center>

There’s a whole lot of Helm that I haven’t made room for in this chapter, because there’s a level of complexity you need to dive into only if you bet big on Helm and plan to use it extensively. If that’s you, you’ll find Helm has the power to cover you. How’s this for an example: you can generate a hash from the contents of a ConfigMap template and use that as a label in a Deployment template, so every time the configuration changes, the Deployment label changes too, and upgrading your configuration triggers a Pod rollout.

​	That’s neat, but it’s not for everybody, so in the next section, we’ll return to a simple demo app and look at how Helm smooths the upgrade and rollback process.

## 10.4	升级及回滚 Helm releases

Upgrading an app with Helm doesn’t do anything special; it just sends the updated specs to Kubernetes, which rolls out changes in the usual way. If you want to configure the specifics of the rollout, you still do that in the YAML files in the chart, using the settings we explored in chapter 9. What Helm brings to upgrades is a consistent approach for all types of resources and the ability to easily roll back to previous versions.

​	One other advantage you get with Helm is the ability to safely try out a new version by deploying an additional instance to your cluster. I started this chapter by deploying version 1.0.0 of the vweb app in my cluster, and it’s still running happily. Version 2.0.0 is available now, but before I upgrade the running app, I can use Helm to install a separate release and test the new functionality.

​	<b>现在就试试</b>	Check that the original vweb release is still there, and then install a version 2 release alongside, specifying settings to keep the app private.**

```
# list all releases:
helm ls -q

# check the values for the new chart version:
helm show values kiamol/vweb --version 2.0.0

# deploy a new release using an internal Service type:
helm install --set servicePort=8020 --set replicaCount=1 --set
  serviceType=ClusterIP ch10-vweb-v2 kiamol/vweb --version 2.0.0

# use a port-forward so you can test the app:
kubectl port-forward svc/ch10-vweb-v2 8020:8020

# browse to localhost:8020, then exit the port-forward with Ctrl-C or
# Cmd-C
```

This exercise uses the parameters the chart supports to install the app without making it publicly available, using a ClusterIP Service type and a port-forward so the app is accessible only to the current user. The original app is unchanged, and I have a chance to smoke-test the new Deployment in the target cluster. Figure 10.14 shows the new version running.

​	Now I’m happy that the 2.0.0 version is good, I can use the Helm upgrade command to upgrade my actual release. I want to make sure I deploy with the same values I set in the previous release, and Helm has features to show the current values and to reuse custom values in the upgrade.

​	<b>现在就试试</b>	Remove the temporary version 2 release, and upgrade the ver- sion 1 release to the version 2 chart reusing the same values set on the current release.**

```
# remove the test release:
helm uninstall ch10-vweb-v2

# check the values used in the current version 1 release:
helm get values ch10-vweb

# upgrade to version 2 using the same values—this will fail:
helm upgrade --reuse-values --atomic ch10-vweb kiamol/vweb --version
2.0.0
```

![图10.14](.\images\Figure10.14.png)

​												<center>图 10.14 Charts that deploy Services typically let you set the type, so you can keep them private</center>

Oh dear. This is a particularly nasty issue that will take some tracking down to understand. The reuse-values flag tells Helm to reuse all the values set for the current release on the new release, but the version 2.0.0 chart includes another value, the type of the Service, which wasn’t set in the current release because it didn’t exist. The net result is that the Service type is blank, which defaults to ClusterIP in Kubernetes, and the update fails because that clashes with the existing Service spec. You can see this hinted at in the output in figure 10.15.

![图10.15](./images/Figure10.15.png)

​											<center>图 10.15 An invalid upgrade fails, and Helm can automatically roll back to the previous release</center>

This sort of problem is where Helm’s abstraction layer really helps. You can get the same issue with a standard kubectl deployment, but if one resource update fails, you need to check through all the other resources and manually roll them back. Helm does that automatically with the atomic flag. It waits for all the resource updates to complete, and if any of them fails, it rolls back every other resource to the previous state. Check the history of the release, and you can see that Helm has automatically rolled back to version 1.0.0.

​	<b>现在就试试</b>	Recall from chapter 9 that Kubernetes doesn’t give you much information on the history of a rollout—compare that to the detail you get from Helm.**

```
# show the history of the vweb release:
helm history ch10-vweb
```

That command gets an exercise all to itself, because there’s a wealth of information that you just don’t get in the history for a standard Kubernetes rollout. Figure 10.16 shows all four revisions of the release: the first install, a successful upgrade, a failed upgrade, and an automatic rollback.

![图10.16](.\images\Figure10.16.png)

​													<center>图 10.16 The release history clearly links application and chart versions to revisions</center>

To fix the failed update, I can manually set all the values in the upgrade command or use a values file with the same settings that are currently deployed. I don’t have that values file, but I can save the output of the get values command to a file and use that in the upgrade, which gives me all my previous settings plus the defaults in the chart for any new settings.

​	<b>现在就试试</b>	Upgrade to version 2 again, this time saving the current version 1 values to a file and using that in the upgrade command.**

```
# save the values of the current release to a YAML file:
helm get values ch10-vweb -o yaml > vweb-values.yaml

# upgrade to version 2 using the values file and the atomic flag:
helm upgrade -f vweb-values.yaml --atomic ch10-vweb kiamol/vweb
  --version 2.0.0
  
# check the Service and ReplicaSet configuration:
kubectl get svc,rs -l app.kubernetes.io/instance=ch10-vweb
```

This upgrade succeeds, so the atomic rollback doesn’t kick in. The upgrade is actually effected by the Deployment, which scales up the replacement ReplicaSet and scales down the current ReplicaSet in the usual way. Figure 10.17 shows that the configuration values set in the previous release have been retained, the Service is listening on port 8010, and three Pods are running.

​	All that’s left is to try out a rollback, which is syntactically similar to a rollback in kubectl, but Helm makes it much easier to track down the revision you want to use. You’ve already seen the meaningful release history in figure 10.16, and you can also use Helm to check the values set for a particular revision. If I want to roll back the web

![图10.17](.\images\Figure10.17.png)

​										<center>图 10.17 The upgrade succeeds by exporting the release settings to a file and using them again</center>

application to version 1.0.0 but preserve the values I set in revision 2, I can check those values first.

​	<b>现在就试试</b>	Roll back to the second revision, which was version 1.0.0 of the app upgraded to use three replicas.**

```
# confirm the values used in revision 2:
helm get values ch10-vweb --revision 2

# roll back to that revision:
helm rollback ch10-vweb 2

# check the latest two revisions:
helm history ch10-vweb --max 2 -o yaml
```

You can see my output in figure 10.18, where the rollback is successful and the history shows that the latest revision is 6, which is actually a rollback to revision 2.

​	The simplicity of this example is good for focusing on the upgrade and rollback workflow, and highlighting some of the quirks, but it hides the power of Helm for major upgrades. A Helm release is an abstraction of an application, and different

![图10.18](.\images\Figure10.18.png)

​																	<center>图 10.18 Helm makes it easy to check exactly what you’re rolling back to</center>

versions of the application might be modeled in different ways. A chart might use a ReplicationController in an early release, then change to a ReplicaSet and then a Deployment; as long as the user-facing parts remain the same, the internal workings become an implementation detail.

## 10.5	理解 Helm 定位

Helm adds a lot of value to Kubernetes, but it’s invasive—once you template your manifests, there’s no going back. Everyone on the team has to switch to Helm, or you have to commit to having multiple sets of manifests: pure Kubernetes for the development team and Helm for every other environment. You really don’t want two sets of manifests getting out of sync, but equally, Kubernetes itself is plenty to learn without adding Helm on top.

​	Whether Helm fits in for you depends very much on the type of applications you’re packaging and the way your teams work. If your app is composed of 50+ microservices, then development teams might work on only a subset of the full app, running it natively or with Docker and Docker Compose, and a separate team owns the full Kubernetes deployment. In that environment, a move to Helm will reduce friction rather than increasing it, centralizing hundreds of YAML files into manageable charts.

​	A couple of other indicators that Helm is a good fit include a fully automated continuous deployment process—which can be easier to build with Helm—running test environments from the same chart version with custom values files, and running verification jobs as part of the deployment. When you find yourself needing to template your Kubernetes manifests—which you will sooner or later—Helm gives you a standard approach, which is better than writing and maintaining your own tools.

​	That’s all for Helm in this chapter, so it’s time to tidy up the cluster before moving on to the lab.

​	<b>现在就试试</b>	Everything in this chapter was deployed with Helm, so we can use Helm to uninstall it all.**

```
# uninstall all the releases:
helm uninstall $(helm ls -q)
```

## 10.6	实验室

It’s back to the to-do app again for the lab. You’re going to take a working set of Kubernetes manifests and package them into a Helm chart. Don’t worry—it’s not the full-on app from chapter 8 with StatefulSets and backup Jobs; it’s a much simpler version. Here are the goals:

- Use the manifests in the lab/todo-list folder as the starting point (there are hints in the YAML for what needs templating).
- Create the Helm chart structure.
- Template the resource names and any other values that need to be templated so the app can run as multiple releases.
- Add parameters for configuration settings to support running the app as different environments.
- Your chart should run as the Test configuration when installed with default values.
- Your chart should run as the Dev configuration when installed using the lab/dev-values.yaml values file.

If you’re planning on making use of Helm, you should really find time for this lab, because it contains the exact set of tasks you’ll need to do when you package apps in Helm. My solution is on GitHub for you to check in the usual place: https://github.com/sixeyed/kiamol/blob/master/ch10/lab/README.md.

​	Happy Helming!