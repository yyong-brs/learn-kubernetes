# 一个月学会 Kubernetes

![header](./header.png)

哈喽，欢迎来到我的课程。我希望本课程可以给大家带来良好的学习体验。

在每一章中都有一个明确的重点，一个有用的话题，并且这些话题是相互关联的，让你有一个全面的了解，了解如何在实践中使用 Kubernetes。你需要大量的练习，每天练习巩固每一章获得的知识，形成肌肉记忆。

可以移步到 [GitHub Pages](https://yyong-brs.github.io/learn-kubernetes/) 页面进行阅读。

更多云原生技术，请关注公众号：云原生拓展

![公众号](./gongzh.png)

# 目录

- **第一部分** 快速了解 Kubernetes

  - 第一章 [开始之前](./chapter1.md)

    - 1.1 [了解 Kubernetes](./chapter1.md#11-了解-kubernetes)

    - 1.2 [这本书适合你吗?](./chapter1.md#12-这本书适合你吗)

    - 1.3 [创建你的实验环境](./chapter1.md#13-创建你的实验环境)

    - 1.4 [立即见效](./chapter1.md#14-立即见效)
  
  - 第二章 [Pods & Deployment 在 Kubernetes 中的应用](./chapter2.md)

    - 2.1 [Kubernetes 如何运行并管理容器](./chapter2.md#21-kubernetes-如何运行并管理容器)

    - 2.2 [通过控制器运行 Pods](./chapter2.md#22-通过控制器运行-pods)

    - 2.3 [在清单文件中定义 Deployments](./chapter2.md#23-在清单文件中定义-deployments)

    - 2.4 [应用在 Pods 中运行](./chapter2.md#24-应用在-pods-中运行)

    - 2.5 [了解 Kubernetes 资源管理](./chapter2.md#25-了解-kubernetes-资源管理)

    - 2.6 [实验室](./chapter2.md#26-实验室)
  
  - 第三章 [通过 Service 网络连接 Pods](./chapter3.md)

    - 3.1 [Kubernetes 如何路由网络流量](./chapter3.md#31-kubernetes-如何路由网络流量)

    - 3.2 [在 Pods 间路由流量](./chapter3.md#32-在-pods-间路由流量)

    - 3.3 [路由外部流量到 Pods](./chapter3.md#33-路由外部流量到-pods)

    - 3.4 [将流量路由到 Kubernetes 外面](./chapter3.md#34-将流量路由到-kubernetes-外面)

    - 3.5 [理解 Kubernetes Service 解析](./chapter3.md#35-理解-kubernetes-service-解析)

    - 3.6 [实验室](./chapter3.md#36-实验室)  

  - 第四章 [通过 ConfigMaps 和 Secrets 配置应用程序](./chapter4.md)

    - 4.1 [Kubernetes 如何为应用提供配置](./chapter4.md#41-kubernetes-如何为应用提供配置)

    - 4.2 [在 ConfigMaps 中存储和使用配置文件](./chapter4.md#42-在-configmaps-中存储和使用配置文件)

    - 4.3 [从 ConfigMaps 中查找配置数据](./chapter4.md#43-从-configmaps-中查找配置数据)

    - 4.4 [使用 Secrets 配置敏感数据](./chapter4.md#44-使用-secrets-配置敏感数据)

    - 4.5 [管理 Kubernetes 中的应用程序配置](./chapter4.md#45-管理-kubernetes-中的应用程序配置)

    - 4.6 [实验室](./chapter4.md#46-实验室) 

  - 第五章 [通过 volumes,mounts,claims 存储数据](./chapter5.md)

    - 5.1 [Kubernetes 如何构建容器文件系统](./chapter5.md#51-kubernetes-如何构建容器文件系统)

    - 5.2 [在节点使用 volumes 及 mounts 存储数据](./chapter5.md#52-在节点使用-volumes-及-mounts-存储数据)

    - 5.3 [使用 persistent volumes 及 claims 存储集群范围数据](./chapter5.md#53-使用-persistent-volumes-及-claims-存储集群范围数据)

    - 5.4 [动态 volume provisioning 及 storage classes](./chapter5.md#54-动态-volume-provisioning-及-storage-classes)

    - 5.5 [理解 Kubernetes 中存储的选择](./chapter5.md#55-理解-kubernetes-中存储的选择)

    - 5.6 [实验室](./chapter5.md#56-实验室)

  - 第六章 [通过 controllers 在多个 Pod 之间扩展应用](./chapter6.md)

    - 6.1 [Kubernetes 如何大规模运行应用程序](./chapter6.md#61-kubernetes-如何大规模运行应用程序)

    - 6.2 [使用 Deployments 和 ReplicaSets 来扩展负载](./chapter6.md#62-使用-deployments-和-replicasets-来扩展负载)

    - 6.3 [使用 DaemonSets 实现高可用性](./chapter6.md#63-使用-daemonsets-实现高可用性)

    - 6.4 [理解 Kubernetes 中的对象所有权](./chapter6.md#64-理解-kubernetes-中的对象所有权)

    - 6.5 [实验室](./chapter6.md#65-实验室)