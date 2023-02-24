# 11 App development——Developer workflows and CI/CD

This is the final chapter on Kubernetes in the real world, and the focus here is the practicality of developing and delivering software to run on Kubernetes. Whether you identify as a developer or you’re on the ops side working with developers, the move to containers impacts the way you work, the tools you use, and the amount of time and effort from making a code change to seeing it running in development and test environments. In this chapter, we’ll examine how Kubernetes affects both the inner loop—the developer workflow on the local machine—and the outer loop—the CI/CD workflow that pushes changes to test and production.

How you use Kubernetes in your organization will be quite different from how you’ve used it so far in this book, because you’ll use shared resources like clusters and image registries. As we explore delivery workflows in this chapter, we’ll also cover lots of small details that can trip you up as you make the change to the real world—things like using private registries and maintaining isolation on a shared cluster. The main focus of the chapter is to help you understand the choice between a Docker-centric  workflow and something more like a Platform-as-a-Service (PaaS) running on Kubernetes.



## 11.1 The Docker developer workflow

Developers love Docker. It was voted the number one most-wanted platform and the number two “most loved” in Stack  Overflow’s annual survey two years in a row. Docker makes some parts of the developer workflow incredibly easy but at a cost: the Docker artifacts become central to the project, and that has an impact on the inner loop. You can run the app in a local environment using the same technologies as production but only if you accept a different way of working. If you’re not familiar with building apps using containers, appendix A in the ebook covers that in detail; it’s the chapter “Packaging Applications from Source Code into Docker Images” from Learn Docker in a Month of Lunches (Manning, 2020).

In this section, we’ll walk through the developer workflow where Docker and Kubernetes are used in every environment, and where developers have their own dedicated cluster. You’ll need to have Docker running if you want to follow along with the exercises, so if your lab environment is Docker Desktop or K3s, then you’re good to go. The first thing we’ll look at is developer onboarding—joining a new project and getting up to speed as quickly as possible.

TRY IT NOW
This chapter offers a whole new demo app—a simple bulletin board where you can post details of upcoming events. It’s written in Node.js, but you don’t need to have Node.js installed to get up and running with the Docker workflow.

```
# switch to this chapter’s folder in the source code:
cd ch11
# build the app:
docker-compose -f bulletin-board/docker-compose.yml build
# run the app:
docker-compose -f bulletin-board/docker-compose.yml up -d
# check the running containers:
docker ps
# browse to the app at http://localhost:8010/
```

This is just about the simplest way you can get started as a developer on a new project. The only software you need installed is Docker, and then you grab a copy of the code, and off you go. You can see my output in figure 11.1. I don’t have Node.js installed on my machine, and it doesn’t matter whether you do or what version you have; your results will be the same.

Behind the magic are two things: a Dockerfile, which has all the steps to build and package the Node.js component, and a Docker Compose file, which specifies all the components and the path to their Dockerfiles. There’s only one component in this app, but there could be a dozen—all using different technologies—and the workflow would be the same. But this isn’t how we’re going to run the app in production, so if we want to use the same technology stack, we can switch to running the app in Kubernetes locally, just using Docker for the build.



TRY IT NOW
Simple Kubernetes manifests for running the app using the local image are in the source folder. Remove the Compose version of the app, and deploy it to Kubernetes instead.

```
# stop the app in Compose:
docker-compose -f bulletin-board/docker-compose.yml down
```

![Figure 11.1](images/Figure11.1.png)

**Figure 11.1 Developer onboarding is a breeze with Docker and Compose—if there are no problems.**

```
# deploy in Kubernetes:
kubectl apply -f bulletin-board/kubernetes/
# get the new URL:
kubectl get svc bulletin-board -o jsonpath='http://{.status.loadBalancer.ingress[0].*}:8011'
# browse
```

This workflow is still pretty simple, although we now have three container artifacts to work with: the Dockerfile, the Compose file, and the Kubernetes manifest. I have my own Kubernetes cluster, and with that I can run the app exactly as it will run in production. My output in figure 11.2 shows it’s the same app, using the same local image, built with Docker Compose in the previous exercise.

![Figure 11.2](images/Figure11.2.png)

**Figure 11.2 You can mix Docker and Kubernetes using Compose to build images to run in Pods.**

