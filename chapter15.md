# 第十五章 使用 Ingress 管理流入流量

Services 将网络流量引入 Kubernetes，您可以拥有多个具有不同公共 IP 地址的 LoadBalancer 服务，以使您的 Web 应用程序可供全世界使用。这样做会给管理带来麻烦，因为这意味着为每个应用程序分配一个新的 IP 地址，并将地址映射到您的 DNS 提供商的应用程序。将流量转移到正确的应用程序是一个路由问题，但您可以使用 Ingress 在 Kubernetes 内部管理它。 Ingress 使用一组规则将域名和请求路径映射到应用程序，因此您可以为整个集群使用一个 IP 地址并在内部路由所有流量。

域名路由是一个老问题，通常已经用反向代理解决了，而 Kubernetes 对 Ingress 使用了可插拔的架构。您将路由规则定义为标准资源，并部署您选择的反向代理来接收流量并根据规则执行操作。所有主要的反向代理都有 Kubernetes 支持，以及一种新的容器感知反向代理。它们都有不同的功能和工作模型，在本章中，您将学习如何使用 Ingress 在集群中托管多个应用程序，其中两个应用程序是最流行的：Nginx 和 Traefik。

## 15.1 Kubernetes 如何使用 Ingress 路由流量

我们已经在本书中多次使用 Nginx 作为反向代理（据我统计 17 次），但我们总是一次将它用于一个应用程序。我们在第 6 章中使用了一个反向代理来缓存来自 Pi 应用程序的响应，在第 13 章中使用了另一个用于缓存随机数 API 的响应。Ingress 将反向代理移至中心角色，将其作为称为 ingress 控制器的组件运行，但方法是相同：代理从 LoadBalancer 服务接收外部流量，并使用 ClusterIP 服务从应用程序中获取内容。图 15.1 显示了架构。

![图15.1](./images/Figure15.1.png)
<center>图 15.1 Ingress 控制器是集群的入口点，根据 Ingress 规则路由流量。</center>

这张图中最重要的是 ingress 控制器，它是可插入的反向代理——它可能是 Nginx、HAProxy、Contour 和 Traefik 等十几个选项之一。 Ingress 对象以通用方式存储路由规则，控制器将这些规则提供给代理。代理具有不同的功能集，并且 Ingress 规范不会尝试对每个可能的选项进行建模，因此控制器使用注释添加对这些功能的支持。您将在本章中了解到，路由和 HTTPS 支持的核心功能使用起来很简单，但复杂性在于 ingress 控制器部署及其附加功能。

我们将从运行第 2 章中的基本 Hello, World Web 应用程序开始，将其作为具有 ClusterIP 服务的内部组件，并使用 Nginx ingress 控制器来路由流量。

立即尝试,运行 Hello, World 应用程序，并确认它只能在集群内部或外部使用 kubectl 中的端口转发访问。

```
# 进入本章目录:
cd ch15
# 部署 web app:
kubectl apply -f hello-kiamol/
# 确认服务是集群内部的:
kubectl get svc hello-kiamol
# 启动到应用程序的端口转发:
kubectl port-forward svc/hello-kiamol 8015:80
# 访问 http://localhost:8015
# 然后按Ctrl-C/Cmd-C退出端口转发
```

该应用程序的部署或服务规范中没有任何新内容——没有特殊标签或注释，没有您尚未使用过的新字段。您可以在图 15.2 中看到该服务没有外部 IP 地址，只有在运行端口转发时我才能访问该应用程序。

![图15.2](./images/Figure15.2.png)
<center>ClusterIP 服务使应用程序在内部可用——它可以通过 Ingress 公开。</center>

要使用 Ingress 规则使应用程序可用，我们需要一个 Ingress 控制器。控制器管理其他对象。你知道 Deployments 管理 ReplicaSets 和ReplicaSets 管理 Pod。Ingress 控制器略有不同；它们在标准 Pod 中运行并监控 Ingress 对象。当他们看到任何变化时，他们会更新代理中的规则，我们将从 Nginx ingress 控制器开始，它是更广泛的 Kubernetes 项目的一部分。控制器有一个生产就绪的 Helm chart，但我使用的部署要简单得多。即便如此，清单中仍有一些我们尚未涵盖的安全组件，但我现在不会详细介绍它们。 （如果您想调查，YAML 中有注释。）

立即尝试,部署 Nginx ingress 控制器。这使用服务中的标准 HTTP 和 HTTPS 端口，因此您的计算机上需要有可用的端口 80 和 443。

```
# 为Nginx Ingress 控制器创建部署和服务:
kubectl apply -f ingress-nginx/
# 确认该服务是公开可用的:
kubectl get svc -n kiamol-ingress-nginx
# 获取代理的URL:
kubectl get svc ingress-nginx-controller -o jsonpath='http://{.status.loadBalancer.ingress[0].*}' -n kiamol-ingress-nginx
# 访问 URL—你将发现 error
```

