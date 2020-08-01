## 使用StatefulSet

创建`headless`服务

```yaml
apiVersion: v1
kind: Service
metadata:
  name: svc-nginx
spec:
  clusterIP: None    
  selector:      
    app: nginx
  ports:
  - name: http
    port: 8888
```

创建sts

```yaml
apiVersion: apps/v1             ## 通过kubectl explain sts.apiVersion查看应该使用什么版本
kind: StatefulSet
metadata:
  name: sts-nginx
spec:
  serviceName: svc-nginx
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:                  
        app: nginx               
    spec:
      containers:
      - name: nginx-1
        image: art.sinovatio.com/public-docker/nginx:alpine
        ports:
        - name: http
          containerPort: 80                  ## pod内的容器的端口
        volumeMounts:
        - name: data            
          mountPath: /var/data  
  volumeClaimTemplates:
  - metadata:                  
      name: data               
    spec:                                    ## 这里和书里的例子稍微有点区别，使用sc创建pv，而不是手动管理
      storageClassName: managed-nfs-storage  ## 在pvc模板中利用storageClass创建pv
      resources:               
        requests:              
          storage: 100Mi                     ## pvc中声明的是100Mi，这样能匹配到对应的pv
      accessModes:             
      - ReadWriteOnce          
```

**注意**：正常阅读可以跳过错误演示。

### 错误演示1

```shell
## 创建service时报错
$ kubectl create -f svc-nginx.yaml 
error: error parsing svc-nginx.yaml: error converting YAML to JSON: yaml: line 7: mapping values are not allowed in this context
```

结合行号查看`svc-nginx.yaml`定义能看到第七行的缩进有问题，修正缩进后重新创建headless服务正常。

![image-20200727135328995](C:\Users\1001544\AppData\Roaming\Typora\typora-user-images\image-20200727135328995.png)

### 错误演示2

```shell
## 创建sts时报错
$ kubectl create -f sts-nginx.yaml 
error: error validating "sts-nginx.yaml": error validating data: ValidationError(StatefulSet.spec): missing required field "selector" in io.k8s.api.apps.v1.StatefulSetSpec; if you choose to ignore these errors, turn validation off with --validate=false
```

结合提示，v1版本的`StatefulSet`必须要提供`selector`，这点和`Deployment`一样，`Deployment`也必须提供`selector`。而`ReplicationController`和`ReplicaSet`控制器会利用`template.metadata.labels`中的标签作为控制器的标签选择器。增加`selector`后，`StatefulSet`创建成功。

![image-20200727134900223](C:\Users\1001544\AppData\Roaming\Typora\typora-user-images\image-20200727134900223.png)

### 错误演示3

```shell
## sts创建完成后，READY一直处于0/2，说明pod未就绪（第一个pod就没有就绪）
$ kubectl get sts
NAME              READY   AGE
mongodb-primary   1/1     112d
mysql-master      1/1     112d
redis-master      1/1     112d
sts-nginx         0/2     9s

## 查看pod列表，确实未就绪，READY为0/1
$ kubectl get po | grep nginx
NAME                                     READY   STATUS    RESTARTS   AGE
sts-nginx-0                              0/1     Pending   0          25s

## 查看pod的事件，查找未就绪原因（省略部分信息）
## 可以看到Events中VolumeBinding有错误，说明和pvc有关
$ kubectl describe po sts-nginx-0
Name:           sts-nginx-0
Namespace:      prod
Labels:         app=nginx
                controller-revision-hash=sts-nginx-84b58ccf54
                statefulset.kubernetes.io/pod-name=sts-nginx-0
Status:         Pending
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  37s   default-scheduler  error while running "VolumeBinding" filter plugin for pod "sts-nginx-0": pod has unbound immediate PersistentVolumeClaims

## 查看pvc列表（省略部分信息）
## 可以看到pvc的STATUS为Pending，说明无法找到匹配pvc声明的pv卷
$ kubectl get pvc
NAME               STATUS    VOLUME       CAPACITY   ACCESS MODES   STORAGECLASS          AGE
data-sts-nginx-0   Pending                                                                3m38s
pvc-nfs-sc         Bound     pvc-b3f...   100Mi      RWO            managed-nfs-storage   8d

## 因为pvc是在sts-nginx.yaml中使用volumeClaimTemplates创建的
## 所以需要打开sts-nginx.yaml查看volumeClaimTemplates定义
## 预期：
## sts使用volumeClaimTemplates创建pvc
## pvc又会使用StorageClass创建pv
## 这样就有pod->pvc->pv的链路就可以形成
## 但实际sc并没有为pvc创建pv，说明pvc中没有指明sc
```

