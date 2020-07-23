## YAML

```shell
## 查看k8s的API列表
$ kubectl api-resources

## 通过explain命令，可以知道pod的yaml中可以写哪些字段
$ kubectl explain pod

## 通过explain命令，可以知道pod.spec的yaml中可以写哪些字段
$ kubectl explain pod.spec

## 以此类推，可以逐级.下去，例如pod.spec.containers.readinessProbe可以知道就绪探针在yaml中可以写哪些字段
$ kubectl explain pod.spec.containers.readinessProbe
```



### yaml语法

yaml中的横杠（`-`）代表数组中的一个元素

如下是一个ReplicationController的yaml：

```yaml
apiVersion: v1
kind: ReplicationController
metadata:                     ## 直接的metadata控制创建资源的元数据，如name，labels
  name: kubia                 ##
spec:                         ##
  replicas: 3                 ## 直接的spec控制创建资源的规则，如副本数
  selector:                   ## spec下的selector，选择具有该标签的pod
    app: kubia                ## 选择具有app标签，值为kubia的pod
  template:                   ## spec.template 指定pod模板
    metadata:                 ## spec.template.metadata pod模板的元数据
      labels:                 ## spec.template.metadata.labels pod模板的元数据：如该pod具有以下标签
        app: kubia            ## 从该模板被创建出的pod会有app标签，标签值为kubia
    spec:                     ##   
      containers:             ##   
      - name: kubia           ##   
        image: luksa/kubia    ##   
        ports:                ##   
        - containerPort: 8080 ##
```

`metadata`和`spec`可以理解为每个资源都有，`template`既然是pod的模板，要从这个模板创建pod，那这个模板也有`metadata`和`spec`。

**注意**：模板中pod的标签必须和ReplicationController的标签选择器匹配，否则ReplicationController会无休止地创建pod。为了防止这种情况，API服务会校验ReplicationController的定义。也可以不指定ReplicationController的标签选择器，它会根据pod模板中的标签自动配置。

提示：定义ReplicationController时不要指定pod选择器，k8s会从pod模板中提取它，这样YAML更简短。

可以对照pod的yaml进行比较

```yaml
apiVersion: v1
kind: Pod                   ## 资源类型是pod
metadata: 
  name: kubia-manual-v2     ## pod的名称
  labels:                   ## pod的标签
    creation_method: manual ## 标签1: key为creation_method, 值为manual
    env: prod               ## 标签2: key为env, 值为prod
spec: 
  containers:
  - image: luksa/kubia
    name: kubia
    ports:
    - containerPort: 8080
      protocol: TCP
```

### metadata.ownerReferences

通过YAML中的该字段可以知道一个pod属于哪个ReplicationController（或ReplicaSet）