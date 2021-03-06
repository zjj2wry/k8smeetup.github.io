---
assignees:
- bprashanth
- janetkuo
title: Replication Controller
redirect_from:
- "/docs/user-guide/replication-controller/"
- "/docs/user-guide/replication-controller/index.html"
---

* TOC
{:toc}



## 什么是 ReplicationController ?

_ReplicationController_ 确保在任何时间上运行 pod 的 "replicas" 数为定义的数量。换句话说，一个 ReplicationController 确保一个 pod 或同类的 pod 的集合总是运行和可用的。
如果 pod 超过指定的数量，它会杀死多出的。 如果少于指定数量，ReplicationController 将启动更多。与手动创建的 pod 不同，如果有 pod 失败、被删除或被终止，ReplicationController 会自动维护并替代这些 pod 。
例如，类似于内核升级这样的中断性维护，您的 pod 会在另一个节点上重新创建。
因此，即使您的应用程序只需要运行一个 pod ，我们也建议您使用 ReplicationController 。 您可以将 ReplicationController 视为与进程监视类似的东西，
ReplicationController 会监视跨多个节点的多个 pod ，而不是单个节点上的单个进程。

ReplicationController 通常会缩写为 “rc” 或 “rcs” ，并作为 kubectl 命令参数中的简写。

一个简单的例子是创建1个 ReplicationController 对象，以便可靠地运行一个永不停止的 pod 实例。更复杂的用例是运行复制的几个相同的副本服务，如 Web 服务器。



## 运行一个 ReplicationController 示例

这是一个 ReplicationController 配置示例。它运行3个 nginx Web 服务副本。

{% include code.html language="yaml" file="replication.yaml" ghlink="/docs/concepts/workloads/controllers/replication.yaml" %}

下载示例文件然后执行此命令运行示例：

```shell
$ kubectl create -f ./replication.yaml
replicationcontroller "nginx" created
```

用如下命令检查 ReplicationController 的状态：

```shell
$ kubectl describe replicationcontrollers/nginx
Name:        nginx
Namespace:   default
Image(s):    nginx
Selector:    app=nginx
Labels:      app=nginx
Replicas:    3 current / 3 desired
Pods Status: 0 Running / 3 Waiting / 0 Succeeded / 0 Failed
Events:
  FirstSeen       LastSeen     Count    From                        SubobjectPath    Type      Reason              Message
  ---------       --------     -----    ----                        -------------    ----      ------              -------
  20s             20s          1        {replication-controller }                    Normal    SuccessfulCreate    Created pod: nginx-qrm3m
  20s             20s          1        {replication-controller }                    Normal    SuccessfulCreate    Created pod: nginx-3ntk0
  20s             20s          1        {replication-controller }                    Normal    SuccessfulCreate    Created pod: nginx-4ok8v
```

这里3个 pod 已经被创建了，但还没有 running ，也许是因为正在拉镜像。
稍后，同样的命令可能显示：

```shell
Pods Status:    3 Running / 0 Waiting / 0 Succeeded / 0 Failed
```

可以使用如下命令列出属于 rc 的所有 pod ：

```shell
$ pods=$(kubectl get pods --selector=app=nginx --output=jsonpath={.items..metadata.name})
echo $pods
nginx-3ntk0 nginx-4ok8v nginx-qrm3m
```

在这里，selector 与 ReplicationController 的 selector 相同（`kubectl describe`可以查看输出信息，不同的`replication.yaml`有不同的输出信息。
`--output = jsonpath`选项指定一个表达式，它只是从返回的列表中获取每个pod的名称。



## 编写 ReplicationController Spec

与其他 Kubernetes 配置一样，ReplicationController 需要`apiVersion`、 `kind` 和 `metadata` 字段。
与配置文件有关的信息请浏览[这里](/docs/user-guide/simple-yaml/)、[这里](/docs/user-guide/configuring-containers/)和[这里](/docs/concepts/tools/kubectl/object-management-overview/).

