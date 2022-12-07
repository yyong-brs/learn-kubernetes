# 第一章 开始之前

Kubernetes 很强大。2014 年，在 GitHub 上它被作为开源项目发布，现如今在全球社区平均每周有 200 次变更提交，拥有2500名贡献者。一年一度的 KubeCon 会议的与会者从 2016年的 1000 多人增加到现在的 12000 多人，现在已经成为美国、欧洲和亚洲举办的全球系列活动。所有主流的云服务商提供托管的 Kubernetes 服务，您可以在数据中心或者在你的笔记本电脑上运行 Kubernete，最终它们都是等同的。

独立性和标准化是 Kubernetes 如此流行的主要原因。一旦您的应用程序在 Kubernetes 中良好运行，就可以部署它们到任何地方，这对迁移到云的组织都很有吸引力，因为
使它们能够在数据中心和其他云之间移动而无需重写代码。一旦你掌握了Kubernetes，你就可以在项目和组织之间快速移动，提高生产力。

但要达到这一点很难，因为 Kubernetes 很难。即使很简单应用程序也需要通过它部署为多个组件，以轻松的就可以跨越数百行的自定义文件格式代码进行描述。Kubernetes 将诸如负载平衡、网络、存储和计算等基础设施级别的问题带入应用程序配置，这可能是新概念，具体取决于您的IT背景。此外，Kubernetes 总是在扩展，它每季度发布新版本，通常会带来大量新功能。

但这是值得的。我花了很多年帮助人们学习 Kubernetes，然后一种共同的模式出现了：问题“为什么这么复杂？” 变成 “你能做到吗？这太神奇了！” Kubernetes确实是一项令人惊叹的技术。你对它了解得越多，你就会越喜欢它，这本书会加速你
的 Kubernetes 精通之旅。

## 1.1 了解 Kubernetes

本书提供了 Kubernetes 的实际使用介绍。每个章节都提供了“现在就试试”的练习，您可以通过练习和实验室获得大量使用 Kubernetes 的经验。我们将在下一章开始实际工作，但我们需要一点
理论先行。让我们先了解一下 Kubernetes到底是什么以及它解决了什么问题。

Kubernetes 是一个运行容器的平台，它负责启动容器化应用程序、滚动更新、Service 层面维护、扩展以满足需求、安全访问等。在 Kubernetes 中有两个核心概念，一个是 API，用于定义你的应用，另外一个是集群，运行你的应用。集群是一组单独的服务器，它们都配置了容器运行时，如Docker，然后使用Kubernetes 连接到单个逻辑单元中。图 1.1 显示了集群的高层级视图：

![图1.1](./images/Figure1.1.png)
<center>图1.1 Kubernetes 集群包含了一组服务器，它们加入到一个组中运行容器 </center>

集群管理员负责管理单独的服务器，它们在 Kubernetes 被称作节点（node）。你可以通过添加节点来扩展集群的容量，也可以使节点脱机以维护，或者升级 Kubernetes 集群。在像 Microsoft Azure Kubernetes Sevice（AKS）或 Amazon Elastic Kubernete Service（EKS）这样的托管服务中，这些功能都封装在简单的 web 界面或命令行中。正常使用时您忘记了底层节点，将集群视为单个实体。

Kubernetes 集群用于运行你的应用程序。你通过 YAML 文件来定义应用，然后将这些文件发送给 Kubernetes API。Kubernetes 会查看你在 Yaml 文件中有什么要求，并将其与集群中已经运行的应用进行比较。它会进行任何必要的更改以达到所需的状态，这可能是更新配置、删除容器或创建新容器。为了高可用，容器将会被分发到集群中去，它们可以通过 Kubernetes 管理的虚拟网络进行通信。图 1.2 显示了部署的过程，但是没有看到节点，因为我们在这个层面上并不真正关心它们。

![图1.2](./images/Figure1.2.png)
<center>图1.2 当您将应用程序部署到 Kubernetes 集群时，通常可以忽略实际节点 </center>

定义应用的结构是你的工作，但是运行和管理的所有工作都交给了 Kubernetes 。如果某个集群中的节点断线了，然后有一些容器在该节点上，Kubernetes 发现了并开始在其它节点创建替换的容器。如果某个应用容器变成不健康状态，Kubernetes 会去重启它。如果组件由于高负载而承受压力，Kubernetes 可以启动额外的该组件的新容器来降低压力。如果你将你的工作通过 Docker image 以及 Kubernetes YAML 文件进行管理，你将得到可以以同样的方式在不同的集群上运行的自我修复应用程序。