对照pvc的yaml示例，可以发现：在volumeClaimTemplates下没有指定storageClassName，指定后重新创建StatefulSet，sts、pod、pvc及pv均正常。

![image-20200727135459322](C:\Users\1001544\AppData\Roaming\Typora\typora-user-images\image-20200727135459322.png)

### 正确场景演示

```shell
## 创建sts
$ kubectl create -f sts-nginx.yaml 
statefulset.apps/sts-nginx created

## 查看sts状态，还没有就绪
$ kubectl get sts
NAME              READY   AGE
mongodb-primary   1/1     112d
mysql-master      1/1     112d
redis-master      1/1     112d
sts-nginx         0/2     4s

## 可以看到一个pod已经READY，另一个pod正在创建中
## 第二个pod会在第一个pod运行且处于就绪状态后创建
## 因为：有状态的pod同时启动可能会造成一些资源争用的情况，所以依次启动每个pod会比较安全
## 这也提醒我们要设置好就绪探针，必须确认第一个pod绝对就绪，不会在第二个pod启动期间与其争用资源（我加的）
$ kubectl get po | grep nginx
NAME                                     READY   STATUS              RESTARTS   AGE
sts-nginx-0                              1/1     Running             0          8s
sts-nginx-1                              0/1     ContainerCreating   0          2s

## 两个pod均就绪
## 注意pod的名称是由<sts.metadata.name>-<0起始的序号>组成
$ kubectl get po | grep nginx
NAME                                     READY   STATUS    RESTARTS   AGE
sts-nginx-0                              1/1     Running   0          14s
sts-nginx-1                              1/1     Running   0          8s

## 考虑到上面的错误演示3，这次查看pvc会发现，由于设置了sc
## sc按照pvc的声明创建了我们需要的pv，所以pvc的STATUS列均显示Bound，也有对应的pv卷列出在VOLUME列
## 注意pvc的名字是data开头的，所以在sts的yaml中配置的volumeClaimTemplates.metadata.name其实是pvc的前缀
## pvc的实际名称由<volumeClaimTemplates.metadata.name>-<pod名称>组成
## 而pod名称又是由<sts.metadata.name>-<0起始的序号>组成
$ kubectl get pvc | grep nginx
NAME               STATUS   VOLUME             CAPACITY   ACCESS MODES   STORAGECLASS          AGE
data-sts-nginx-0   Bound    pvc-f26f4b5d-...   100Mi      RWO            managed-nfs-storage   22s
data-sts-nginx-1   Bound    pvc-0ed80506-...   100Mi      RWO            managed-nfs-storage   16s

## 省略了部分列以更好的显示
## pv列表中，pv也都绑定到pvc
$ kubectl get pv | grep nginx
NAME              CAPACITY  ACCESS MODES  STATUS  CLAIM                  STORAGECLASS         AGE
pvc-0ed80506-...  100Mi     RWO           Bound   prod/data-sts-nginx-1  managed-nfs-storage  22s
pvc-f26f4b5d-...  100Mi     RWO           Bound   prod/data-sts-nginx-0  managed-nfs-storage  28s

## 再次查看sts，由于所有2个pod均就绪，所以sts也显示就绪
$ kubectl get sts
NAME              READY   AGE
mongodb-primary   1/1     112d
mysql-master      1/1     112d
redis-master      1/1     112d
sts-nginx         2/2     39s
```

