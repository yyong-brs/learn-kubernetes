# 第一章 开始之前

Kubernetes is big. Really big. It was released as an open source project on GitHub
in 2014, and now it averages 200 changes every week from a worldwide community
of 2,500 contributors. The annual KubeCon conference has grown from 1,000
attendees in 2016 to more than 12,000 at the most recent event, and it’s now a
global series with events in America, Europe, and Asia. All the major cloud services
offer a managed Kubernetes service, and you can run Kubernetes in a data center
or on your laptop—and they’re all the same Kubernetes.

Independence and standardization are the main reasons Kubernetes is so popu-
lar. Once you have your apps running nicely in Kubernetes, you can deploy them
anywhere, which is attractive for organizations moving to the cloud, because it
enables them to move between data centers and other clouds without a rewrite. It’s
also very attractive for practitioners—once you’ve mastered Kubernetes, you can
move between projects and organizations and be productive quickly.

Getting to that point is hard, though, because Kubernetes is hard. Even simple
apps are deployed as multiple components, described in a custom file format that
can easily span many hundreds of lines. Kubernetes brings infrastructure-level concerns like load balancing, networking, storage, and compute into app configuration,
which might be new concepts, depending on your IT background. In addition, Kuber-
netes is always expanding—new releases come out every quarter, often bringing a
ton of new functionality.

But it’s worth it. I’ve spent many years helping people learn Kubernetes, and a
common pattern arises: the question “Why is this so complicated?” transforms to “You
can do that? This is amazing!” Kubernetes truly is an amazing piece of technology.
The more you learn about it, the more you’ll love it—and this book will accelerate you
on your journey to Kubernetes mastery.

## 1.1 了解 Kubernetes

This book provides a hands-on introduction to Kubernetes. Every chapter offers try-it-
now exercises and labs for you to get lots of experience using Kubernetes. All except
this one. :) We’ll jump into the practical work in the next chapter, but we need a little
theory first. Let’s start by understanding what Kubernetes actually is and the problems
it solves.
Kubernetes is a platform for running containers. It takes care of starting your con-
tainerized applications, rolling out updates, maintaining service levels, scaling to meet
demand, securing access, and much more. The two core concepts in Kubernetes are
the API, which you use to define your applications, and the cluster, which runs your
applications. A cluster is a set of individual servers that have all been configured with a
container runtime like Docker, and then joined into a single logical unit with Kuber-
netes. Figure 1.1 shows a high-level view of the cluster.

![图1.1](./images/Figure1.1.png)
<center>图1.1 Kubernetes 集群包含了一组服务器，它们加入到一个组中运行容器 </center>

Cluster administrators manage the individual servers, called nodes in Kubernetes. You
can add nodes to expand the capacity of the cluster, take nodes offline for servicing, or
roll out an upgrade of Kubernetes across the cluster. In a managed service like Microsoft
Azure Kubernetes Service (AKS) or Amazon Elastic Kubernetes Service (EKS), those
functions are all wrapped in simple web interfaces or command lines. In normal usage
you forget about the underlying nodes and treat the cluster as a single entity.

The Kubernetes cluster is there to run your applications. You define your apps in
YAML files and send those files to the Kubernetes API. Kubernetes looks at what
you’re asking for in the YAML and compares it to what’s already running in the clus-
ter. It makes any changes it needs to get to the desired state, which could be updating
a configuration, removing containers, or creating new containers. Containers are dis-
tributed around the cluster for high availability, and they can all communicate over
virtual networks managed by Kubernetes. Figure 1.2 shows the deployment process,
but without the nodes because we don’t really care about them at this level.

![图1.2](./images/Figure1.2.png)
<center>图1.2 当您将应用程序部署到 Kubernetes 集群时，通常可以忽略实际节点 </center>

Defining the structure of the application is your job, but running and managing everything is down to Kubernetes. If a node in the cluster goes offline and takes some containers with it, Kubernetes sees that and starts replacement containers on other nodes.
If an application container becomes unhealthy, Kubernetes can restart it. If a component is under stress because of a high load, Kubernetes can start extra copies of the
component in new containers. If you put the work into your Docker images and
Kubernetes YAML files, you’ll get a self-healing app that runs in the same way on any
Kubernetes cluster.