Kubernetes 不仅仅管理容器，这促使它成为一个功能完整的应用程序平台。Kubernetes 集群有一个分布式数据库，您可以使用它来存储应用程序的配置文件和API密钥以及连接凭据等机密信息。Kubernetes 将这些信息无缝交付给您的容器
允许您在每个环境中使用相同的容器镜像，并应用正确的配置。Kubernetes还提供存储，因此您的应用程序可以在容器外维护数据，为有状态应用程序提供高可用性。Kubernetes 还实现将网络流量发送到正确的容器进行处理。图1.3显示了其他资源类型：包括 Kubernetes的主要功能。

![图1.3](./images/Figure1.3.png)
<center>图1.3 Kubernetes 不仅仅管理容器，集群还管理其他资源 </center>

我还没有谈到容器中的应用程序是什么样子的；那是因为 Kubernetes 并不在乎。您可以在多个容器中运行通过云原生理念设计的跨多个微服务的应用。您可以运行作为一个整体构建在一个大容器中的旧版应用程序。它们可能是Linux应用程序或
Windows应用程序。您可以使用相同的API在YAML文件中定义所有类型的应用程序，你可以在一个集群上运行它们。使用Kubernetes的乐趣在于，它在所有应用之上上增加了一层一致性——老的 .Net和 Java 单体应用以及新的 Node.js和 Go 微服务都是以相同的方式被描述、部署和管理的。

这正是我们开始使用 Kubernetes 所需要的所有理论，但在此之前我们再深入一点，我想为我所讲的概念取一些恰当的名字。关于这些YAML文件被正确地称为应用程序清单（manifests），因为它们是一个应用程序的所有组件的列表。这些组件是Kubernetes 资源；他们也有自己的名字。图1.4采用了图1.3中的概念并应用正确的Kubernetes资源名称。

![图1.4](./images/Figure1.4.png)
<center>图1.4 真实情况：这些是您需要掌握的最基本的 Kubernetes 资源 </center>

我告诉过你Kubernetes很难。:)但我们在接下来的几章中将一次覆盖所有这些资源，对理解进行分层。当你完成第6章时，该图将完全有意义，您将在 YAML文件中定义这些资源并在自己的Kubernetes 中运行这些资源拥有很多经验。

## 1.2 这本书适合你吗?

本书的目标是快速跟踪您的 Kubernetes 学习, 让你有信心在 Kubernetes 中定义和运行自己的应用程序，并且让你了解上生产实践之路是怎样的。学习 Kubernetes 的最佳方法是练习，如果你遵循章节中的所有示例并在实验室中工作，
等你读完这本书，那么您将对Kubernetes的所有最重要的部分有一个坚实的理解。

但 Kubernetes 是一个巨大的话题，我不会涵盖所有内容。最大的差距在管理方面，我不会深入讨论集群设置和管理，因为它们在不同的基础设施中有所不同。如果你计划小跑进入云环境中的Kubernetes作为您的生产环境，那么无论如何，托管服务中都会处理这些问题。如果你想获得 Kubernetes 认证，这本书是一个很好的开始，但它不会让你一直受益。有两个主要的Kubernetes认证：Certified Kubernetes Application Developer (CKAD) 以及 Certified Kubernetes Administrator (CKA)。这本书约占 CKAD 课程的 80%，约占 CKA 课程的50%。

此外，你还需要掌握合理数量的背景知识来有效地阅读本书。当我们在讲解 Kubernetes 的特性时，但我不会填补任何关于容器的空白。如果您不熟悉镜像、容器和注册表等概念，我建议从我的这本书《一个月学会 Docker》开始。你不需要在使用 Kubernetes 时使用 Docker，但它是打包您的应用程序，以便您可以在Kubernetes的容器中运行它们的工具。

如果您将自己归类为一个全新的或想提升 Kubernetes 知识，那么这就是适合你的书。你的背景角色可能是开发、运维、架构、DevOps或站点可靠性工程（SRE）——Kubernetes涉及所有这些角色，因此他们都受到欢迎，并且你会学到很多东西。

## 1.3 创建你的实验环境