Kubernetes is happy to use a local image that you’ve created or pulled with Docker, but you must follow some rules about whether it uses the local image or pulls it from a registry. If the image doesn’t have an explicit tag in the name (and uses the default :latest tag), then Kubernetes will always try to pull the image first. Otherwise, Kubernetes will use the local image if it exists in the image cache on the node. You can override the rules by specifying an image pull policy. Listing 11.1 shows the Pod spec for the bulletin board app, which includes an explicit policy.



> Listing 11.1 bb-deployment.yaml, specifying image pull policies

```
spec: # This is the Pod spec within the Deployment.
  containers:
    - name: bulletin-board
      image: kiamol/ch11-bulletin-board:dev
      imagePullPolicy: IfNotPresent # Prefer the local image if it exists
```

That’s the sort of detail that can be a nasty stumbling block in the developer workflow. The Pod spec might be configured so the registry image is preferred, and then you can rebuild your own local image as much as you like and never see any changes, because Kubernetes will always use the remote image. Similar complications exist around image versions, because an image can be replaced with another version using the same name and tag. That doesn’t play well with the Kubernetes desired state approach, because if you deploy an update with an unchanged Pod spec, nothing happens, even if the image contents have changed.

Back to our demo app. Your first task on the project is to add some more detail to the events list, which is an easy code change for you. Testing your change is more challenging, because you can repeat the Docker Compose command to rebuild the image,
but if you repeat the kubectl command to deploy the changes, you’ll see that nothing happens. If you’re into containers, you can do some investigation to understand the problem and delete the Pod to force a replacement, but if you’re not, then your work-
flow is already broken.

TRY IT NOW
You don’t really need to make a code change—a new file has the changes in it. Just replace the code file and rebuild the image, then delete the Pod to see the new app version running in the replacement Pod.

```
# remove the original code file:
rm bulletin-board/src/backend/events.js
# replace it with an updated version:
cp bulletin-board/src/backend/events-update.js bulletin-
board/src/backend/events.js
# rebuild the image using Compose:
docker-compose -f bulletin-board/docker-compose.yml build
# try to redeploy using kubectl:
kubectl apply -f bulletin-board/kubernetes/
# delete the existing Pod to recreate it:
kubectl delete pod -l app=bulletin-board
```

You can see my output in figure 11.3. The updated application is running in the screenshot, but only after the Pod was manually deleted and then recreated by the Deployment controller, using the latest image version. 

If you chose a Docker-centric workflow, then this is just one of the complications that will slow down and frustrate the development teams (debugging and making live app updates are the next ones they’ll hit). Container technologies are not easy topics to learn as you go—they really need some dedicated time to understand the principles, and not everyone on every team will want to make that investment.

The alternative is to centralize all the container technologies in a single team that provides a CI/CD pipeline that development teams can plug into to deploy their apps. The pipeline takes care of packaging container images and deploying to the cluster, so the development teams don’t need to bring Docker and Kubernetes into their own work.

![Figure 11.3](images/Figure11.3.png)

**Figure 11.3 Docker images are mutable, but renaming images doesn’t trigger an update in Kubernetes.**

## 11.2 The Kubernetes-as-a-Service developer workflow
A Platform-as-a-Service experience running on top of Kubernetes is an attractive option for a lot of organizations. You can run a single cluster for all your test environments that also hosts the CI/CD service to take care of the messy details about running in containers. All of the Docker artifacts are removed from the developer workflow so developers work on components directly, running Node.js and everything else they need on their machines, and they don’t use containers locally. 

This method moves containers to the outer loop—when developers push changes to source control, that triggers a build, which creates the container images, pushes them to a registry, and deploys the new version to a test environment in the cluster.
You get all the benefits of running in a container platform, without the friction containers bring to development. Figure 11.4 shows how that looks with one set of technology options.

The promise of this approach is that you get to run your app on Kubernetes without affecting the developer workflow or requiring every team member to skill up on Docker and Compose. It can work well in organizations where development teams
work on small components and a separate team assembles all the pieces into a working system, because only the assembly team needs the container skills. You can also remove Docker entirely, which is useful if your cluster uses a different container runtime. If you

![Figure 11.4](images/Figure11.4.png)

**Figure 11.4 Using containers in the outer loop lets developers focus on code.**

want to build container images without Docker, however, you need to replace it with a lot of other moving pieces. You’ll end up with more complexity overall, but it will be centralized in the delivery pipelines and not the projects.