Kubernetes manages more than just containers, which is what makes it a complete
application platform. The cluster has a distributed database, and you can use that to
store both configuration files for your applications and secrets like API keys and connection credentials. Kubernetes delivers these seamlessly to your containers, which
lets you use the same container images in every environment and apply the correct
configuration from the cluster. Kubernetes also provides storage, so your applications
can maintain data outside of containers, giving you high availability for stateful apps.
Kubernetes also manages network traffic coming into the cluster by sending it to the
right containers for processing. Figure 1.3 shows those other resources, which are the
main features of Kubernetes.

![图1.3](./images/Figure1.3.png)
<center>图1.3 Kubernetes 不仅仅管理容器，集群还管理其他资源 </center>

I haven’t talked about what those applications in the containers look like; that’s
because Kubernetes doesn’t really care. You can run a new application built with
cloud-native design across microservices in multiple containers. You can run a legacy
application built as a monolith in one big container. They could be Linux apps or
Windows apps. You define all types of applications in YAML files using the same API,
and you can run them all on a single cluster. The joy of working with Kubernetes is that it adds a layer of consistency on top of all your apps—old .NET and Java mono-
liths and new Node.js and Go microservices are all described, deployed, and managed
in the same way.

That’s just about all the theory we need to get started with Kubernetes, but before
we go any further, I want to put some proper names on the concepts I’ve been talking
about. Those YAML files are properly called application manifests, because they’re a list
of all the components that go into shipping the app. Those components are Kuberne-
tes resources; they have proper names, too. Figure 1.4 takes the concepts from figure 1.3
and applies the correct Kubernetes resource names.

![图1.4](./images/Figure1.4.png)
<center>图1.4 真实情况：这些是您需要掌握的最基本的 Kubernetes 资源 </center>

I told you Kubernetes was hard. :) But we will cover all of these resources one at a time
over the next few chapters, layering on the understanding. By the time you’ve finished
chapter 6, that diagram will make complete sense, and you’ll have had lots of experience in defining those resources in YAML files and running them in your own Kuber-
netes cluster.

## 1.2 这本书适合你吗?

The goal of this book is to fast-track your Kubernetes learning to the point where you
have confidence defining and running your own apps in Kubernetes, and you under-
stand what the path to production looks like. The best way to learn Kubernetes is to
practice, and if you follow all the examples in the chapters and work through the labs,
then you’ll have a solid understanding of all the most important pieces of Kubernetes
by the time you finish the book.

But Kubernetes is a huge topic, and I won’t be covering everything. The biggest
gaps are in administration. I won’t cover cluster setup and management in any depth,
because they vary across different infrastructures. If you’re planning on running
Kubernetes in the cloud as your production environment, then a lot of those con-
cerns are taken care of in a managed service anyway. If you want to get a Kubernetes
certification, this book is a great place to start, but it won’t get you all the way. There
are two main Kubernetes certifications: Certified Kubernetes Application Developer
(CKAD) and Certified Kubernetes Administrator (CKA). This book covers about 80%
of the CKAD curriculum and about 50% of CKA.

There’s also a reasonable amount of background knowledge you’ll need to work
with this book effectively. I’ll explain lots of core principles as we encounter them in
Kubernetes features, but I won’t fill in any gaps about containers. If you’re not familiar with ideas like images, containers, and registries, I’d recommend starting with my
book, Learn Docker in a Month of Lunches (Manning, 2020). You don’t need to use
Docker with Kubernetes, but it is the easiest and most flexible way to package your
apps so you can run them in containers with Kubernetes.

If you classify yourself as a new or improving Kubernetes user, with a reasonable
working knowledge of containers, then this is the book for you. Your background
could be in development, operations, architecture, DevOps, or site reliability engineering (SRE)—Kubernetes touches all those roles, so they’re all welcome here, and
you are going to learn an absolute ton of stuff.

## 1.3 创建你的实验环境

A Kubernetes cluster can have hundreds of nodes, but for the exercises in this book, a
single-node cluster is fine. We’ll get your lab environment set up now so you’re ready
to get started in the next chapter. Dozens of Kubernetes platforms are available, and
the exercises in this book should work with any certified Kubernetes setup. I’ll describe
how to create your lab on Linux, Windows, Mac, Amazon Web Services (AWS), and
Azure, which covers all the major options. I’m using Kubernetes version 1.18, but earlier or later versions should be fine, too.

The easiest option to run Kubernetes locally is Docker Desktop, which is a single
package that gives you Docker and Kubernetes and all the command-line tools. It
also integrates nicely with your computer’s network and has a handy Reset Kuber-
netes button, which clears everything, if necessary. Docker Desktop is supported on Windows 10 and macOS, and if that doesn’t work for you, I’ll also walk through
some alternatives.