每个 Kubernetes 集群可以包含上百个节点，但是对于本书的练习来说，只需要单节点就够了。我们现在将会配置你的实验环境，为下一章做好准备。有数十种 kubernetes 平台可供使用，本书中的练习适用于任何经认证的 Kubernetes。我将会讲解如何在 Linux、Windows、Mac、AWS 以及 Azure 中创建你的实验环境，这些包括了主流的选项。我使用了 Kubernetes 1.18 版本，但是更早或更新的版本也是可以的。

本地运行 Kubernetes 最简单的选择是 Docker Desktop，它是一个软件包为您提供 Docker 和 Kubernetes 以及所有命令行工具。它还可以很好地与您的计算机网络集成，并有一个方便的重置 Kubernetes 按钮，必要时可以清除所有内容。Docker Desktop 在 Windows 10 和 macOS 上受支持，如果这对您不起作用，我也将介绍一些替代方案。

有一点您应该知道：Kubernetes 本身的组件需要作为 Linux 容器运行。您不能在 Windows 中运行 Kubernetes（尽管您可以在带有多节点的 Kubernetes 集群中的容器运行 Windows 应用程序），因此您需要一个 Linux虚拟机（VM）（如果您在Windows上工作）。Docker Desktop 启动该虚拟机并为您管理它。

对于 Windows 用户，最后一个注意事项是：请使用 PowerShell 来进行相关练习。PowerShell 支持许多 Linux 命令，如果您尝试使用经典的 Windows 命令终端，您将从一开始就遇到一些问题。

### 1.3.1 下载本书源码

本书的所有例子和练习的源码都存储在 GitHub 仓库中，同时还提供了一些样例解决方案。如果你真在使用 Git 并且已经安装 Git 客户端，你可以通过如下命令 Clone 仓库到你的电脑上：

`git clone https://github.com/yyong-brs/learn-kubernetes.git`

如果你不熟悉 Git 使用，你可以访问 https://github.com/yyong-brs/learn-kubernetes 点击 Code - Download ZIP 下载源文件。

根目录下有个 kiamol 文件夹包含了源码信息，然后它也包含了每一章的练习内容。

### 1.3.2 安装 Docker Desktop

Docker Desktop 运行在 Windows 10 或者 macOS Sierra (版本 10.12 或者更高版本)。浏览器访问 https://www.docker.com/products/docker-desktop 并选择稳定版本。下载之后运行它，接受所有的默认设置。在 Windows 上，这可能包括重新启动以添加新的 Windows 功能的操作。当 Docker Desktop 成功运行，您将在 Windows 任务栏或Mac菜单栏上的时钟附近看到 Docker 的鲸鱼图标。如果您是 Windows 上经验丰富的 Docker Desktop用户，您需要确保您处于Linux 容器模式（对于新安装时这是默认模式）。

默认情况下，Kubernetes 并未默认设置安装，因此你需要单机鲸鱼图标打开面板，然后点击“设置”，这将打开图 1.5 所示的窗口：从菜单中选择 Kubernetes ,然后选择 Enable Kubernetes。

![图1.5](./images/Figure1.5.png)
<center>图1.5 Docker Desktop 创建了一个 Linux 虚拟机去运行容器，同时可以运行 Kubernetes </center>

Docker Desktop 将会下载 Kubernetes 运行时的所有容器镜像——这将会花费一些时间，最终会启动所有服务。当你在“设置”屏幕底部看到两个绿色
的圆点，Kubernetes集群已准备就绪。Docker Desktop 已安装您所需的所有功能，因此您可以跳到 1.3.7 节。

其他 Kubernetes 发行版可以在 Docker Desktop 上运行，但它们没有与 Docker Desktop 使用的网络设置完美集成，练习时会出现问题。Docker Desktop 中的 Kubernetes 选项具有
这本书所需要的功能，绝对是最简单的选择。

### 1.3.3 安装 Docker 社区版本以及 K3s

如果您使用的是 Linux机器 或 Linux 虚拟机，您可以有几个选项来运行单节点集群。Kind 和 minikube 很受欢迎，但我更喜欢K3s,它包含最小的安装，但具有练习所需的所有功能。（k3s 是Kubernetes的缩写“K8s”上的一个参照。K3s对Kubernetes代码库进行了精简，名称表明它的大小是K8s的一半。）