We’ll walk through an example of that in this chapter, but to manage the complexity, we’ll do it in stages, starting with the view from inside the build service. To keep it simple, we’ll run our own Git server so we can push changes and trigger builds all from our lab cluster.

TRY IT NOW
Gogs is a simple but powerful Git server that is published as an image on Docker Hub. It’s a great way to run a private Git server in your organization or to quickly spin up a backup if your online service goes offline. Run Gogs in your cluster to push a local copy of the book’s source code.

```
# deploy the Git server:
kubectl apply -f infrastructure/gogs.yaml
# wait for it to spin up:
kubectl wait --for=condition=ContainersReady pod -l app=gogs
# add your local Git server to the book’s repository—
# this grabs the URL from the Service to use as the target:
git remote add gogs $(kubectl get svc gogs -o jsonpath=
'http://{.status.loadBalancer.ingress[0].*}:3000/kiamol/kiamol.git')
# push the code to your server—authenticate with
# username kiamol and password kiamol
git push gogs
# find the server URL:
kubectl get svc gogs -o
jsonpath='http://{.status.loadBalancer.ingress[0].*}:3000'
# browse and sign in with the same kiamol credentials
```

Figure 11.5 shows my output. You don’t need to run your own Git server for this workflow; it works in the same way using GitHub or any other source control system, but doing this makes for an easily reproducible environment—the Gogs setup for the chapter is preconfigured with a user account, so you can get up and running quickly.

![Figure 11.5](images/Figure11.5.png)

**Figure 11.5 Running your own Git server in Kubernetes is easy with Gogs.**

Now we have a local source control server into which we can plug the other components. Next is a system that can build container images. To make this portable so it runs on any cluster, we need something that doesn’t require Docker, because the cluster might use a different container runtime. We have a few options, but one of the best is BuildKit, an open source project from the Docker team. BuildKit started as a replacement for the image-building component inside the Docker Engine, and it has a pluggable architecture, so you can build images with or without Dockerfiles. You can run BuildKit as a server, so other components in the toolchain can use it to build images.

TRY IT NOW
Run BuildKit as a server inside the cluster, and confirm it has all the tools it needs to build container images without Docker.

```
# deploy BuildKit:
kubectl apply -f infrastructure/buildkitd.yaml
# wait for it to spin up:
kubectl wait --for=condition=ContainersReady pod -l app=buildkitd
# verify that Git and BuildKit are available:
kubectl exec deploy/buildkitd -- sh -c 'git version && buildctl
--version'
# check that Docker isn’t installed—this command will fail:
kubectl exec deploy/buildkitd -- sh -c 'docker version'
```

You can see my output in figure 11.6, where the BuildKit Pod is running from an image with BuildKit and the Git client installed but not Docker. It’s important to realize

![Figure 11.6](images/Figure11.6.png)

**Figure 11.6 BuildKit running as a container image-building service, without requiring Docker**

that BuildKit is completely standalone—it doesn’t connect to the container runtime in Kubernetes to build images; that’s all going to happen within the Pod.

We need to set up a few more pieces before we can see the full PaaS workflow, but we have enough in place now to see how the build part of it works. We’re targeting a Docker-free approach here, so we’re going to ignore the Dockerfile we used in the last
section and build the app into a container image directly from source code. How so? By using a CNCF project called Buildpacks, a technology pioneered by Heroku to power their PaaS product.

Buildpacks use the same concept as multistage Dockerfiles: running the build tools inside a container to compile the app and then packaging the compiled app on top of another container image that has the application runtime. You can do that with a
tool called Pack, which you run over the source code for your app. Pack works out what language you’re using, matches it to a Buildpack, and then packages your app into an image—no Dockerfile required. Right now Pack runs only with Docker, but we’re not using Docker, so we can use an alternative to integrate Buildpacks with BuildKit.

TRY IT NOW
We’re going to step inside the build process to manually run a build that we’ll go on to automate later in the chapter. Connect to the BuildKit Pod, pull the book’s code from your local Git server, and build it using Buildpacks instead of the Dockerfile.

```
# connect to a session on the BuildKit Pod:
kubectl exec -it deploy/buildkitd -- sh
# clone the source code from your Gogs server:
cd ~
git clone http://gogs:3000/kiamol/kiamol.git
# switch to the app directory:
cd kiamol/ch11/bulletin-board/
# build the app using BuildKit; the options tell BuildKit
# to use Buildpacks instead of a Dockerfile as input and to
# produce an image as the output:
buildctl build --frontend=gateway.v0 --opt source=kiamol/buildkit-
buildpacks --local context=src --output
type=image,name=kiamol/ch11-bulletin-board:buildkit
# leave the session when the build completes
exit
```

