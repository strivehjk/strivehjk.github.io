---
title: kubernetes 节点管理
date: 2019-01-17 07:41:25
tags:
- k8s
- nodes
categories: kubernetes
description: kubernetes节点管理
---

source: [https://blog.csdn.net/dkfajsldfsdfsd/article/details/80985062][1]
翻译自: [https://kubernetes.io/docs/concepts/architecture/nodes/][2]

在自己搭建的单点集群中，slaver虚机的IP发生变化时，在master上还总是能看到之前的结点信息。
下面的内容，解释了为什么失联的node不会被删掉。

－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－

node是kubernetes集群中的工作节点，可以是虚拟机也可以是物理机。node节点上运行一些服务用来运行pod以及与master通信等。一个node上的服务包括docker运行时环境、kubelet、kube-proxy以及其它一些可选的add-ons。

# 节点状态
node的状态包含如下几个方面的信息:
```
Addresses
Condition
Capacity
Info
Addresses
```
节点的地址信息由node的配置决定，包含如下三项:
```
HostName：同node内核决定的主机名，可以由kubelet的--hostname-override选项覆盖。
ExternalIP：外网访问IP。
InternalIP：内网IP，集群内部的私有地址。
```

## Conditions
Conditions描述所有处于RUNNING状态的节点信息。

| Node Condition         | 	Description                                                                                                                                                                                                                                                |
| :-:                    | :-                                                                                                                                                                                                                                                             |
| OutOfDisk	          | _True_ if there is insufficient free space on the node for adding new pods, otherwise _False_                                                                                                                                                              |
| Ready	              | _True_ if the node is healthy and ready to accept pods,<br> _False_ if the node is not healthy and is not accepting pods, and<br> _Unknown_ if the node controller has not heard from the node in the last _node-monitor-grace-period_ (default is 40 seconds) |
| MemoryPressure         | _True_ if pressure exists on the node memory – that is, if the node memory is low; otherwise _False_                                                                                                                                                       |
| PIDPressure	        | _True_ if pressure exists on the processes – that is, if there are too many processes on the node; otherwise _False_                                                                                                                                       |
| DiskPressure	       | _True_ if pressure exists on the disk size – that is, if the disk capacity is low; otherwise _False_                                                                                                                                                       |
| NetworkUnavailable	 | _True_ if the network for the node is not correctly configured, otherwise _False_                                                                                                                                                                          |
| ConfigOK	           | _True_ if the kubelet is correctly configured, otherwise _False_                                                                                                                                                                                           |

关键问题是以上这些信息是如何计算判定的呢？

Node的conditions用json对象表示，如以下应答表示健康的node：
```
"conditions": [
  {
    "type": "Ready",
    "status": "True"
  }
]
```
如果节点的Ready condition是“Unknown” 或者 “False”，kube-controller-manager则将这个节点标记为不可用状态。节点上所有的pod被Node Controller调度排除，默认的排除超时时间是５分钟。有时节点因为网络原因不可达，kubernetes的控制面无法与节点上的kubelet建立联系，删除pod的指令无法下发到kubelet上，则可能的情况是控制面一直试图联系失联节点直到通信恢复，在此期间在此失联节点上的pod一直持续运行。

在kubernetes１.５及以前版本中，node controller可能会强制从系统中删除不可达的pod。但是，在１.５及更高的版本中，node controller直到明确确认这些pod停止运行后才会将它们从系统记录中删除，也就是说node controller必需明确知道pod的状态如“Terminating” 或者 “Unknown”。对于永久失联的node，kubernetes无法确定这个node是暂时失联稍后恢复还是永久从集群中删除，在这种情况下需要管理员手动删除node对象，删除node会引发kubernetes删除节点上的所有pod，并且释放它们所占用的名称。

从1.8版本开始介绍了一种新的测试特性，就是自动创建taints表示conditions。要想打开这种特性，需要给API server，controller manager，scheduler传递一个开关： --feature-gates=...,TaintNodesByCondition=true。当TaintNodesByCondition被打开后，scheduler在调度时就会忽略掉节点的conditions，代之以查看节点的taints与pod的tolerations。

现在用户可以在旧的调度模式与新的调度模式，一种更灵活的调度模式之间作出选择。对于没有任何tolerations的pod使用旧的调度模式，但是，如果pod能够容忍在特定节点上被污染的话，就会被调度到那个特定节点上。

需要注意：node的condition被观测到taints被创建的时间跨度，通常小于１秒，因为这种时间延迟，有可能会轻微的增加pod被成功调度出去，但是kubelet却拒绝接收的情况。

## Capacity
描述节点的可用资源：CPU、内在以及其上可以调度的最大的pod数等。

## Info
节点常规信息，如内核版本、kubernetes版本，docker版本、操作系统名称等。这些信息由节点上的kubelet提供。

# 节点管理
node不同于pod、service等kubernetes内部创建的对象，它是由外部创建。当在kubernetes内部创建node时，只是在kubernetes内部创建一个表示已经存在节点的对象。创建完以后kubernetes会检查节点是否可用，比如创建了如下节点:
```
{
  "kind": "Node",
  "apiVersion": "v1",
  "metadata": {
    "name": "10.240.79.157",
    "labels": {
      "name": "my-first-k8s-node"
    }
  }
}
```
kubernetes由提供的信息在内部创建一个代表节点的node对象，然后对节点执行可用性检查，如必需的服务是否在正常运行，是否具备运行pod的资格。如果不可用则不让这个节点参与任何集群活动。系统会保留不可用node并且持续监控它的状态，直到它可用为止，或者是用户明确删除创建的node对象。

目前，有三个组件会与kubernetes的node接口交互：node controller, kubelet, and kubectl。

## 节点控制器
Node controller是kubernetes控制面中的一个组件，负责管理node的方方面面。

在node的整个生命周期中，node controller充当多种角色，首先在node注册入集群时为它分配一个CIDR地址块（如果此特性打开的话）。

第二个是维护node controller内部的可用节点列表，并与低层的云供商的可用节点的列表保持一致。当运行在云环境中时，如果node controller检测到某个节点不健康，它就会向供应商询问节点虚拟机是否可用，如果不可用就从列表删除。

第三个是监控节点的健康状态。Node controller检测到node不可达时，负责将node状态中的NodeReady condition更新到ConditionUnknown，然后将node上的所有pod排除（如果节点不可达的时候超过40s（默认值），会被标记成ConditionUnknown，此时并没有发生排除pod的动作，系统试图重新联系不可达的node，如果恢复的话则node重新被标记成NodeReady。如果不可达的时间超过５分钟（默认值）才会采取排除步骤）。系统每隔--node-monitor-period设置的秒数检测一次节点的状态。

在kubernetes１.４中，改善了node controller处理集群中node数过多的逻辑。从1.4开始，node controller在决定是否排除pod之前，先检测一次集群中所有node的状态。
在多数情况下，node controller通过--node-eviction-rate的设定值限制了排除pod的速率,默认是0.1次每秒，意思是对于单个node，排除的速率小于10秒一次。

当node在一个可用的zone变得不健康时，node的排除行为会发生改变，也就是说排除行为并不是一成不变。Node controller检测在zone内不健康node的百分数，如果不健康node的比例低于--unhealthy-zone-threshold (default 0.55)，那么排除速率将会降低：如果集群规模小（node数小于或者等于--large-cluster-size-threshold，默认50），则排除行为终止，则否，排除速率将会降低到--secondary-node-eviction-rate（默认0.01）每秒。如此策略的主要原因主要是为在将一个可用的zone分区时提供一种过渡缓冲。如果你的集群没有跨多个云供应商的话，那么你只有一个可用的zone，就是整个集群。

将node分布在不同可用zone的主要原因是当一个zone整体不能用时，可以将workload切换到另一个可用的zone。所以当一个zone不可用时，kubernetes以正常速率排除。极端情况下，如果所有zone都不可用，那么kubernetes假定master之间的联通性出现了问题，停止一切排除行为直到master之间的联通性恢复。

从1.6版本开始，当pod不容忍taints时，node controller也负责排除运行在node上的NoExecute taints，这是一个测试特性，默认禁止。Node controller负责添加与node问题对应的taint，比如不可达。

从1.8版本开始，node controller能够被用来创建表示node conditions的taint，这是1.8版本的一个测试特性。

## 自注册节点
当node中kubelete的标志--register-node被设置成true时，kubelete尝试主动注册自已到系统中。这是一种被大多数distros使用的比较好的模式。本质是是由kubelete在系统中创建node对象而非用户手动。
自注册node的kubelet需要指定如下几个启动参数：
+ __--kubeconfig__ - Path to credentials to authenticate itself to the apiserver.
+ __--cloud-provider__ - How to talk to a cloud provider to read metadata about itself.
+ __--register-node__ - Automatically register with the API server.
+ __--register-with-taints__ - Register the node with the given list of taints (comma separated <key>=<value>:<effect>). No-op if register-node is false.
+ __--node-ip__ - IP address of the node.
+ __--node-labels__ - Labels to add when registering the node in the cluster.
+ __--node-status-update-frequency__ - Specifies how often kubelet posts node status to master.
目前kubelete被授权任意创建node对象，但在被际操作中它只创建、修改它自己。在将来的版本中计划将kubelete的授权收缩，使它只能创建、修改它自己。

# 手动管理node
集群管理员可以手动创建、修改node对象，当然先要把node的自注册功能关掉，设置node的kubelete标志--register-node=false。

用户可以手动设置node标签、或者使node不可以被安排调度。

标签可以被node selectors用来聚合node从而控制调度，比如约束pod只能调度到符合条件的node上。

使node不可调度，可以阻止node授受新的调度任务，对已经存在的pod没有影响。这个特性很有用。使node不可调度可运行如下命令：
<pre>kubectl cordon <b>$NODENAME</b></pre>

注意，使node不可调度，对DaemonSet类型的pod无效。

## Node capacity
Node的capacity也是node对象的一部分。自注册节点在创建node对象时自动注册此部分内容。但是，如果是手动管理node对象的话，则也需要手动设置此部分内部。

Kubernetes scheduler在调度pod之前，首先要确保node上有足够运行pod的资源，它要确保node上运行的所有container占用的资源不能超过节点的 capacity。但是，默认情况下在统计node的资源占用情况时，只计算由kubelete启动的容器，其它进程占用的资源不在统计之内。

如果想明确检视非pod进程占用的资源的话，按如下模板在每个node节点上创建manifest类型的pod：
```
apiVersion: v1
kind: Pod
metadata:
  name: resource-reserver
spec:
  containers:
  - name: sleep-forever
    image: k8s.gcr.io/pause:0.8.0
    resources:
      requests:
        cpu: 100m
        memory: 100Mi
```


[1]: https://blog.csdn.net/dkfajsldfsdfsd/article/details/80985062
[2]: https://kubernetes.io/docs/concepts/architecture/nodes/