One point you should know: the components of Kubernetes itself need to run as
Linux containers. You can’t run Kubernetes in Windows (although you can run Win-
dows apps in containers with a multinode Kubernetes cluster), so you’ll need a Linux
virtual machine (VM) if you’re working on Windows. Docker Desktop sets that up and
manages it for you.

And one last note for Windows users: please use PowerShell to follow along with
the exercises. PowerShell supports many Linux commands, and the try-it-now exer-
cises are built to run on Linux (and Mac) shells and PowerShell. If you try to use the
classic Windows command terminal, you’re going to run into issues from the start.

### 1.3.1 下载本书源码

Every example and exercise is in the book’s source code repository on GitHub,
together with sample solutions for all of the labs. If you’re comfortable with Git and
you have a Git client installed, you can clone the repository onto your computer with
the following command:

`git clone https://github.com/sixeyed/kiamol`

If you’re not a Git user, you can browse to the GitHub page for the book at https://
github.com/sixeyed/kiamol and click the Clone or Download button to download a
zip file, which you can expand.

The root of the source code is a folder called kiamol, and within that is a folder for
each chapter: ch02, ch03, and so on. The first exercise in the chapter usually asks you
to open a terminal session and switch to the chXX directory, so you’ll need to navigate
to your kiamol folder first.

The GitHub repository is the quickest way for me to publish any corrections to the
exercises, so if you do have any problems, you should check for a README file with
updates in the chapter folder.

### 1.3.2 安装 Docker Desktop

Docker Desktop runs on Windows 10 or macOS Sierra (version 10.12 or higher).
Browse to https://www.docker.com/products/docker-desktop and choose to install
the stable version. Download the installer and run it, accepting all the defaults. On
Windows, that might include a reboot to add new Windows features. When Docker
Desktop is running, you’ll see Docker’s whale icon near the clock on the Windows
taskbar or the Mac menu bar. If you’re an experienced Docker Desktop user on Win-
dows, you’ll need to make sure you’re in Linux container mode (which is the default
for new installations).

Kubernetes isn’t set up by default, so you’ll need to click the whale icon to open
the menu and click Settings. That opens the window shown in figure 1.5; select Kubernetes from the menu and select Enable Kubernetes.

![图1.5](./images/Figure1.5.png)
<center>图1.5 Docker Desktop 创建了一个 Linux 虚拟机去运行容器，同时可以运行 Kubernetes </center>

Docker Desktop downloads all the container images for the Kubernetes runtime—
which might take a while—and then starts up everything. When you see two green
dots at the bottom of the Settings screen, your Kubernetes cluster is ready to go.
Docker Desktop installs everything else you need, so you can skip to section 1.4.7.

Other Kubernetes distributions can run on top of Docker Desktop, but they don’t
integrate well with the network setup that Docker Desktop uses, so you’ll encounter
problems running the exercises. The Kubernetes option in Docker Desktop has all
the features you need for this book and is definitely the easiest option.

### 1.3.3 安装 Docker 社区版本以及 K3s

If you’re using a Linux machine or a Linux VM, you have several options for running
a single-node cluster. Kind and minikube are popular, but my preference is K3s, which
is a minimal installation but has all the features you’ll need for the exercises. (The
name is a play on “K8s,” which is an abbreviation of Kubernetes. K3s trims the Kubernetes codebase, and the name indicates that it’s half the size of K8s.)

K3s works with Docker, so first, you should install Docker Community Edition. You
can check the full installation steps at https://rancher.com/docs/k3s/latest/en/
quick-start/, but this will get you up and running:

```
# install Docker:
curl -fsSL https://get.docker.com | sh
# install K3s:
curl -sfL https://get.k3s.io | sh -s - --docker --disable=traefik --write-
   kubeconfig-mode=644
```

If you prefer to run your lab environment in a VM and you’re familiar with using
Vagrant to manage VMs, you can use the following Vagrant setup with Docker and K3s
found in the source repository for the book:

```
# from the root of the Kiamol repo:
cd ch01/vagrant-k3s
# provision the machine:
vagrant up
# and connect:
vagrant ssh
```

K3s installs everything else you need, so you can skip to section 1.4.7.

### 1.3.4 安装 Kubernetes 命令行工具