This exercise takes a while to run, but keep an eye on the output from BuildKit, and you’ll see what’s happening—first, it downloads the component that provides the Buildpacks integration, and then that runs and finds this is a Node.js app; it packages the app into a compressed archive and then exports the archive into a container image that has the Node.js runtime installed. My output is shown in figure 11.7.

You can’t run a container from that image on the BuildKit Pod because it doesn’t have a container runtime configured, but BuildKit is able to push images to a registry

![Figure 11.7](images/Figure11.7.png)

**Figure 11.7 Building container images without Docker and Dockerfiles adds a lot of complexity.**

after building, and that’s what we’ll do in the complete workflow. So far, we’ve seen that you can build and package your apps to run in containers without Dockerfiles or Docker, which is pretty impressive, but it comes at a cost.

The biggest issue is the complexity of the build process and the maturity of all the pieces. BuildKit is a stable tool, but it isn’t anywhere near as well used as the standard Docker build engine. Buildpacks are a promising approach, but the dependency
on Docker means they don’t work well in a Docker-free environment like a managed Kubernetes cluster in the cloud. The component we’re using to bridge them is a tool written by Tõnis Tiigi, a maintainer on the BuildKit project. It’s really just a proof of concept to plug Buildpacks into BuildKit; it works well enough to demonstrate the workflow, but it’s not something you would want to rely on to build apps for production.

There are alternatives. GitLab is a product that combines a Git server with a build pipeline that uses Buildpacks, and Jenkins X is a native build server for Kubernetes. They are complex products themselves, and you need to be aware that if you want to remove Docker from your developer workflow, you’ll be trading it for more complexity in the build process. You’ll be able to decide whether the result is worth it by the end of this chapter. Next we’ll look at how you can isolate workloads in Kubernetes, so a single cluster can run your delivery pipelines and all your test environments.

## 11.3 Isolating workloads with contexts and namespaces

Way back in chapter 3, I introduced Kubernetes namespaces—and very quickly moved on. You need to be aware of them to make sense of the fully qualified DNS names Kubernetes uses for Services, but you don’t need to use them until you start dividing up  your cluster. Namespaces are a grouping mechanism—every Kubernetes object belongs to a namespace—and you can use multiple namespaces to create virtual clusters from one real cluster.

Namespaces are very flexible, and organizations use them in different ways. You might use them in a production cluster to divide it up for different products or to divide up a nonproduction cluster for different environments—integration test, system test, and user testing. You might even have a development cluster where each developer has their own namespace, so they don’t need to run their own cluster. Namespaces are a boundary where you can apply security and resource restrictions, so deployment, but we’ll start with a simple walkthrough.

TRY IT NOW
Kubectl is namespace aware. You can explicitly create a namespace, and then deploy and query resources using the namespace flag—this creates a simple sleep Deployment.

```
# create a new namespace:
kubectl create namespace kiamol-ch11-test
# deploy a sleep Pod in the new namespace:
kubectl apply -f sleep.yaml --namespace kiamol-ch11-test
# list sleep Pods—this won’t return anything:
kubectl get pods -l app=sleep
# now list the Pods in the namespace:
kubectl get pods -l app=sleep -n kiamol-ch11-test
```

My output is shown in figure 11.8, where you can see that namespaces are an essential part of resource metadata. You need to explicitly specify the namespace to work with an object in kubectl. The only reason we’ve avoided this for the first 10 chapters is that every cluster has a namespace called default, which is used if you don’t specify a namespace, and that’s where we’ve created and used everything so far.

![Figure 11.8](images/Figure11.8.png)

**Figure 11.8 Namespaces isolate workloads—you can use them to represent different environments.**

Objects within a namespace are isolated, so you can deploy the same apps with the same object names in different namespaces. Resources can’t see resources in other namespaces. Kubernetes networking is flat, so Pods in different namespaces can communicate through Services, but a controller looks for Pods only in its own namespace. Namespaces are ordinary Kubernetes resources, too. Listing 11.2 shows a namespace spec in YAML, along with the metadata for another sleep Deployment that uses the new namespace.

> Listing 11.2 sleep-uat.yaml, a manifest that creates and targets a namespace

