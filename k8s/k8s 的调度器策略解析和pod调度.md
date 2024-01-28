# k8s 的调度器策略解析和pod调度

## pod调度

### nodeName

`nodeName` 是比亲和性或者 `nodeSelector` 更为直接的形式。`nodeName` 是 Pod 规约中的一个字段。如果 `nodeName` 字段不为空，调度器会忽略该 Pod， 而指定节点上的 kubelet 会尝试将 Pod 放到该节点上。

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  nodeName: node-02 # 调度 Pod 到指定的节点，指定一个 node的 lable 只需要写该lable的value
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
```

使用 `nodeName` 来选择节点的方式有一些局限性：

- 如果所指代的节点不存在，则 Pod 无法运行，而且在某些情况下可能会被自动删除。
- 如果所指代的节点无法提供用来运行 Pod 所需的资源，Pod 会失败， 而其失败原因中会给出是否因为内存或 CPU 不足而造成无法运行。
- 在云环境中的节点名称并不总是可预测的，也不总是稳定的。

> `nodeName` 旨在供自定义调度器或需要绕过任何已配置调度器的高级场景使用。 如果已分配的 Node 负载过重，绕过调度器可能会导致 Pod 失败。 你可以使用[节点亲和性](https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/assign-pod-node/#node-affinity)或 [`nodeselector` 字段](https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/assign-pod-node/#nodeselector)将 Pod 分配给特定 Node，而无需绕过调度器。

### nodeselector

`nodeSelector` 是节点选择约束的最简单推荐形式。你可以将 `nodeSelector` 字段添加到 Pod 的规约中设置你希望目标节点所具有的节点标签。 只会将 Pod 调度到拥有你所指定的该标签的节点上。

```shell
kubectl label nodes node-01 device=gpu #给node-01 添加一个label，可以给多个node打标签，最后会根据scheduler的策略从有该标签的node上选择出最优node
kubectl get nodes -l 'device' -o custom-columns=NAME:.metadata.name #查看那个节点拥有改标签
```

```yaml
#selector-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  nodeSelector:
    device: gpu
```

```shell
kubectl get pod -o wide #查看
```

当 `nodeName` 和`nodeSelector` 同时存在并且配置相反时，会调度到nodename所指定的node，但会报Warning  NodeAffinity     kubelet  Predicate **NodeAffinity** failed，因此不要这样做，为什么会`nodeName` 和`nodeSelector` 同时存在也需要思考

### 亲和性（Affinity）和反亲和性（Anti-affinity）

亲和性：则基于节点的标签（Labels）和 Pods 的标签，调度到满足特定规则的节点上

反亲和性：反亲和性正好与亲和性相反，

亲和性和反亲和性的策略有两种：

- `requiredDuringSchedulingIgnoredDuringExecution`：强制性的，要求节点或pod必须满足指定的标签要求，否则 Pod 无法被调度

- `preferredDuringSchedulingIgnoredDuringExecution`：优选的，调度器会尽量满足标签要求，但不是强制要求。

> 亲和性和反亲和性中只存在：nodeAffinity、podAffinity、podAntiAffinity

#### nodeAffinity

```
kubectl label nodes node-01 disktype=ssd 
```

```yaml
#nodeAffinity.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions: 
          - key: disktype
            operator: In
            values:
            - ssd            
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
```

```shell
k get pod -o wide 
NAME    READY   STATUS    RESTARTS   AGE   IP             NODE      NOMINATED NODE   READINESS GATES
nginx   1/1     Running   0          7s    10.88.190.36   node-01   <none>           <none>
```

```shell
kubectl label nodes  node-02 device=gpu
```

```yaml
#nodeAffinity.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
      - weight: 100
        preference:
          matchExpressions:
          - key: device
            operator: In
            values:
            - gpu
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
```

```yaml
kubectl get pod -o wide
NAME    READY   STATUS    RESTARTS   AGE     IP             NODE      NOMINATED NODE   READINESS GATES
nginx   1/1     Running   0          3m48s   10.88.184.19   node-02   <none>           <none>