当你运行这个练习时，你会在浏览时看到一个 404 错误页面。这证明服务正在接收流量并将其定向到 ingress 控制器，但是还没有任何路由规则，因此 Nginx 没有内容可显示，它返回默认的未找到页面。我的输出如图 15.3 所示，您可以在其中看到服务正在使用标准 HTTP 端口。

![图15.3](./images/Figure15.3.png)
<center>图 15.3 控制器接收传入流量，但它们需要路由规则来知道如何处理它 </center>

现在我们有一个正在运行的应用程序和一个 Ingress 控制器，我们只需要部署一个带有路由规则的 ingress 对象来告诉控制器每个传入请求使用哪个应用程序服务。清单 15.1 显示了 Ingress 对象的最简单规则，它将每个进入集群的请求路由到 Hello, World 应用程序。

> 清单 15.1 localhost.yaml，Hello, World 应用程序的路由规则

```
apiVersion: networking.k8s.io/v1beta1 # Beta API版本意味着规范不是最终的，可能会改变。
kind: Ingress 
metadata:
  name: hello-kiamol
spec:
  rules:
  - http: # Ingress 仅用于HTTP/S流量
      paths:
        - path: / # 将每个传入请求映射到hello-kiamol服务
          backend:
            serviceName: hello-kiamol
            servicePort: 80
```

Ingress 控制器正在监视新的和更改的 ingress 对象，因此当您部署任何对象时，它会将规则添加到 Nginx 配置中。在 Nginx 术语中，它将设置一个代理服务器，其中 hello-kiamol 服务是上游（内容的来源），它将为根路径的传入请求提供该内容。

立即尝试,创建通过 ingress 控制器发布 Hello, World 应用程序的入口规则。

```
# 部署 rule:
kubectl apply -f hello-kiamol/ingress/localhost.yaml
# 确认Ingress对象已经创建:
kubectl get ingress
# 从前面的练习中刷新浏览器
```

好吧，这很简单——在 Ingress 对象中为应用程序映射到后端服务的路径，控制器负责处理其他所有事情。我在图 15.4 中的输出显示了本地主机地址，它之前返回了 404 错误，现在返回了 Hello, World 应用程序的所有荣耀。

![图15.4](./images/Figure15.4.png)
<center>图 15.4 Ingress 对象规则将 Ingress 控制器链接到应用服务。</center>

Ingress 通常是集群中的集中服务，例如日志记录和监控。管理团队可能会部署和管理 Ingress 控制器，而每个产品团队都拥有将流量路由到其应用程序的 Ingress 对象。这个过程可能会产生冲突—— ingress 规则不必是唯一的，一个团队的更新最终可能会将另一个团队的所有流量重定向到其他应用程序。这种情况不会发生，因为这些应用程序将托管在不同的域中，并且 Ingress 规则将包含一个域名来限制它们的范围。

## 15.2 使用 Ingress rules 路由 Http 流量

Ingress works only for web traffic—HTTP and HTTPS requests—because it needs to use the route specified in the request to match it to a backend service. The route in an HTTP request contains two parts: the host and the path. The host is the domain name, like www.manning.com, and the path is the location of the resource, like /dotd for the Deal of the Day page. Listing 15.2 shows an update to the Hello, World Ingress object that uses a specific host name. Now the routing rules will apply only if the incoming request is for the host hello.kiamol.local.

> Listing 15.2 hello.kiamol.local.yaml, specifying a host domain for Ingress rules

```
spec:
rules:
- host: hello.kiamol.local # Restricts the scope of the
http: # rules to a specific domain
paths:
- path: / # All paths in that domain will
backend: # be fetched from the same Service.
serviceName: hello-kiamol
servicePort: 80
```
When you deploy this code, you won’t be able to access the app because the domain name hello.kiamol.local doesn’t exist. Web requests normally look up the IP address for a domain name from a public DNS server, but all computers also have their own local list in a hosts file. In the next exercise, you’ll deploy the updated Ingress object and register the domain name in your local hosts file—you’ll need admin access in your terminal session for that.

TRY IT NOW 
Editing the hosts file is restricted. You’ll need to use the “Run as Administrator” option for your terminal session in Windows and have scripts enabled with the Set-ExecutionPolicy command. Be ready to enter your admin (sudo) password in Linux or macOS.

   ```
   # add domain to hosts—on Windows:
   ./add-to-hosts.ps1 hello.kiamol.local ingress-nginx
   # OR on Linux/macOS:
   chmod +x add-to-hosts.sh && ./add-to-hosts.sh hello.kiamol.local
   ingress-nginx
   # update the Ingress object, adding the host name:
   kubectl apply -f hello-kiamol/ingress/hello.kiamol.local.yaml
   # confirm the update:
   kubectl get ingress
   # browse to http://hello.kiamol.local
   ```