```
apiVersion: v1
kind: Namespace # Namespace specs need only a name.
metadata:
  name: kiamol-ch11-uat
---
apiVersion: apps/v1
kind: Deployment
metadata: # The target namespace is part of the
  name: sleep # object metadata. The namespace needs
  namespace: kiamol-ch11-uat # to exist, or the deployment fails.
# The Pod spec follows.
```

The Deployment and Pod specs in that YAML file use the same names as the objects you deployed in the previous exercise, but because the controller is set to use a different namespace, all the objects it creates will be in that namespace, too. When you deploy this manifest, you’ll see the new objects created without any naming collisions.

TRY IT NOW
Create a new UAT namespace and Deployment from the YAML in listing 11.2. The controller uses the same name, and you can see objects across namespaces using kubectl. Deleting a namespace deletes all its resources.

```
# create the namespace and Deployment:
kubectl apply -f sleep-uat.yaml
# list the sleep Deployments in all namespaces:
kubectl get deploy -l app=sleep --all-namespaces
# delete the new UAT namespace:
kubectl delete namespace kiamol-ch11-uat
# list Deployments again:
kubectl get deploy -l app=sleep --all-namespaces
```

You can see my output in figure 11.9. The original sleep Deployment didn’t specify a namespace in the YAML file, and we created it in the kiamol-ch11-test namespace by specifying that in the kubectl command. The second sleep Deployment specified the   kiamol-ch11-uat namespace in the YAML, so it was created there without needing a kubectl namespace flag.

![Figure 11.9](images/Figure11.9.png)

**Figure 11.9 Namespaces are a useful abstraction for managing groups of objects.**

In a shared cluster environment, you might regularly use different namespaces— deploying apps in your own development namespace and then looking at logs in the test namespace. Switching between them using kubectl flags is time consuming and
error prone, and kubectl provides an easier way with contexts. A context defines the connection details for a Kubernetes cluster and sets the default namespace to use in kubectl commands. Your lab environment will already have a context set up, and you
can modify that to switch namespaces.

TRY IT NOW
Show your configured contexts, and update the current one to set the default namespace to the test namespace.

```
# list all contexts:
kubectl config get-contexts
# update the default namespace for the current context:
kubectl config set-context --current --namespace=kiamol-ch11-test
# list the Pods in the default namespace:
kubectl get pods
```

You can see in figure 11.10 that setting the namespace for the context sets the default namespace for all kubectl commands. Any queries that don’t specify a namespace and any create commands where the YAML doesn’t specify a namespace will now all use the test namespace. You can create multiple contexts, all using the same cluster but different namespaces, and switch between them with the kubectl use-context command.

The other important use for contexts is to switch between clusters. When you set up Docker Desktop or K3s, they create a context for your local cluster—the details all live in a configuration file, which is stored in the .kube directory in your home folder. Managed

![Figure 11.10](images/Figure11.10.png)

**Figure 11.10 Contexts are an easy way to switch between namespaces and clusters.**

Kubernetes services usually have a feature to add a cluster to your config file, so you can work with remote clusters from your local machine. The remote API server will be secured using TLS, and your kubectl configuration will use a client certificate to identify you as the user. You can see those security details by viewing the configuration.

TRY IT NOW
Reset your context to use the default namespace, and then print out the details of the client configuration.

```
# setting the namespace to blank resets the default:
kubectl config set-context --current --namespace=
# printing out the config file shows your cluster connection:
kubectl config view
```

Figure 11.11 shows my output, with a local connection to my Docker Desktop cluster using TLS certificates—which aren’t shown by kubectl—to authenticate the connection.

![Figure 11.11](images/Figure11.11.png)

**Figure 11.11 Contexts contain the connection details for the cluster, which could be local or remote.**

Kubectl can also use a token to authenticate with the Kubernetes API server, and Pods are provided with a token they can use as a Secret, so apps running in Kubernetes can connect to the Kubernetes API to query or deploy objects. That’s a long way to getting where we want to go next: we’ll run a build server in a Pod that triggers a build when the source code changes in Git, builds the image using BuildKit, and deploys it to Kubernetes in the test namespace.