### 通过API服务器直接与pod通信

**注意**：不应该直接与pod通信。所以下面章节介绍了使用k8s API服务器，通过Service与pod通信。

```shell
## 与pod通信的一种方法是在另一个pod内部使用curl与目标pod通信
## 另一种方法是在节点物理机上使用端口转发（kubectl port-forward），然后使用curl与目标pod通信
## 第三种方法是使用k8s API服务器与目标pod通信
## 演示第三种方式的通信：前提是在另一个窗口使用kubectl proxy启动proxy
## 语法 curl <apiServerHost>:<port>/api/v1/namespaces/<namespace名称>/pods/<pod名称>/proxy/<path>
## 注意 结尾的proxy为通过k8s proxy访问的固定值
## proxy后面为pod内的容器对应的路径，/代表pod内的容器的根目录
$ curl localhost:8001/api/v1/namespaces/prod/pods/sts-nginx-0/proxy/
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

### 删除pod与缩容

```shell
## 该步骤需要使用作者的镜像创建sts并测试

## 先在两个pod中存储数据，验证StatefulSet是否能保证像pet一样完全一致

## 使用post，存储Hey, there字符串到pod sts-kubia-0
$ curl -X POST -d "Hey, there" localhost:8001/api/v1/namespaces/prod/pods/sts-kubia-0/proxy/
Data stored on pod sts-kubia-0
## 验证该字符串已被存储在pod sts-kubia-0
$ curl localhost:8001/api/v1/namespaces/prod/pods/sts-kubia-0/proxy/
You've hit sts-kubia-0
Data stored on this pod: Hey, there

## 使用post，存储sth sotred in sts-kubia-1字符串到pod sts-kubia-1
$ curl -X POST -d "sth sotred in sts-kubia-1" localhost:8001/api/v1/namespaces/prod/pods/sts-kubia-1/proxy/
Data stored on pod sts-kubia-1
## 验证该字符串已被存储在pod sts-kubia-1
$ curl localhost:8001/api/v1/namespaces/prod/pods/sts-kubia-1/proxy/
You've hit sts-kubia-1
Data stored on this pod: sth sotred in sts-kubia-1



## 测试手动删除高标号pod后重建
## 验证重建后的高标号pod能返回原始高标号pod存储的数据
$ kubectl delete po sts-kubia-1
pod "sts-kubia-1" deleted
## pod sts-kubia-1处于删除状态
$ kubectl get po | grep kubia
NAME                                     READY   STATUS        RESTARTS   AGE
sts-kubia-0                              1/1     Running       0          10m
sts-kubia-1                              1/1     Terminating   0          10m
## pod sts-kubia-1重建成功
$ kubectl get po | grep kubia
sts-kubia-0                              1/1     Running   0          10m
sts-kubia-1                              1/1     Running   0          3s
## sts正常
$ kubectl get sts | grep kubia
sts-kubia         2/2     10m
## 访问pod sts-kubia-1，验证存储在该原始pod的字符串是否能返回
## 能正常返回删除重建前存储的字符串数据
$ curl localhost:8001/api/v1/namespaces/prod/pods/sts-kubia-1/proxy/
You've hit sts-kubia-1
Data stored on this pod: sth sotred in sts-kubia-1