In this exercise, the existing Ingress object is updated, so there’s still only one routing rule for the ingress controller to map. Now that rule is restricted to an explicit domain name. You can see in figure 15.5 that the request to hello.kiamol.local returns the app, and I’ve also browsed to the ingress controller at localhost, which returns a 404 error because there are no rules for the localhost domain.

![图15.5](./images/Figure15.5.png)
<center>图 15.5 You can publish apps by domain name with Ingress rules and use them locally by editing your hosts file. </center>

Routing is an infrastructure-level concern, but like the other shared services we’ve seen in this section of the book, it runs in lightweight containers so you can use
xactly the same setup in development, test, and production environments. That lets you run multiple apps in your nonproduction cluster with friendly domain names, without having to use different ports—the Service for the ingress controller uses the standard HTTP port for every app.

You need to fiddle with your hosts file if you want to run multiple apps with different domains in your lab environment. Typically, all the domains will resolve to 127.0.0.1, which is your machine’s local address. Organizations might run their own DNS server in a test environment, so anyone can access hello.kiamol.test from the company network, and it will resolve to the IP address of the test cluster, running in the data center. Then, in production, the DNS resolution is from a public DNS service, so hello.kiamol.net resolves to a Kubernetes cluster running in the cloud.

You can combine host names and paths in Ingress rules to present a consistent set of addresses for your application, although you could be using different components in the backend. You might have a REST API and a website running in separate Pods, and you could use Ingress rules to make the API available on a subdomain (api.rng.com) or as a path on the main domain (rng.com/api). Listing 15.3 shows Ingress rules for the simple versioned web app from chapter 9, where both versions of the app are available from one domain.

> Listing 15.3 vweb/ingress.yaml, Ingress rules with host name and paths

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
name: vweb # Configures a specific feature
annotations: # in Nginx
nginx.ingress.kubernetes.io/rewrite-target: /
spec:
rules:
- host: vweb.kiamol.local # All rules apply to this domain.
http:
paths:
- path: / # Requests to the root path are
backend: # proxied from the version 2 app.
serviceName: vweb-v2
servicePort: 80
- path: /v1 # Requests to the /v1 path are
backend: # proxied from the version 1 app.
serviceName: vweb-v1
servicePort: 80
```
Modeling paths adds complexity because you’re presenting a fake URL, which needs to be modified to match the real URL in the service. In this case, the ingress controller will respond to requests for http://vweb.kiamol.local/v1 and fetch the content from the vweb-v1 service. But the application doesn’t have any content at /v1, so the proxy needs to rewrite the incoming URL—that’s what the annotation does in listing 15.3. It’s a basic example that ignores the path in the request and always uses the root path in the backend. You can’t express URL rewrites with the Ingress spec, so it needs custom support from the ingress controller. A more realistic rewrite rule would use regular expressions to map the requested path to the target path.

We’ll deploy this simple version to avoid any regular expressions and see how the ingress controller uses routing rules to identify the backend service and modify the request path.

TRY IT NOW 
Deploy a new app with a new Ingress rule, and add a new domain to your hosts file to see the ingress controller serving multiple applications from the same domain.

   ```
   # add the new domain name—on Windows:
   ./add-to-hosts.ps1 vweb.kiamol.local ingress-nginx
   # OR on Linux/macOS:
   ./add-to-hosts.sh vweb.kiamol.local ingress-nginx
   # deploy the app, Service, and Ingress:
   kubectl apply -f vweb/
   # confirm the Ingress domain:
   kubectl get ingress
   # browse to http://vweb.kiamol.local
   # and http://vweb.kiamol.local/v1
   ```
In figure 15.6, you can see that two separate apps are made available at the same domain name, using the request path to route between different components, which are different versions of the application in this exercise.

![图15.6](./images/Figure15.6.png)
<center>图 15.6 Ingress routing on the host name and path presents multiple apps on the same domain name. </center>

Mapping the routing rules is the most complicated part of publishing a new app to your ingress controller, but it does give you a lot of control. The Ingress rules are the public face of your app, and you can use them to compose several components—or to restrict access to features. In this section of the book, we’ve seen that apps work better in Kubernetes if they have health endpoints for container probes and metrics endpoints for Prometheus to scrape, but those shouldn’t be publicly available. You canuse Ingress to control that, using exact path mappings, so only paths that are explicitly

listed are available outside of the cluster. Listing 15.4 shows an example of that for the to-do list app. It’s abridged because the downside with this approach is that you need to specify every path you want to publish, so any paths not specified are blocked.

> Listing 15.4 ingress-exact.yaml, using exact path matching to restrict access

```
rules:
- host: todo.kiamol.local
http:
paths:
- pathType: Exact # Exact matching means only the /new
path: /new # path is matched—there are other
backend: # rules for the /list and root paths.
serviceName: todo-web
servicePort: 80
- pathType: Prefix # Prefix matching means any path that
path: /static # starts with /static will be mapped,
backend: # including subpaths like /static/app.css.
serviceName: todo-web
servicePort: 80
```
The to-do list app has several paths that shouldn’t be available outside of the clusteras well as /metrics, there’s a /config endpoint that lists all of the application configurations and a diagnostics page. None of those paths is included in the new Ingress spec, and we can see that they’re effectively blocked when the rules are applied. The PathType field is a later addition to the Ingress spec, so your Kubernetes cluster needs to be running at least version 1.18; otherwise, you’ll get an error in this exercise.

TRY IT NOW 
Deploy the to-do list app with an Ingress spec that allows all access, and then update it with exact path matching, and confirm that the sensitive paths are no longer available.

   ```
   # add a new domain for the app—on Windows:
   ./add-to-hosts.ps1 todo.kiamol.local ingress-nginx
   # OR on Linux/macOS:
   ./add-to-hosts.sh todo.kiamol.local ingress-nginx
   # deploy the app with an Ingress object that allows all paths:
   kubectl apply -f todo-list/
   # browse to http://todo.kiamol.local/metrics
   # update the Ingress with exact paths:
   kubectl apply -f todo-list/update/ingress-exact.yaml
   # browse again—the app works, but metrics and diagnostics blocked
   ```

You’ll see when you run this exercise that all of the sensitive paths are blocked when you deploy the updated Ingress rules. My output is shown in figure 15.7. It’s not a perfect solution, but you can extend your ingress controller to show a friendly 404 error page instead of the Nginx default. (Docker has a nice example: try `https://www.docker.com/not-real-url`.) The app still shows a menu for the diagnostics page because it’s not an app setting that removes the page; it’s happening earlier in the process.