## 11.4 Continuous delivery in Kubernetes without Docker
Actually, we’re not quite there yet, because the build process needs to push the image to a registry, so Kubernetes can pull it to run Pod containers. Real clusters have multiple nodes, and each of them needs to be able to access the image registry. That’s been easy so far because we’ve used public images on Docker Hub, but in your own builds, you’ll push to a private repository first. Kubernetes supports pulling private images by storing registry credentials in a special type of Secret object.

You’ll need to have an account set up on an image registry to follow along with this section—Docker Hub is fine, or you can create a private registry on the cloud using Azure Container Registry (ACR) or Amazon Elastic Container Registry (ECR). If you’re running your cluster in the cloud, it makes sense to use that cloud’s registry to reduce download times, but all registries use the same API as Docker Hub, so they’re interchangeable.

TRY IT NOW
Create a Secret to store registry credentials. To make it easier to follow along, there’s a script to collect the credentials into local variables. Don’t worry—the scripts don’t email your credentials to me. . .

```
# collect the details—on Windows:
. .\set-registry-variables.ps1
# OR on Linux/Mac:
. ./set-registry-variables.sh
# create the Secret using the details from the script:
kubectl create secret docker-registry registry-creds --docker-
server=$REGISTRY_SERVER --docker-username=$REGISTRY_USER --docker-
password=$REGISTRY_PASSWORD
# show the Secret details:
kubectl get secret registry-creds
```

My output appears in figure 11.12. I’m using Docker Hub, which lets you create temporary access tokens that you can use in the same way as a password for your account. When I’m done with this chapter, I’ll revoke the access token—that’s a nice security
feature in Hub.

Okay, now we’re ready. We have a Docker-less build server running in the BuildKit Pod, a local Git server we can use to quickly iterate over the build process, and a registry Secret stored in the cluster. We can use all of those pieces with an automation

![Figure 11.12](images/Figure11.12.png)

**Figure 11.12 Your organization may use a private image registry—you need a Secret to authenticate.**

server to run the build pipeline, and we’ll be using Jenkins for that. Jenkins has a long legacy as a build server, and it’s very popular, but you don’t need to be a Jenkins guru to set up this build, because I have it already configured in a custom
Docker Hub image.

The Jenkins image for this chapter has the BuildKit and kubectl command lines installed, and the Pod is set up to surface credentials in the right places. The registry Secret you created in the previous exercise is mounted in the Pod container, so  BuildKit can use it to authenticate to the registry when it pushes the image. Kubectl is configured to connect to the local API server in the cluster using the token Kubernetes provides in another Secret. Deploy the Jenkins server, and check that everything is correctly configured.

TRY IT NOW
Jenkins gets everything it needs from Kubernetes Secrets, using a startup script in the container image. Start by deploying Jenkins and confirming it can connect to Kubernetes.

```
# deploy Jenkins:
kubectl apply -f infrastructure/jenkins.yaml
# wait for the Pod to spin up:
kubectl wait --for=condition=ContainersReady pod -l app=jenkins
# check that kubectl can connect to the cluster:
kubectl exec deploy/jenkins -- sh -c 'kubectl version --short'
# check that the registry Secret is mounted:
kubectl exec deploy/jenkins -- sh -c 'ls -l /root/.docker'
```

In this exercise, you’ll see kubectl report the version of your own Kubernetes lab cluster—that confirms the Jenkins Pod container is set up correctly to authenticate to Kubernetes, so it can deploy applications to the same cluster where it is running. My output is shown in figure 11.13.

![Figure 11.13](images/Figure11.13.png) 

**Figure 11.13 Jenkins runs the pipeline, so it needs authentication details for Kubernetes and the registry.**

Everything is in place now for Jenkins to fetch application code from the Gogs Git server, connect to the BuildKit server to build the container image using Buildpacks and push it to the registry, and deploy the latest application version to the test namespace. That work is already set up using a Jenkins pipeline, but the pipeline steps just use simple build scripts in the application folder. Listing 11.3 shows the build stage, which packages and pushes the image.

> Listing 11.3 build.sh, the build script using BuildKit

```
buildctl --addr tcp://buildkitd:1234 \ # The command runs on Jenkins,
  build \ # but it uses the BuildKit server.
  --frontend=gateway.v0 \
  --opt source=kiamol/buildkit-buildpacks \ # Uses Buildpacks as input
  --local context=src \
  --output type=image,name=${REGISTRY_SERVER}/${REGISTRY_USER}/bulletin-board:
     ${BUILD_NUMBER}-kiamol,push=true # Pushes the output to the registry
```