K3s 与 Docker 兼容，因此首先，您应该安装 Docker 社区版。你可以在查看完整的安装步骤 https://rancher.com/docs/k3s/latest/en/quick-start/，这将使您快速启动：

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
如果您喜欢在 VM 中运行实验室环境，并且您熟悉使用 Vagrant 管理 VM，您可以使用 Docker 和 K3s 的以下 Vagrant 设置，在本书的源存储库中可以找到：

```
# from the root of the Kiamol repo:
cd ch01/vagrant-k3s
# provision the machine:
vagrant up
# and connect:
vagrant ssh
```

K3s 安装了您需要的所有功能，因此您可以跳到 1.3.7 节。

### 1.3.4 安装 Kubernetes 命令行工具

您可以使用名为kubectl（发音为“cube-cutle”）的工具管理 Kubernetes。它连接到Kubernetes 集群并与 Kubernetes API 交互工作。Docker Desktop 和 K3s 都安装 kubectl，但如果您使用下面描述的其他选项之一，则需要自己安装。

完整的安装说明位于 https://kubernetes.io/docs/tasks/tools/install-kubectl/。您可以在 macOS 上使用 Homebrew，在 Windows 上使用Chocolatey，以及
对于Linux，您可以下载二进制文件：

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

您可以使用 AKS 在 Microsoft Azure 中运行托管 Kubernetes 集群。如果您想从多台计算机访问集群，或者具有 Azure 信用的MSDN订阅这可能是比较好的选择。您可以运行最小的单个节点集群，这不会花费巨大的费用，但要记住，你没有办法停止集群，您将支付 24*7 的费用，直到您移除它。

Azure 门户有一个很好的用户界面来创建AKS集群，但它使用 az 命令要容易得多。您可以在查看最新文档https://docs.microsoft.com/en-us/azure/aks/kubenetes-walkthrough，但您可以通过下载az命令行工具并运行一些命令，如下所示：

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

最后一个命令从本地kubectl命令行下载连接到 Kubernetes API 的凭据。

### 1.3.6 在 AWS 中运行一个单节点的 Kubernetes

AWS 中的托管Kubernetes服务称为 Elastic Kubernetes Service（EKS）。您可以创建一个单节点EKS集群，但需要注意的是您将在该节点运行的所有服务和资源的时间支付费用。

您可以使用 AWS 门户创建 EKS 集群，但推荐的方法是使用名为 eksctl 的专用工具。该工具的最新文档位于https://eksctl.io，使用起来很简单。首先，为您的操作系安装最新的工具如下：

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

假设您已经安装了AWS CLI，eksctl将使用来自 CLI 的凭证（如果没有，请查看安装指南以验证eksctl）。然后创建一个简单的单节点集群，如下所示：

```
# create a single node cluster with 2 CPU cores and 8GB RAM:
eksctl create cluster --name=kiamol --nodes=1 --node-type=t3.large
```

该工具设置从本地 kubectl 到EKS 集群的连接。

### 1.3.7 验证你的集群

现在你已经拥有了一个运行的 Kubernetes 集群，不论你如何安装的，它们都以同样的方式工作。运行下面的命令来检查集群运行状态：

`kubectl get nodes`

你应该会看到他 1.6 类似的输出。输出显示了集群中所有的节点列表，同时显示了一些基本信息比如状态以及 kubernetes 版本。你的集群信息跟我的可能会有所不通，但是一旦你看到节点列表并且是 ready 状态，那就说明你的集群已经正常运行。


![图1.6](./images/Figure1.6.png)
<center>图1.6 如果你可以运行 Kubectl 命令并且节点已就绪，那么你就可以继续往后了 </center>

## 1.4 立即见效

“立即见效” 是本书的核心原则，总结来说，就是聚焦于能力练习，跟随我在每一章中进行实操。

每一章都以一个简单的主体说明开始，随后跟随的就是“现在就试试”的练习，通过它你可以使用你自己的集群来练习。然后是一个包含更多细节的说明，以解决您可能因为深入学习遇到的一些问题。最后，有一个动手实验室供您自己尝试，请对你的新知识的掌握充满信心。

所有的主题都集中在现实世界中真正有用的任务上。你会在本章中学习如何立即有效地处理该主题，您将通过了解如何应用新技能来完成。让我们开始运行一些容器化应用程序！