![图15.7](./images/Figure15.7.png)
<center>图 15.7 Exact path matching in Ingress rules can be used to block access to features. </center>

The separation between Ingress rules and the ingress controller makes it easy to compare different proxy implementations and see which gives you the combination of features and usability you’re happy with. But it comes with a warning because there isn’t a strict ingress controller specification, and not every controller implements Ingress rules in the same way. Some controllers ignore the PathType field, so if you’re relying on that to build up an access list with exact paths, you may find your site becomes access-all-areas if you switch to a different ingress controller.

Kubernetes does let you run multiple ingress controllers, and in a complex environment, you may do that to provide different sets of capabilities for different applications.

## 15.3 比较 Ingress 控制器

Ingress controllers come in two categories: reverse proxies, which have been around for a long time and work at the network level, fetching content using host names; and modern proxies, which are platform-aware and can integrate with other services (cloud controllers can provision external load balancers). Choosing between them comes down to the feature set and your own technology preferences. If you have an established relationship with Nginx or HAProxy, you can continue that in Kubernetes. Or if you have an established relationship with Nginx or HAProxy, you might be glad to try a more lightweight, modern option.

Your ingress controller becomes the single public entry point for all of the apps in your cluster, so it’s a good place to centralize common concerns. All controllers support SSL termination, so the proxy provides the security layer, and you get HTTPS for all of your applications. Most controllers support web application firewalls, so you can provide protection from SQL injection and other common attacks in the proxy layer. Some controllers have special powers—we’ve already used Nginx as a caching proxy, and you can use it for caching at the ingress level, too.

TRY IT NOW 
Deploy the Pi application using Ingress, then update the Ingress object so the Pi app makes use of the Nginx cache in the ingress controller.

   ```
   # add the Pi app domain to the hosts file—Windows:
   ./add-to-hosts.ps1 pi.kiamol.local ingress-nginx
   # OR Linux/macOS:
   ./add-to-hosts.sh pi.kiamol.local ingress-nginx
   # deploy the app and a simple Ingress:
   kubectl apply -f pi/
   # browse to http://pi.kiamol.local?dp=30000
   # refresh and confirm the page load takes the same time
   # deploy an update to the Ingress to use caching:
   kubectl apply -f pi/update/ingress-with-cache.yaml
   # browse to the 30K Pi calculation again—the first
   # load takes a few seconds, but now a refresh will be fast
   ```

You’ll see in this exercise that the ingress controller is a powerful component in the cluster. You can add caching to your app just by specifying new Ingress rules—no updates to the application itself and no new components to manage. The only requirement is that the HTTP responses from your app include the right caching headers, which they should anyway. 图 15.8 shows my output, where the Pi calculation took 1.2 seconds, but the response came from the ingress controller’s cache, so the page loaded pretty much instantly.

![图15.8](./images/Figure15.8.png)
<center>图 15.8 If your ingress controller supports response caching, that’s an easy performance boost. </center>

Not every ingress controller provides a response cache, so that’s not a specific part of the Ingress spec. Any custom configuration is applied with annotations, which the controller picks up. Listing 15.5 shows the metadata for the updated cache setting you applied in the previous exercise. If you’re familiar with Nginx, you’ll recognize these as the proxy cache settings you would normally set in the configuration file.