The script is an extension of the simpler BuildKit command you ran in section 11.2, when you were pretending to be the build server. The buildctl command uses the same integration component for Buildpacks, so there’s no Dockerfile in here. This command runs inside the Jenkins Pod, so it specifies an address for the BuildKit server, which is running in a separate Pod behind the Service called buildkitd. No Docker here, either. The variables in the image name are all set by Jenkins, but they’re standard
environment variables, so there’s no dependency on Jenkins in the build scripts.

When this stage of the pipeline completes, the image will have been built and pushed to the registry. The next stage is to deploy the updated application, which is in a separate script, shown in listing 11.4. You don’t need to run this yourself—it’s all in the Jenkins pipeline.

> Listing 11.4 run.sh, the deployment script using Helm

```
helm upgrade --install --atomic \ # Upgrades or installs the release
  --set registryServer=${REGISTRY_SERVER}, \ # Sets the values for the
        registryUser=${REGISTRY_USER}, \ # image tag, referencing
        imageBuildNumber=${BUILD_NUMBER} \ # the new image version
--namespace kiamol-ch11-test \ # Deploys to the test namespace
bulletin-board \
helm/bulletin-board
```

The deployment uses Helm with a chart that has values for the parts of the image name. They’re set from the same variables used in the build stage, which are compiled from the Docker registry Secret and the build number in Jenkins. In my case, the first
build pushes an image to Docker Hub named sixeyed/bulletin-board:1-kiamol and installs a Helm release using that image. To run the build in your cluster and push to your registry, you just need to log in to Jenkins and enable the build—the pipeline itself is already set up.

TRY IT NOW
Jenkins is running and configured, but the pipeline job isn’t enabled. Log in to enable the job, and you will see the pipeline execute and the app deployed to the cluster.

```
# get the URL for Jenkins:
kubectl get svc jenkins -o
jsonpath='http://{.status.loadBalancer.ingress[0].*}:8080/job/kiamol'
# browse and login with username kiamol and password kiamol;
# if Jenkins is still setting itself up you’ll see a wait screen
# click enable for the Kiamol job and wait . . .
# when the pipeline completes, check the deployment:
kubectl get pods -n kiamol-ch11-test -l
app.kubernetes.io/name=bulletin-board -o=custom-
columns=NAME:.metadata.name,IMAGE:.spec.containers[0].image
# find the URL of the test app:
kubectl get svc -n kiamol-ch11-test bulletin-board -o
jsonpath='http://{.status.loadBalancer.ingress[0].*}:8012'
# browse
```

The build should be fast because it’s using the same BuildKit server that has already cached the images for the Buildpack build from section 11.2. When the build has completed, you can browse to the application deployed by Helm in the test namespace and see the app running—mine is shown in figure 11.14.

![Figure 11.14](images/Figure11.14.png)

**Figure 11.14 The pipeline in action, built and deployed to Kubernetes without Docker or Dockerfiles**

So far so good. We’re playing the ops role, so we understand all the moving parts in the delivery of this app—we would own the pipeline in the Jenkinsfile and the application specs in the Helm chart. Lots of small fiddly details are in there, like the templated image name and the image pull Secret in the Deployment YAML, but from the developer’s point of view, that’s all hidden.

The developer’s view is that you can work on the app using your local environment, push changes, and see them running at the test URL, without worrying what happens in between. We can see that workflow now. You made an application change earlier to add event descriptions to the site, and to deploy that, all you need to do is push the changes to your local Git server and wait for the Jenkins build to complete.

TRY IT NOW
Push your code change to your Gogs server; Jenkins will see the change within one minute and start a new build. That will push a new image version to your registry and update the Helm release to use that version.

```
# add your code change, and push it to Git:
git add bulletin-board/src/backend/events.js
git commit -m 'Add event descriptions'
git push gogs
# browse back to Jenkins, and wait for the new build to finish
# check that the application Pod is using the new image version:
kubectl get pods -n kiamol-ch11-test -l
app.kubernetes.io/name=bulletin-board -o=custom-
columns=NAME:.metadata.name,IMAGE:.spec.containers[0].image
# browse back to the app
```

This is the git push PaaS workflow applied to Kubernetes. We’re dealing with a simple app here, but the approach is the same for a large system with many components: a shared namespace could be the deployment target for all the latest versions, pushed by many different teams. Figure 11.15 shows an application update in Kubernetes triggered from a push of code, with no  requirement for developers to use Docker, Kubernetes, or Helm.