ReplicationController还需要一个[`.spec` 部分](https://git.k8s.io/community/contributors/devel/api-conventions.md#spec-and-status)。



### Pod 模板

`.spec.template`是`.spec`的唯一必填字段。

`.spec.template`是一个 pod 模板。它与[pod](/docs/concepts/workloads/pods/pod/)的样式完全相同，除了它是嵌套的，并且没有 `apiVersion` 和 `kind`。

除了 Pod 的必填字段之外，ReplicationController 中的 pod 模板必须指定适当的 labels （即不与其他 controller 重叠，请参阅 [pod selector](#pod-selector))和适当的重启策略。

[`.spec.template.spec.restartPolicy`](/docs/concepts/workloads/pods/pod-lifecycle/)只允许等于`Always`，这也是未指定情况下的默认值。

对于本地容器的重新启动，ReplicationController 将委托给节点上的代理，例如 [Kubelet](/docs/admin/kubelet/) 或 Docker 。



### ReplicationController 的 Labels

ReplicationController 本身可以有标签（`.metadata.labels`）。 通常，您需要将其设置成与`.spec.template.metadata.labels`相同; 
如果没有指定`.metadata.labels`，那么默认值为`.spec.template.metadata.labels`。 但是，它们允许设置成不同的，`.metadata.labels` 不会影响 ReplicationController 的行为。



### Pod Selector

`.spec.selector`字段是指 [label selector](/docs/user-guide/labels/#label-selectors)。 ReplicationController 通过匹配 selector 的 labels 来管理所有 pod。 
它不区分 pod 是某人或某进程创建或删除的。 这允许替换 ReplicationController 而不影响正在运行的 pod。

如果指定 selector ，`.spec.template.metadata.labels`必须与`.spec.selector`相同，否则将被 API 拒绝。 
如果`.spec.selector`未指定，则将默认为`.spec.template.metadata.labels`。

此外，您不能直接通过另一个 ReplicationController 或另一个类似于 Job 的 controller 来创建与这个 labels 相同 selector 的 pod。 
否则， ReplicationController 会认为这些 pod 是由它创建的。 
但是， Kubernetes 不会阻止你这样做。

如果最终使用具有相同 selector 的多个 controller ，则必须自己去删除(参见[下文](#updating-a-replication-controller))。



### 多个 Replicas

您可以设置`.spec.replicas`来指定您想要同时运行的 pod 数量。 任何时候运行的数量可能会多于或少于指定的数量，例如， replicas 刚刚被增加或减少，或者 pod 被正常停止了，或者较久的被替换了。

如果您没有指定`.spec.replicas`的数量, 默认值是1。



## 使用 ReplicationControllers

### 删除 ReplicationController 和它的 Pods

要删除 ReplicationController 及其它的所有 pod ，请使用 [`kubectl delete`](/docs/user-guide/kubectl/{{page.version}}/#delete)。 
在删除 ReplicationController 本身之前， Kubectl 会将 ReplicationController 缩容为零并等待它删除完 pod 。 如果 kubectl 命令被中断，它会被重启。

当使用 REST API 或 go 客户端库时，您需要明确地执行这些步骤（将 replicas 缩容为0，等待 pod 删除，然后删除 ReplicationController ）。



### 只删除 ReplicationController

您可以删除 ReplicationController ，而不影响它的 pod 。

使用 kubectl ，为 [`kubectl delete`](/docs/user-guide/kubectl/{{page.version}}/#delete)指定`--cascade = false`选项。

使用 REST API 或go 客户端库时，只需删除 ReplicationController 对象即可。

当原始的 ReplicationController 被删除后，您可以创建一个新的 ReplicationController 来替换它。 
只要旧的和新的`.spec.selector`是一样的，那么新的 ReplicationController 将会接管旧的 pod 。
但是，在不同 pod template 的情况下，现有的 pod 是不会去匹配新的 template 。
使用 [rolling update](#滚动升级) 以可控的方式将 pod 更新为新的 spec 。



### 用 ReplicationController 隔离 pod

可以通过更改其 labels 来移除目标 ReplicationController 中的 pod 。 
该技术可用于从服务中删除 pod 以进行调试或数据恢复等。以这种方式移除的 pod 将被自动替换（假设 replicas 的数量没有变化）。



## 常见的使用方式

### 重新调度

如上所述，无论您是要运行1个 pod 还是1000个， ReplicationController 将保持指定数量的 pod 存在，即使在节点故障或 pod 终止的情况下（例如，由于其他控制代理的操作）。



### 扩容与缩容

通过简单地更新`replicas`字段， ReplicationController 可以通过控制代理轻松地手动或自动扩容和缩容 replica 的数量。



### 滚动升级

ReplicationController 旨在通过逐个替换 pod 来对服务进行滚动升级。

如[#1353](http://issue.k8s.io/1353)中所述，推荐的方法是创建一个新的 ReplicationController replicas ，逐个缩放新的（+1）和旧的（-1） pod 数量，
然后待 replicas 数量为0后删除旧的 controller 。 无论发生什么不可预料的故障，这种方法升级 pod 都是可以预见的。

理想情况下，滚动升级 controller 将考虑到应用程序的准备情况，并且确保足够数量的 pod 在任何时候都可以提供正常服务。

如果是两个 ReplicationController ，需要至少创建一个不同的 labels 来区别 Pod，就像 Pod 主容器的镜像标签，因为它是典型的滚动升级中的镜像更新。
 
滚动升级在客户端工具中实现 [`kubectl rolling-update`](/docs/user-guide/kubectl/{{page.version}}/#rolling-update)。 
更多具体的例子，请访问 [`kubectl rolling-update` task](/docs/tasks/run-application/rolling-update-replication-controller/)。



### 多版本轨道

当多版本应用程序正在运行的时候进行滚动升级，长时间运行多版本甚至持续运行多版本轨道。轨道将被 labels 区分开来。

举例来说，一个 service 可能会通过`tier in (frontend), environment in (prod)`来指向所有的 pod 。
那如果现在，你说你有10个 pod 的 replicas 组成了这一层，但是你希望能够创建一个新的'canary'版本的组件，
你可以设置 ReplicationController 的9个 replicas 的 labels 为`tier=frontend, environment=prod, track=stable`，
另外1个 replicationController 为 canary 的 labels 设置为`tier=frontend, environment=prod, track=canary`。
现在 service 就有 canary 和非 canary 的两种 pod 。因此你可能会搞混两个 replicationController 分别进行测试和监测得到的结果。



### 使用 ReplicationControllers 与 Services

多个 ReplicationControllers 可以关联为单个 service ，以便例如有些流量跑到旧版本，有些流量转到新版本。

ReplicationController 将永远不会自动终止，但不会像服务一样长期存在。 
服务可能由多个 ReplicationControllers 控制的 pod 组成，预计在 service 的整个生命周期内可能会创建和销毁许多 ReplicationControllers （例如，运行 service 的 pod 的更新）。 
不管是服务本身还是其客户端都应该为ReplicationControllers保留维护服务的pod。



## 编写 Replication 程序

由 ReplicationController 创建的 Pods 旨在可替换并且语义相同，尽管它们的配置可能随着时间的推移而变化。 
这明显适合复制无状态服务，但是也可以使用 ReplicationControllers 来维护主选、分片和工作池应用程序的可用性。 
这样的应用程序应该使用动态工作分配机制，例如 [etcd 锁模块](https://coreos.com/docs/distributed-configuration/etcd-modules/)
或 [RabbitMQ工作队列](https://www.rabbitmq.com/tutorials/tutorial-two-python.html)，而不是静态/一次性定制每个 pod 的配置，这被认为是一种反模式。 
任何 pod 的定制，例如资源的自动调整大小（如 cpu 或内存）应由另一个在线 controller 进程执行，与 ReplicationController 本身不一样。



## ReplicationController 的职责

ReplicationController 简单地确保所需数量的 pod 与其 label selector 相同并且是运行的。终止的 pod 一般是不算在数量里的。 
将来可能会考虑到 [readiness](http://issue.k8s.io/620)和系统可用的其他信息，我们可能会对更换政策增加更多的控制权，
我们计划发布可由外部客户端使用的事件来执行任意复杂的替换和/或缩减策略。

ReplicationController 永远被限制在这个狭隘的责任范围之内。 它本身不会执行 readiness 或 liveness 探测。 它不是执行自动缩放，
而是由外部自动缩放器控制（如[#492](http://issue.k8s.io/492)中所讨论的），它会更改其`replicas`字段。 
我们不会将调度策略（例如 [spreading](http://issue.k8s.io/367#issuecomment-48428019)）添加到 ReplicationController 。 
也不会校验控制的 pod 是否与当前指定的模板相同，因为这将阻碍自动调整大小和其他自动化过程。 类似于完成期限、排序依赖关系、配置扩展和其他功能是属于其他地方的。 
我们甚至计划考虑大容量创建 pod 的机制([#170](http://issue.k8s.io/170))。

ReplicationController 旨在成为可组合原始组件。 我们希望将来可以建立更高级别的 API 和/或工具，以及其他原始组件，以便日后方便用户使用。 
kubectl(run, stop, scale, rolling-update)目前支持的“宏”操作是验证概念示例。 例如，我们可以想像像 [Asgard](http://techblog.netflix.com/2012/06/asgard-web-based-cloud-management-and.html)
一样管理 ReplicationController、auto-scalers,、services,、scheduling policies,、canaries 等



## API 对象

Replicationcontroller 是 Kubernetes REST API 中的顶级资源。 有关 API 对象的更多详细信息，
请参阅：[ReplicationController API object](/docs/api-reference/{{page.version}}/#replicationcontroller-v1-core)。



## ReplicationController 的替代

### ReplicaSet

[`ReplicaSet`](/docs/concepts/workloads/controllers/replicaset/) 是下一代 ReplicationController ，支持新的 [set-based label selector](/docs/user-guide/labels/#set-based-requirement)。
它主要用[`Deployment`](/docs/concepts/workloads/controllers/deployment/) 作为协调 pod 创建、删除和更新的机制。
请注意，我们建议使用 Deployments 来代替直接使用 Replica Sets，除非您需要自定义更新编排或根本不需要更新。



### Deployment (推荐的)

[`Deployment`](/docs/concepts/workloads/controllers/deployment/)是一个更高级的 API 对象，它更新 Pod 类似于 Replica Sets 的`kubectl rolling-update`方式。 
如果您想要这种滚动更新功能，建议使用 Deployments ，因为与`kubectl rolling-update`不同，它是声明式的、服务器端的，并且还具有其他功能。



### 裸 Pods

与用户直接创建 pod 的情况不同，ReplicationController 会替换由于某些原因而被删除或终止的 pod ，例如在节点故障或节点维护需要中断（例如内核升级）的情况下。 
因此，即使您的应用程序只需要一个 pod ，我们也建议您使用 ReplicationController 。 与进程监视类似，监视多个节点上的多个 pod ，而不是只监视单个节点上的单个进程。 
ReplicationController 将本地容器重新启动委托给节点上的某个代理（例如 Kubelet 或 Docker ）。



### Job

使用 [`Job`](/docs/concepts/jobs/run-to-completion-finite-workloads/) 代替 ReplicationController ，用于自己预计终止的 pod （即批处理作业）。




### DaemonSet

使用[`DaemonSet`](/docs/concepts/workloads/controllers/daemonset/)代替 ReplicationController 来提供机器级功能，例如机器监控或机器日志记录。 
这些 pod 的生命周期与机器生命周期绑在一起：这些 pod 会在其他 pod 启动之前启动，并且当机器准备重新启动/关闭时会安全的终止。



## 了解更多信息

请参阅 [Run Stateless AP Replication Controller](/docs/tutorials/stateless-application/run-stateless-ap-replication-controller/)。