> Listing 15.5 ingress-with-cache.yaml, using the Nginx cache in the ingress controller

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata: # The ingress controller looks in annotations for
name: pi # custom configuration—this adds proxy caching.
annotations:
nginx.ingress.kubernetes.io/proxy-buffering: "on"
nginx.ingress.kubernetes.io/configuration-snippet: |
proxy_cache static-cache;
proxy_cache_valid 10m;
```
The configuration in an Ingress object applies to all of its rules, but if you need different features for different parts of your app, you can have multiple Ingress rules. That’s true for the to-do list app, which needs some more help from the ingress controller to work properly at scale. Ingress controllers use load balancing if a Service has many Pods, but the to-do app has some cross-site forgery protection, which breaks if the request to create a new item is sent to a different app container from the one that originally rendered the new item page. Lots of apps have a restriction like this, which proxies solve using sticky sessions.

Sticky sessions are a mechanism for the ingress controller to send requests from the same end user to the same container, which is often a requirement for older apps where components aren’t stateless. It’s something to avoid where possible because it limits the cluster’s potential for load balancing, so in the to-do list app, we want to restrict it to just one page. 图 15.9 shows the Ingress rules we’ll apply to get differ- ent features for different parts of the app.

![图15.9](./images/Figure15.9.png)
<center>图 15.9 A domain can be mapped with multiple Ingress rules, using different proxy features. </center>

We can scale up the to-do app now to understand the problem and then apply the updated Ingress rules to fix it.

TRY IT NOW 
Scale up the to-do application to confirm that it breaks without sticky sessions, and then deploy the updated Ingress rules from figure 15.9 and confirm it all works again.

   ```
   # scale up—the controller load-balances between the Pods:
   kubectl scale deploy/todo-web --replicas 3
   # wait for the new Pods to start:
   kubectl wait --for=condition=ContainersReady pod -l app=todo-web
   # browse to http://todo.kiamol.local/new, and add an item
   # this will fail and show a 400 error page
   # print the application logs to see the issue:
   kubectl logs -l app=todo-web --tail 1 --since 60s
   # update Ingress to add sticky sessions:
   kubectl apply -f todo-list/update/ingress-sticky.yaml
   # browse again, and add a new item—this time it works
   ```
You can see my output in figure 15.10, but unless you run the exercise yourself, you’ll have to take my word for which is the “before” and which is the “after” screenshot. Scaling up the application replicas means requests from the ingress controller are load-balanced, which triggers the antiforgery error. Applying sticky sessions stops load balancing on the new item path, so a user’s requests are always routed to the same Pod, and the forgery check passes.

![图15.10](./images/Figure15.10.png)
<center>图 15.10 Proxy features can fix problems as well as improve performance. </center>

The Ingress resources for the to-do app use a combination of host, paths, and annotations to set all the rules and the features to apply. Behind the scenes, the job of the controller is to convert those rules into proxy configuration, which in the case of Nginx means writing a config file. The controller features lots of optimizations to minimize the number of file writes and configuration reloads, but as a result, the Nginx configuration file is horribly complex. If you opt for the Nginx ingress controller because you have Nginx experience and you’d be comfortable debugging the config file, you’re in for an unpleasant surprise.

TRY IT NOW 
The Nginx configuration is in a file in the ingress controller Pod. Run a command in the Pod to check the size of the file.

   ```
   # run the wc command to see how many lines are in the file:
   kubectl exec -n kiamol-ingress-nginx deploy/ingress-nginx-controller -
   - sh -c 'wc -l /etc/nginx/nginx.conf'
   ```

图 15.11 shows there are more than 1,700 lines in my Nginx configuration file. If you run cat instead of wc, you’ll find the contents are strange, even if you’re familiar

![图15.11](./images/Figure15.11.png)
<center>图 15.11 The generated Nginx configuration file is not made to be human-friendly. </center>

with Nginx. (The controller uses Lua scripts so it can update endpoints without a configuration reload.)

The ingress controller owns that complexity, but it’s a critical part of your solution, and you need to be happy with how you’ll troubleshoot and debug the proxy. This is when you might want to consider an alternative ingress controller that is platform aware and doesn’t run from a complex configuration file. We’ll look at Traefik in this chapter—it’s an open source proxy that has been gaining popularity since it launched in 2015. Traefik understands containers, and it builds its routing list from the plat- form API, natively supporting Docker and Kubernetes, so it doesn’t have a config file to maintain.

Kubernetes supports multiple Ingress controllers running in a single cluster. They’ll be exposed as LoadBalancer Services, so in production, you might have different IP addresses for different ingress controllers, and you’ll need to map domains to Ingress in your DNS configuration. In our lab environment, we’ll be back to using different ports. We’ll start by deploying Traefik with a custom port for the ingress controller Service.

TRY IT NOW 
Deploy Traefik as an additional ingress controller in the cluster.

   ```
   # create the Traefik Deployment, Service, and security resources:
   kubectl apply -f ingress-traefik/
   # get the URL for the Traefik UI running in the ingress controller:
   kubectl get svc ingress-traefik-controller -o
   jsonpath='http://{.status.loadBalancer.ingress[0].*}:8080' -n
   kiamol-ingress-traefik
   # browse to the admin UI to see the routes Traefik has mapped
   ```
You’ll see in that exercise that Traefik has an admin UI. It shows you the routing rules the proxy is using, and as traffic passes through, it can collect and show performance metrics It’s much easier to work with than the Nginx config file. 图 15.12 shows two routers, which are the incoming routes Traefik manages. If you explore the dashboard, you’ll see those aren’t Ingress routes; they’re internal routes for Traefik’s own dashboard—Traefik has not picked up any of the existing Ingress rules in the cluster.

![图15.12](./images/Figure15.12.png)
<center>图 15.12 Traefik is a container-native proxy that builds routing rules from the platform and has a UI to display them. </center>

Why hasn’t Traefik built a set of routing rules for the to-do list or the Pi applications? It would if we had configured it differently, and all the existing routes would be available via the Traefik Service, but that’s not how you use multiple ingress controllers because they would end up fighting over incoming requests. You run more than one controller to provide different proxy capabilities, and you need the application to choose which one to use. You do that with ingress classes, which are a similar concept to storage classes. Traefik has been deployed with a named ingress class, and only Ingress objects that request that class will be routed through Traefik.

The ingress class isn’t the only difference between ingress controllers, and you may need to model your routes quite differently for different proxies. 图 15.13 shows how the to-do app needs to be configured in Traefik. There’s no response cache in Traefik so we don’t get caching for static resources, and sticky sessions are configured
at the Service level, so we need an additional Service for the new item route.

![图15.13](./images/Figure15.13.png)
<center>图 15.13 Ingress controllers work differently, and your route model will need to change accordingly. </center>

That model is significantly different from the Nginx routing in figure 15.9, so if you do plan to run multiple ingress controllers, you need to appreciate the high risk of misconfiguration, with teams confusing the different capabilities and approaches. Traefik uses annotations on Ingress resources to configure the routing rules. List- ing 15.6 shows a spec for the new-item path, which selects Traefik as the ingress class

and uses an annotation for exact path matching, because Traefik doesn’t support the PathType field.

> Listing 15.6 ingress-traefik.yaml, selecting the ingress class with Traefik annotations

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata: # Annotations select the Traefik
name: todo2-new # ingress class and apply exact
annotations: # path matching.
kubernetes.io/ingress.class: traefik
traefik.ingress.kubernetes.io/router.pathmatcher: Path
spec:
rules:
- host: todo2.kiamol.local # Uses a different host so the app
http: # stays avaialable through Nginx
paths:
- path: /new
backend:
serviceName: todo-web-sticky # Uses the Service that has sticky
servicePort: 80 # sessions configured for Traefik
```
We’ll deploy a new set of Ingress rules using a different host name, so we can route traffic to the same set of to-do list Pods via Nginx or Traefik.