Of course, the PaaS approach and the Docker approach are not mutually exclusive. If your cluster is running on Docker, you can take advantage of a simpler build process for Docker-based apps but still support a Docker-free PaaS approach for other apps, all in the same cluster. Each approach offers benefits and drawbacks, and we’ll end by looking at how you should choose between them.

## 11.5 Evaluating developer workflows on Kubernetes

In this chapter, we’ve looked at developer workflows at extreme ends of the spectrum, from teams who fully embrace containers and want to make them front and center in every environment, to teams who don’t want to add any ceremony to their development process, want to keep working natively, and leave all the container bits to the CI/CD pipeline. There are plenty of places in between, and the likelihood is that you’ll build an approach to suit your organization, your application architectures, and
your Kubernetes platform.

![Figure 11.15](images/Figure11.15.png)

**Figure 11.15 It’s PaaS on your own Kubernetes cluster—a lot of complexity is hidden from the developer.**

The decision is as much about culture as about technology. Do you want every team to level up on container knowledge, or do you want to centralize that knowledge in a service team and leave the developer teams to focus on delivering software? Although I’d love to see copies of Learn Docker in a Month of Lunches and Learn Kubernetes in a Month of Lunches on every desk, skilling up on containers does require a pretty big commitment. Here are the major advantages I see in keeping Docker and Kubernetes visible in your projects:

* The PaaS approach is complicated and bespoke—you’ll be plugging together lots of different technologies with different maturity levels and support structures.

* The Docker approach is flexible—you can add any dependencies and setup you need in a Dockerfile, whereas PaaS approaches are more prescriptive, so they won’t fit every app.

* PaaS technologies don’t have the optimizations you can get when you fine-tune your Docker images; the bulletin board image from the Docker workflow is 95 MB compared to 1 GB for the Buildpacks version—that’s a much smaller surface area to secure.
* The commitment to learning Docker and Kubernetes pays off because they’re portable skills—developers can easily move between projects using a standard toolset.
* Teams don’t have to use the full container stack; they can opt out at different stages—some developers might just use Docker to run containers, whereas others might use Docker Compose and others Kubernetes.
* Distributed knowledge makes for a better collaborative culture—centralized service teams might be resented for being the only ones who get to play with all the fun technology.

Ultimately, it’s a decision for your organization and teams, and the pain of migrating from the current workflow to the desired workflow needs to be considered. In my own consulting work, I’m often balancing development and operations roles, and I tend to be pragmatic. When I’m actively developing, I use native tooling (I typically work on .NET projects using Visual Studio), but before I push any changes, I run the CI process locally to build container images with Docker Compose and then spin everything
up in my local Kubernetes cluster. That won’t fit every scenario, but I find it a good balance between development speed and  confidence that my changes will work the same way in the next environment.

That’s all for the developer workflow, so we can tidy up the cluster before we move on. Leave your build components running (Gogs, BuildKit, and Jenkins)—you’ll need them for the lab.

TRY IT NOW
Remove the bulletin board deployments.

```
# uninstall the Helm release from the pipeline:
helm -n kiamol-ch11-test uninstall bulletin-board
# delete the manual deployment:
kubectl delete all -l app=bulletin-board
```

## 11.6 Lab
This lab is a bit nasty, so I’ll apologize in advance—but I want you to see that going down the PaaS path with a custom set of tools has danger in store. The bulletin board app for this chapter used a very old version of the Node runtime, version 10.5.0, and in
the lab, that needs updating to a more recent version. There’s a new source code folder for the lab that uses Node 10.6.0, and your job is to set up a pipeline to build that version, and then find out why it fails and fix it. There are a few hints that follow because the goal isn’t for you to learn Jenkins but to see how to debug failing pipelines:

- Start by creating a new item from the Jenkins home page: choose the option to copy an existing job, and copy the kiamol job; call the new job anything you like.
- In the new job configuration in the Pipeline tab, change the path to the pipeline file to the new source code folder: ch11/lab/bulletin-board/Jenkinsfile.
- Build the job, and look through the logs to find out why it failed.
- You’ll need to make a change in the lab source folder and push it to Gogs to fix the build


My sample solution is on GitHub with some screenshots for the Jenkins setup to help you: https://github.com/sixeyed/kiamol/blob/master/ch11/lab/README.md.