## 使用rolling-update升级

示例yaml

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: rc-nodejs-v1          ## rc的名称
spec:
  replicas: 3
  template:
    metadata:
      name: pod-nodejs        ## pod的名称（似乎不对，pod的name会由rc的name加上随机字符串组成）
      labels:                
        app: nodejs           ## pod的标签           
    spec:
      containers:
      - image: art.sinovatio.com/public-docker/luksa/kubia:v1
        name: nodejs-1        ## pod内的容器名称
---                           ## ---表示另一个yaml文档
apiVersion: v1
kind: Service
metadata:
  name: svc-nodejs
spec:
  type: NodePort
  selector:                  
    app: nodejs               ## svc的标签选择器
  ports:
  - port: 8888                ## 集群内部pod间访问使用的端口
    targetPort: 8080          ## 转发到pod内的容器的端口
```

升级演示

```shell
$ kubectl create -f rc-and-svc.yaml

## rc及对应的3个pod副本就绪
$ kubectl get rc
NAME           DESIRED   CURRENT   READY   AGE
rc-nodejs-v1   3         3         3       3m56s

## 在节点物理机上可以通过curl http://192.168.42.79:42336访问pod内的容器
$ kubectl get svc
NAME                    TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
svc-nodejs              NodePort       10.43.91.144    <none>        8888:42336/TCP      16m
```

新开一个窗口（窗口1），一直通过NodePort服务svc-nodejs访问pod内的容器，会一直输出如下的打印信息
```shell
## 循环请求，通过NodePort服务svc-nodejs访问pod内的容器，会一直输出如下的打印信息
$ while true; do curl http://192.168.42.79:42336; done
...
This is v1 running in pod rc-nodejs-v1-swkf8
This is v1 running in pod rc-nodejs-v1-tpgnv
This is v1 running in pod rc-nodejs-v1-x6gl6
...
```

在窗口2中执行升级

```shell
## 执行升级
## kubectl rolling-update <旧版本rc名称> <新版本rc名称> --image=<新版本镜像>
## 从提示信息可以看出以下几点：
## rolling-update已经过时（Command "rolling-update" is deprecated, use "rollout" instead）
## 会自动创建新版本的rc（Created rc-nodejs-v2）
## 会伸缩新版本的rc，从0-3增加pod的数量
## 会伸缩旧版本的rc，从3-0减少pod的数量
## 会始终保证3个pod可用（keep 3 pods available），STATUS为Running
## 但不会超过4个pod的总数（don't exceed 4 pods），STATUS为Running的不超过4个，Terminating的不算
## 升级完成后，会删除旧版本的rc（Update succeeded. Deleting rc-nodejs-v1）
$ kubectl rolling-update rc-nodejs-v1 rc-nodejs-v2 --image=art.sinovatio.com/public-docker/luksa/kubia:v2
Command "rolling-update" is deprecated, use "rollout" instead
Created rc-nodejs-v2
Scaling up rc-nodejs-v2 from 0 to 3, scaling down rc-nodejs-v1 from 3 to 0 (keep 3 pods available, don't exceed 4 pods)
Scaling rc-nodejs-v2 up to 1
Scaling rc-nodejs-v1 down to 2
Scaling rc-nodejs-v2 up to 2
Scaling rc-nodejs-v1 down to 1
Scaling rc-nodejs-v2 up to 3
Scaling rc-nodejs-v1 down to 0
Update succeeded. Deleting rc-nodejs-v1
replicationcontroller/rc-nodejs-v2 rolling updated to "rc-nodejs-v2"
```

此时打开窗口3

```shell
## 自动创建了rc-nodejs-v2，并且在升级完成前，2个rc并存
$ kubectl get rc
NAME           DESIRED   CURRENT   READY   AGE
rc-nodejs-v1   3         3         3       9m26s
rc-nodejs-v2   1         1         1       9s

## 查看rc rc-nodejs-v1的描述，关注标签选择器
## 增加了一个deployment=534be9c07cf8cf93d26a75181a2b12dc-orig的标签选择器
$ kubectl get rc rc-nodejs-v1 -o yaml
apiVersion: v1
kind: ReplicationController
metadata:
  labels:
    app: nodejs
  name: rc-nodejs-v1
  namespace: prod
  resourceVersion: "38437634"
spec:
  replicas: 2
  selector:
    app: nodejs
    deployment: 534be9c07cf8cf93d26a75181a2b12dc-orig
...

## 查看rc rc-nodejs-v2的描述，关注标签选择器
## 也增加了一个deployment=6a403b489ea0a126d0d798c6cba80977的标签选择器
$ kubectl get rc rc-nodejs-v2 -o yaml
apiVersion: v1
kind: ReplicationController
metadata:
  labels:
    app: nodejs
  name: rc-nodejs-v2
  namespace: prod
  resourceVersion: "38437672"
spec:
  replicas: 2
  selector:
    app: nodejs
    deployment: 6a403b489ea0a126d0d798c6cba80977
...

## 可以看到旧版本的pod都加上了deployment=534be9c07cf8cf93d26a75181a2b12dc-orig的标签
## 新版本的pod则加上了deployment=6a403b489ea0a126d0d798c6cba80977的标签
$ kubectl get po --show-labels | grep nodejs
NAME                 READY STATUS    RESTARTS   AGE   LABELS
rc-nodejs-v1-swkf8   1/1   Running   0          13m   app=nodejs,deployment=534be9c07cf8cf93d26a75181a2b12dc-orig
rc-nodejs-v1-tpgnv   1/1   Running   0          15m   app=nodejs,deployment=534be9c07cf8cf93d26a75181a2b12dc-orig
rc-nodejs-v1-x6gl6   1/1   Running   0          16m   app=nodejs,deployment=534be9c07cf8cf93d26a75181a2b12dc-orig
rc-nodejs-v2-vczn6   1/1   Running   0          16m   app=nodejs,deployment=6a403b489ea0a126d0d798c6cba80977

## 可以看到有一个v1版本的pod处在Terminating状态
## rolling-update会输出keep 3 pods available, don't exceed 4 pods，与下面的情况符合
## 
$ kubectl get po | grep nodejs
NAME                 READY   STATUS        RESTARTS   AGE
rc-nodejs-v1-tpgnv   1/1     Running       0          11m
rc-nodejs-v1-x6gl6   1/1     Terminating   0          11m
rc-nodejs-v2-5w2fw   1/1     Running       0          16s
rc-nodejs-v2-plwk5   1/1     Running       0          82s
rc-nodejs-v2-vczn6   1/1     Running       0          2m28s

## 最终，所有的v1版本的pod都会被删除，只留下v2版本的pod
$ kubectl get po | grep nodejs
NAME                 READY   STATUS    RESTARTS   AGE
rc-nodejs-v2-5w2fw   1/1     Running   0          20m
rc-nodejs-v2-plwk5   1/1     Running   0          21m
rc-nodejs-v2-vczn6   1/1     Running   0          22m
```

回到窗口1，查看下历史输出，会看到v1和v2交替出现，最后全部变成v2的输出

```shell
...
This is v1 running in pod rc-nodejs-v1-tpgnv
This is v2 running in pod rc-nodejs-v2-plwk5
This is v1 running in pod rc-nodejs-v1-x6gl6
This is v1 running in pod rc-nodejs-v1-swkf8
This is v2 running in pod rc-nodejs-v2-vczn6
This is v2 running in pod rc-nodejs-v2-5w2fw
This is v2 running in pod rc-nodejs-v2-5w2fw
This is v2 running in pod rc-nodejs-v2-plwk5
This is v2 running in pod rc-nodejs-v2-vczn6
...
```