TRY IT NOW 
Publish the to-do app through the Traefik ingress controller, using the Ingress routes modeled in figure 15.13.

   ```
   # add a new domain for the app—on Windows:
   ./add-to-hosts.ps1 todo2.kiamol.local ingress-traefik
   # OR on Linux/macOS:
   ./add-to-hosts.sh todo2.kiamol.local ingress-traefik
   # apply the new Ingress rules and sticky Service:
   kubectl apply -f todo-list/update/ingress-traefik.yaml
   # refresh the Traefik admin UI to confirm the new routes
   # browse to http://todo2.kiamol.local:8015
   ```
Traefik watches for events from the Kubernetes API server and refreshes its routing list automatically. When you deploy the new Ingress objects, you’ll see the paths shown as routers in the Traefik dashboard, linked to the backend Services. 图 15.14 shows part of the routing list, together with the to-do app available through the new URL.

![图15.14](./images/Figure15.14.png)
<center>图 15.14 Ingress controllers achieve the same goals from different configuration models. </center>

If you’re evaluating ingress controllers, you should look at the ease of modeling your application paths, together with the approach to troubleshooting and the performance of the proxy. Dual-running controllers in a dedicated environment helps with that because you can isolate other factors and run comparisons using the same application components. A more realistic app will have more complex Ingress rules, and you’ll want to be comfortable with how the controller implements features like rate limiting, URL rewrites, and client IP access lists.

The other major feature of Ingress is publishing apps over HTTPS without configuring certificates and security settings in your applications. This is one area that is consistent among ingress controllers, and in the next section, we’ll see it with Traefik and Nginx.

## 15.4 使用 Ingress 通过 HTTPS 保护您的应用程序

