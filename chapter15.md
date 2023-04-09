# 第十五章 使用 Ingress 管理流入流量

Services bring network traffic into Kubernetes, and you can have multiple LoadBalancer Services with different public IP addresses to make your web apps available to the world. Doing this creates a management headache because it means allocating a new IP address for every application and mapping addresses to apps with your DNS provider. Getting traffic to the right app is a routing problem, but you can manage it inside Kubernetes instead, using Ingress. Ingress uses a set of rules to map domain names and request paths to applications, so you can have a single IP address for your whole cluster and route all traffic internally.

Routing by domain name is an old problem that has usually been solved with a reverse proxy, and Kubernetes uses a pluggable architecture for Ingress. You define the routing rules as standard resources and deploy your choice of reverse proxy to receive traffic and act on the rules. All the major reverse proxies have Kubernetes support, along with a new species of container-aware reverse proxy. They all have different capabilities and working models, and in this chapter, you’ll learn how you can use Ingress to host multiple apps in your cluster with two of the most popular:Nginx and Traefik.

## 15.1 Kubernetes 如何使用 Ingress 路由流量

We’ve used Nginx as a reverse proxy several times in this book already (17, by my count), but we’ve always used it for one application at a time. We had a reverse proxy to cache responses from the Pi app in chapter 6 and another for the randomnumber API in chapter 13. Ingress moves the reverse proxy into a central role, running it as a component called the ingress controller, but the approach is the same: the proxy receives external traffic from a LoadBalancer Service, and it fetches content from apps using ClusterIP Services. 图 15.1 shows the architecture.

![图15.1](./images/Figure15.1.png)
<center>图 15.1 Ingress controllers are the entry point to the cluster, routing traffic based on Ingress rules. </center>

The important thing in this diagram is the ingress controller, which is the pluggable reverse proxy—it could be one of a dozen options including Nginx, HAProxy, Contour, and Traefik. The Ingress object stores routing rules in a generic way, and the controller feeds those rules into the proxy. Proxies have different feature sets, and the Ingress spec doesn’t attempt to model every possible option, so controllers add support for those features using annotations. You’ll learn in this chapter that the core functionality of routing and HTTPS support is simple to work with, but the complexity is in the ingress controller deployment and its additional features.

We’ll start by running the basic Hello, World web app from way back in chapter 2, keeping it as an internal component with a ClusterIP Service and using the Nginx ingress controller to route traffic.

TRY IT NOW
Run the Hello, World app, and confirm that it’s accessible only inside the cluster or externally using a port-forward in kubectl.

   ```
   # switch to this chapter’s folder:
   cd ch15
   # deploy the web app:
   kubectl apply -f hello-kiamol/
   # confirm the Service is internal to the cluster:
   kubectl get svc hello-kiamol
   # start a port-forward to the app:
   kubectl port-forward svc/hello-kiamol 8015:80
   # browse to http://localhost:8015
   # then press Ctrl-C/Cmd-C to exit the port-forward
   ```

There’s nothing new in the Deployment or Service specs for that application—no special labels or annotations, no new fields you haven’t already worked with. You can see in figure 15.2 that the Service has no external IP address, and I can access the app only while I have a port-forward running.

![图15.2](./images/Figure15.2.png)
<center>ClusterIP Services make an app available internally—it can go public with Ingress. </center>

To make the app available using Ingress rules, we need an ingress controller. Controllers manage other objects. You know that Deployments manage ReplicaSets and ReplicaSets manage Pods. Ingress controllers are slightly different; they run in standard Pods and monitor Ingress objects. When they see any changes, they update the rules in the proxy We’ll start with the Nginx ingress controller, which is part of the wider Kubernetes project. There’s a production-ready Helm chart for the controller, but I’m using a much simpler deployment. Even so, there are a few security components in the manifest that we haven’t covered yet, but I won’t go through them now. (There are comments in the YAML if you want to investigate.)

TRY IT NOW
Deploy the Nginx ingress controller. This uses the standard HTTP and HTTPS ports in the Service, so you need to have ports 80 and 443 available on your machine.

   ```
   # create the Deployment and Service for the Nginx ingress controller:
   kubectl apply -f ingress-nginx/
   # confirm the service is publicly available:
   kubectl get svc -n kiamol-ingress-nginx
   # get the URL for the proxy:
   kubectl get svc ingress-nginx-controller -o jsonpath='http://{.status
   .loadBalancer.ingress[0].*}' -n kiamol-ingress-nginx
   # browse to the URL—you’ll get an error
   ```

When you run this exercise, you’ll see a 404 error page when you browse. That proves the Service is receiving traffic and directing it to the ingress controller, but there aren’t any routing rules yet so Nginx has no content to show, and it returns the default not-found page. My output is shown in figure 15.3, where you can see the Service is using the standard HTTP port.

![图15.3](./images/Figure15.3.png)
<center>图 15.3 Ingress controllers receive incoming traffic, but they need routing rules to know what to do with it. </center>

Now that we have an application running and an ingress controller, we just need to deploy an Ingress object with routing rules to tell the controller which application Service to use for each incoming request. Listing 15.1 shows the simplest rule for an Ingress object, which will route every request coming into the cluster to the Hello, World application.

> Listing 15.1 localhost.yaml, a routing rule for the Hello, World app

```
apiVersion: networking.k8s.io/v1beta1 # Beta API versions mean the spec
kind: Ingress # isn’t final and could change.
metadata:
name: hello-kiamol
spec:
rules:
- http: # Ingress is only for HTTP/S traffic
paths:
- path: / # Maps every incoming request
backend: # to the hello-kiamol Service
serviceName: hello-kiamol
servicePort: 80
```

The ingress controller is watching for new and changed Ingress objects, so when you deploy any, it will add the rules to the Nginx configuration. In Nginx terms, it will set up a proxy server where the hello-kiamol Service is the upstream—the source of the content—and it will serve that content for incoming requests to the root path.

TRY IT NOW 
Create the Ingress rule that publishes the Hello, World app through the ingress controller.

   ```
   # deploy the rule:
   kubectl apply -f hello-kiamol/ingress/localhost.yaml
   # confirm the Ingress object is created:
   kubectl get ingress
   # refresh your browser from the previous exercise
   ```
Well, that was simple—map a path to the backend Service for the application in an Ingress object, and the controller takes care of everything else. My output in figure 15.4 shows the localhost address, which previously returned a 404 error, now returns the Hello, World app in all its glory.

![图15.4](./images/Figure15.4.png)
<center>图 15.4 Ingress object rules link the Ingress controller to the app Service. </center>

Ingress is usually a centralized service in the cluster, like logging and monitoring. An admin team might deploy and manage the Ingress controller, whereas each product team owns the Ingress objects that route traffic to their apps. This process creates the potential for collisions—Ingress rules do not have to be unique, and one team’s update could end up redirecting all of another team’s traffic to some other app. Inpractice that doesn’t happen because those apps will be hosted at different domains, and the Ingress rules will include a domain name to restrict their scope.

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