```

> weight 参数用于指定约束的优先级，较高的权重值表示更重要的约束。调度器会更倾向于满足具有较高权重的约束。其取值范围是 1 到 100。 当调度器找到能够满足 Pod 的其他调度请求的节点时，调度器会遍历节点满足的所有的偏好性规则， 并将对应表达式的 `weight` 值加和。
>
> 最终的加和值会添加到该节点的其他优先级函数的评分之上。 在调度器为 Pod 作出调度决定时，总分最高的节点的优先级也最高。

#### podAffinity和podAntiAffinity

```
#pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: with-pod-affinity
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: security
            operator: In
            values:
            - s1
        topologyKey: kubernetes.io/hostname
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: security
              operator: In
              values:
              - S2
          topologyKey: kubernetes.io/hostname //node上默认的标签，这些标签常用来区分node区域
  containers:
  - name: with-pod-affinity
    image: registry.k8s.io/pause:2.0
```

`matchExpressions`: 是一种基于表达式的选择器，它允许您使用条件表达式来指定要匹配的标签。

`matchFields`:  与 `matchExpressions` 类似，但它是用于基于资源的字段而不是标签来进行选择。这允许您基于资源的内部属性（如 metadata.fields）来选择资源。

`operator` 字段中可以使用的所有逻辑运算符。

| **操作符** |            **行为**            |
| :--------: | :----------------------------: |
|    `In`    |  标签值存在于提供的字符串集中  |
|  `NotIn`   | 标签值不包含在提供的字符串集中 |
|  `Exists`  |    对象上存在具有此键的标签    |

以下操作符只能与 `nodeAffinity` 一起使用。

| 操作符 |                             行为                             |
| :----: | :----------------------------------------------------------: |
|  `Gt`  | 提供的值将被解析为整数，并且该整数小于通过解析此选择算符命名的标签的值所得到的整数 |
|  `Lt`  | 提供的值将被解析为整数，并且该整数大于通过解析此选择算符命名的标签的值所得到的整数 |

### 污点(Taint)和容忍度(Toleration)

`污点（Taint）`使节点能够排斥一类特定的 Pod。

`容忍度（Toleration）` 是应用于 Pod 上的。容忍度允许调度器调度带有对应污点的 Pod。 容忍度允许调度但并不保证调度：作为其功能的一部分， 调度器也会。

污点和容忍度（Toleration）相互配合，可以用来避免 Pod 被分配到不合适的节点上。 每个节点上都可以应用一个或多个污点，这表示对于那些不能容忍这些污点的 Pod， 是不会被该节点接受的。

每个污点（Taint）都有一个排斥等级（Taint Effect），指定了如何处理具有不同容忍度（Toleration）的 Pod。排斥等级有以下三种：

- NoSchedule：新的 Pod 不会被调度到该带有污点的节点上。 当前正在节点上运行的 Pod **不会**被驱逐。
- NoExecute：新的pod不会调度到该带有污点的节点上。并且删除已经运行的pod。

- PreferNoSchedule：尽可能地避免在该节点上调度没有匹配容忍度的 Pod，但不会强制要求。如果没有其他可用节点，Pod 将被调度到该节点上。

```shell
kubectl taint nodes node1 key1=value1:NoExecute #给node1配置一个污点
kubectl taint nodes node1 key1=value1:NoExecute- #删除污点
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    security: s1
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  nodeSelector:
    disktype: ssd
  tolerations:  #配置容忍
  - key: "k1"
    operator: "Equal"
    value: "v1"
    effect: "NoExecute"
    tolerationSeconds: 120 #表示在给节点添加了上述污点之后， Pod 还能继续在节点上运行的时间
```

operator 字段的值（operator` 的默认值是 `Equal）：

1、Equal：容忍度与污点必须在key、value和effect三个字段完全匹配

2、Exists：key和effect必须完全匹配



### 调度器策略解析

kube-scheduler 调度分为两个阶段，predicate和priority

- predicate：过滤不符合条件的节点，节点预选
- priority: 优先级排序，选择优先级最高的节点，节点优先级排序