Your web apps should be published over HTTPS, but encryption needs server certificates, and certificates are sensitive data items. It’s a good practice to make HTTPS an ingress concern, because it centralizes certificate management. Ingress resources can specify a TLS certificate in a Kubernetes Secret (TLS is Transport Layer Security, the encryption mechanism for HTTPS). Moving TLS away from application teams means you can have a standard approach to provisioning, securing, and renewing certificatesand you won’t have to spend time explaining why packaging certificates inside a container image is a bad idea.

All ingress controllers support loading a TLS certificate from a Secret, but Traefik makes it easier still. If you want to use HTTPS in development and test environments without provisioning any Secrets, Traefik can generate its own self-signed certificate when it runs. You configure that with annotations in the Ingress rules to enable TLS and the default certificate resolver.

TRY IT NOW 
Using Traefik’s generated certificate is a quick way to test your app over HTTPS. It’s enabled with more annotations in the Ingress objects.

   ```
   # update the Ingress to use Traefik’s own certifcate:
   kubectl apply -f todo-list/update/ingress-traefik-certResolver.yaml
   # browse to https://todo2.kiamol.local:9443
   # you’ll see a warning in your browser
   ```
Browsers don’t like self-signed certificates because anyone can create them—there’s no verifiable chain of authority. You’ll see a big warning when you first browse to the site, telling you it’s not safe, but you can proceed, and the to-do list app will load. As shown in figure 15.15, the site is encrypted with HTTPS but with a warning so you know it’s not really secure.

![图15.15](./images/Figure15.15.png)
<center>图 15.15 Not all HTTPS is secure—self-signed certificates are fine for development and test environments. </center>

Your organization will probably have its own idea about certificates. If you’re able to own the provisioning process, you can have a fully automated system where your cluster fetches short-lived certificates from a certificate authority (CA), installs them,and renews them when required. Let’s Encrypt is a great choice: it issues free certificates through an easily automated process. Traefik has native integration with Let’s Encrypt; for other ingress controllers, you can use the open source cert-manager tool (`https://cert-manager.io`), which is a CNCF project.

Not everyone is ready for an automated provisioning process, though. Some issuers require a human to download certificate files, or your organization may create certificate files from its own certificate authority for nonproduction domains. Then you’ll need to deploy the TLS certificate and key files as a Secret in the cluster. This scenario is common, so we’ll walk through it in the next exercise, generating a certificate of our own.

TRY IT NOW 
Run a Pod that generates a custom TLS certificate, and connect to the Pod to deploy the certificate files as a Secret. The Pod spec is configured to connect to the Kubernetes API server it’s running on.

   ```
   # run the Pod—this generates a certificate when it starts:
   kubectl apply -f ./cert-generator.yaml
   # connect to the Pod:
   kubectl exec -it deploy/cert-generator -- sh
   # inside the Pod, confirm the certificate files have been created:
   ls
   # rename the certificate files—Kubernetes requires specific names:
   mv server-cert.pem tls.crt
   mv server-key.pem tls.key
   # create and label a Secret from the certificate files:
   kubectl create secret tls kiamol-cert --key=tls.key --cert=tls.crt
   kubectl label secret/kiamol-cert kiamol=ch15
   # exit the Pod:
   exit
   # back on the host, confirm the Secret is there:
   kubectl get secret kiamol-cert --show-labels
   ```
That exercise simulates the situation where someone gives you a TLS certificate as a pair of PEM files, which you need to rename and use as the input to create a TLS Secret in Kubernetes. The certificate generation is all done using a tool called OpenSSL, and the only reason for running it inside a Pod is to package up the tool and the scripts to make it easy to use. 图 15.16 shows my output, with a Secret created in the cluster that can be used by an Ingress object.

![图15.16](./images/Figure15.16.png)
<center>图 15.16 If you’re given PEM files from a certificate issuer, you can create them as a TLS Secret. </center>

HTTPS support is simple with an ingress controller. You add a TLS section to your Ingress spec and state the name of the Secret to use— that’s it. Listing 15.7 shows an update to the Traefik ingress, which applies the new certificate to the `todo2.kiamol.local host`.

> Listing 15.7 ingress-traefik-https.yaml, using the standard Ingress HTTPS feature

```
spec:
rules:
- host: todo2.kiamol.local
http:
paths:
- path: /new
backend:
serviceName: todo-web-sticky
servicePort: 80
tls: # The TLS section switches on HTTPS
- secretName: kiamol-cert # using the certificate in this Secret.
```

The TLS field with the Secret name is all you need, and it’s portable across all ingress controllers. When you deploy the updated Ingress rules, the site will be served over HTTPS with your custom certificate. You’ll still get a security warning from the browser because the certificate authority is untrusted, but if your organization has its own CA, then it will be trusted by your machine and the organization’s certificates will be valid.