## 测试缩容再扩容
## 修改sts的replicas副本数为1，进行缩容
$ kubectl edit sts sts-kubia
statefulset.apps/sts-kubia edited
## sts正在缩容READY为2/1
$ kubectl get sts | grep kubia
NAME              READY   AGE
sts-kubia         2/1     13m
## 高标号的pod sts-kubia-1正处于Terminating状态
$ kubectl get po | grep kubia
NAME                                     READY   STATUS        RESTARTS   AGE
sts-kubia-0                              1/1     Running       0          13m
sts-kubia-1                              1/1     Terminating   0          2m49s
## 缩容成功
$ kubectl get sts | grep kubia
NAME              READY   AGE
sts-kubia         1/1     13m
## 仅保留了pod sts-kubia-0
$ kubectl get po | grep kubia
NAME                                     READY   STATUS    RESTARTS   AGE
sts-kubia-0                              1/1     Running   0          13m


## 修改sts的replicas副本数为2，进行扩容
$ kubectl edit sts sts-kubia
statefulset.apps/sts-kubia edited
## sts正在进行扩容，READY为1/2
$ kubectl get sts | grep kubia
NAME              READY   AGE
sts-kubia         1/2     30m
## 高标号的pod sts-kubia-1创建成功
$ kubectl get po | grep kubia
NAME                                     READY   STATUS    RESTARTS   AGE
sts-kubia-0                              1/1     Running   0          30m
sts-kubia-1                              1/1     Running   0          5s
## 访问pod sts-kubia-1，验证存储在该原始pod的字符串是否能返回
## 能正常返回缩容再扩容前存储的字符串数据
$ curl localhost:8001/api/v1/namespaces/prod/pods/sts-kubia-1/proxy/
You've hit sts-kubia-1
Data stored on this pod: sth sotred in sts-kubia-1
```

### 通过API服务器与Service通信（通过Service与pod通信）

<font style="color:red"><strong>暂时未跑通，提示no endpoints available for service "svc-nginx"</strong></font>

**注意**：不应该直接与pod通信，而应该通过Service与pod通信。

**语法**：`curl <apiServerHost>:<port>/api/v1/namespaces/<namespace名称>/services/<service名称>/proxy/<path>`

创建一个`ClusterIP`服务替代一开始的`headless`服务

```yaml
apiVersion: v1
kind: Service
metadata:
  name: svc-nginx-public       
spec:                   ## 不在spec中指定clusterIP为None就创建了一个ClusterIP服务
  selector:      
    app: nginx
  ports:
  - name: http
    port: 8888
    targetPort: 80
```

<font style="color:red"><strong>由于无法利用service的proxy url跑通，所以无法测试整个10.4章节：在StatefulSet中发现伙伴节点</strong></font>

```shell
## 通过一个能使用dig的pod测试dig SRV
## 使用dig列出DNS SRV记录
$ kubectl exec -it vpr-5cc48467c7-fgbgl -- dig SRV svc-kubia.prod.svc.cluster.local

; <<>> DiG 9.10.3-P4-Debian <<>> SRV svc-kubia.prod.svc.cluster.local
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 17883
;; flags: qr aa rd; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 3
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;svc-kubia.prod.svc.cluster.local. IN	SRV

;; ANSWER SECTION:
svc-kubia.prod.svc.cluster.local. 5 IN	SRV	0 50 80 sts-kubia-0.svc-kubia.prod.svc.cluster.local.
svc-kubia.prod.svc.cluster.local. 5 IN	SRV	0 50 80 sts-kubia-1.svc-kubia.prod.svc.cluster.local.

;; ADDITIONAL SECTION:
sts-kubia-0.svc-kubia.prod.svc.cluster.local. 5	IN A 172.42.0.176
sts-kubia-1.svc-kubia.prod.svc.cluster.local. 5	IN A 172.42.0.179

;; Query time: 84 msec
;; SERVER: 10.43.0.10#53(10.43.0.10)
;; WHEN: Mon Jul 27 14:16:00 CST 2020
;; MSG SIZE  rcvd: 373
```


<font style="color:red"><strong>由于测试环境是单节点，无法模拟几点掉线，所以无法测试整个10.5章节：了解StatefulSet如何处理节点失效</strong></font>