You manage Kubernetes with a tool called kubectl (which is pronounced “cube-cuttle”
as in “cuttlefish”—don’t let anyone tell you different). It connects to a Kubernetes
cluster and works with the Kubernetes API. Both Docker Desktop and K3s install
kubectl for you, but if you’re using one of the other options described below, you’ll
need to install it yourself.

The full installation instructions are at https://kubernetes.io/docs/tasks/tools/
install-kubectl/. You can use Homebrew on macOS and Chocolatey on Windows, and
for Linux you can download the binary:

```
# macOS:
brew install kubernetes-cli
# OR Windows:
choco install kubernetes-cli
# OR Linux:
curl -Lo ./kubectl https://storage.googleapis.com/kubernetes-
   release/release/v1.18.8/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```

### 1.3.5 在 Azure 中运行一个单节点的 Kubernetes

You can run a managed Kubernetes cluster in Microsoft Azure using AKS. This might
be a good option if you want to access your cluster from multiple machines or if you
have an MSDN subscription with Azure credits. You can run a minimal single-node
cluster, which won’t cost a huge amount, but bear in mind that there’s no way to stop
the cluster and you’ll be paying for it 24/7 until you remove it.

The Azure portal has a nice user interface for creating an AKS cluster, but it’s
much easier to use the az command. You can check the latest docs at https://docs
.microsoft.com/en-us/azure/aks/kubernetes-walkthrough, but you can get started by
downloading the az command-line tool and running a few commands, as follows:

```
# log in to your Azure subscription:
az login
# create a resource group for the cluster:
az group create --name kiamol --location eastus
# create a single-code cluster with 2 CPU cores and 8GB RAM:
az aks create --resource-group kiamol --name kiamol-aks --node-count 1 --
node-vm-size Standard_DS2_v2 --kubernetes-version 1.18.8 --generate-ssh-keys
# download certificates to use the cluster with kubectl:
az aks get-credentials --resource-group kiamol --name kiamol-aks
```

That final command downloads the credentials to connect to the Kubernetes API
from your local kubectl command line.

### 1.3.6 在 AWS 中运行一个单节点的 Kubernetes

The managed Kubernetes service in AWS is called the Elastic Kubernetes Service
(EKS). You can create a single-node EKS cluster with the same caveat as Azure—that
you’ll be paying for that node and associated resources all the time it’s running.

You can use the AWS portal to create an EKS cluster, but the recommended way is
with a dedicated tool called eksctl. The latest documentation for the tool is at https://
eksctl.io, but it’s pretty simple to use. First, install the latest version of the tool for your
operating system as follows:

```
# install on macOS:
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl
# OR on Windows:
choco install eksctl
# OR on Linux:
curl --silent --location
"https://github.com/weaveworks/eksctl/releases/download/latest/eksctl_$(uname
-s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```

Assuming you already have the AWS CLI installed, eksctl will use the credentials from
the CLI (if not, then check the installation guide for authenticating eksctl). Then create a simple one-node cluster as follows:

```
# create a single node cluster with 2 CPU cores and 8GB RAM:
eksctl create cluster --name=kiamol --nodes=1 --node-type=t3.large
```

The tool sets up the connection from your local kubectl to the EKS cluster.

### 1.3.7 验证你的集群

Now you have a running Kubernetes cluster, and whichever option you chose, they all
work in the same way. Run the following command to check that your cluster is up
and running:

`kubectl get nodes`

You should see output like that shown in figure 1.6. It’s a list of all the nodes in your
cluster, with some basic details like the status and Kubernetes version. The details of
your cluster may be different, but as long as you see a node listed and in the ready
state, then your cluster is good to go.


![图1.6](./images/Figure1.6.png)
<center>图1.6 如果你可以运行 Kubectl 命令并且节点已就绪，那么你就可以继续往后了 </center>

## 1.4 立即见效

“Immediately effective” is a core principle of the Month of Lunches series. In all, the
focus is on learning skills and putting them into practice, in every chapter that follows.

Each chapter starts with a short introduction to the topic, followed by try-it-now
exercises where you put the ideas into practice using your own Kubernetes cluster.
Then there’s a recap with some more detail, to fill in some of the questions you may
have from diving in. Last, there’s a hands-on lab for you to try by yourself, to really
gain confidence in your new understanding.

All the topics center on tasks that are genuinely useful in the real world. You’ll
learn how to be immediately effective with the topic during the chapter, and you’ll finish by understanding how to apply the new skill. Let’s start running some containerized apps!