TRY IT NOW 
Update the to-do list Ingress objects to publish HTTPS using the Traefik ingress controller and your own TLS cert.

   ```
   # apply the Ingress update:
   kubectl apply -f todo-list/update/ingress-traefik-https.yaml
   # browse to https://todo2.kiamol.local:9443
   # there’s still a warning, but this time it’s because
   # the KIAMOL CA isn’t trusted
   ```

You can see my output in figure 15.17. I’ve opened the certificate details in one screen to confirm this is my own “kiamol” certificate. I accepted the warning in the second screen, and the to-do list traffic is now encrypted with the custom certificate. The

![图15.17](./images/Figure15.17.png)
<center>图 15.17 Ingress controllers can apply TLS certs from Kubernetes Secrets. If the certificate had come from a trusted issuer, the site would be secure </center>

script that generates the certificate sets it for all the kiamol.local domains we’ve used in this chapter, so the certificate is valid for the address, but it’s not from a trusted issuer.

We’ll switch back to Nginx for the final exercise—using the same certificate with the Nginx ingress controller, just to show that the process is identical. The updated Ingress specs use the same rules as the previous Nginx deployment, but now they add the TLS field with the same Secret name as listing 15.7.

TRY IT NOW 
Update the to-do Ingress rules for Nginx, so the app is available using HTTPS over the standard port 443, which the Nginx ingress controller is using.

   ```
   # update the Ingress resources:
   kubectl apply -f todo-list/update/ingress-https.yaml
   # browse to https://todo.kiamol.local
   # accept the warnings to view the site
   # confirm that the HTTP requests are redirected to HTTPS:
   curl http://todo.kiamol.local
   ```

I cheated when I ran that exercise and added the Kiamol CA to my trusted issuer list in the browser. You can see in figure 15.18 that the site is shown as secure, without any warnings, which is what you’d see for an organization’s own certificates. You can also see that the ingress controller redirects HTTP requests to HTTPS—the 308 redirect response in the curl command is taken care of by Nginx.

![图15.18](./images/Figure15.18.png)
<center>图 15.18 The TLS Ingress configuration works in the same way with the Nginx ingress controller. </center>

The HTTPS part of Ingress is solid and easy to use, and it’s good to head to the end of the chapter on a high note. But using an ingress controller features a lot of complexity, and in some cases, you’ll spend more time crafting your Ingress rules than you will modeling the deployment of the app.

## 15.5 理解 Ingress 及 Ingress 控制器

You’ll almost certainly run an ingress controller in your cluster, because it centralizes routing for domain names and moves TLS certificate management away from the applications. The Kubernetes model uses a common Ingress spec and a pluggable implementation that is very flexible, but the user experience is not straightforward. The Ingress spec records only the most basic routing details, and to use more advanced features from your proxy, you’ll need to add chunks of configuration as annotations.

Those annotations are not portable, and there is no interface specification for the features an ingress controller must support. There will be a migration project if you want to move from Nginx to Traefik or HAProxy or Contour (an open source project accepted into the CNCF on the very day I wrote this chapter), and you may find the features you need aren’t all available. The Kubernetes community is aware of the limitations of Ingress and is working on a long-term replacement called the Service API, but as of 2021, that’s still in the early stages.

That’s not to say that Ingress should be avoided—it’s the best option right now, and it’s likely to be the production choice for many years. It’s worth evaluating different ingress controllers and then settling on a single option. Kubernetes supports multiple ingress controllers, but the trouble will really start if you use different implementations and have to manage sets of Ingress rules with incompatible feature sets invoked through incomprehensible annotations. In this chapter, we looked at Nginx and Traefik, which are both good options, but there are plenty of others, including commercial options backed with support contracts.

We’re done with Ingress now, so we can tidy up the cluster to get ready for the lab.

TRY IT NOW 
Clear down the Ingress namespaces and the application resources.

   ```
   kubectl delete ns,all,secret,ingress -l kiamol=ch15
   ```

## 15.6 实验室

Here is a nice lab for you to do, following the pattern from chapters 13 and 14. Your job is to build the Ingress rules for the Astronomy Picture of the Day app. Simple . . .
+ Start by deploying the ingress controller in the lab/ingress-nginx folder.
+ The ingress controller is restricted to look for Ingress objects in one namespace, so you’ll need to figure out which one and deploy the lab/apod/ folder to that namespace.
+ The web app should be published at www.apod.local and the API at api.apod.local.
+ We want to prevent distributed denial-of-service attacks, so you should use the rate-limiting feature in the ingress controller to prevent too many requests from the same IP address.
+ The ingress controller uses a custom class name, so you’ll need to find that, too. This is partly about digging into the ingress controller configuration and partly about
the documentation for the controller—be aware that there are two Nginx ingress controllers. We’ve used the one from the Kubernetes project in this chapter, but there’s an alternative published by the Nginx project. My solution is ready for you to check against:
<https://github.com/sixeyed/kiamol/blob/master/ch15/lab/README.md>.