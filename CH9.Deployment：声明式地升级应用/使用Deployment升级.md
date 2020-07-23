## 使用Deployment

### 为什么rolling-update过时？

最重要的一点，kubectl是客户端程序，它通过向k8s API服务器发送HTTP请求来完成伸缩，最后完成升级。如果在升级过程中发生异常（比如网络断开），整个升级过程就中断了。pod和rc都会处于中间状态。

另外，k8s的哲学是你告诉他你期望达到什么状态，由k8s挑选最佳方式去完成；而不是你事无巨细地告诉k8s怎么做。（面向yaml编程）

所以引入了Deployment资源。

### 使用Deployment

Deployment示例yaml

```yaml
apiVersion: apps/v1            ## 可以使用kubectl explain deploy.apiVersion确定当前集群环境应使用的值
kind: Deployment              
metadata:
  name: deploy-nodejs          ## deployment的名称
spec:
  selector:                    ## v1版本的Deployment必须提供标签选择器
    matchLabels:               ## 这一点和其他控制器如rc、rs不太一样
      app: nodejs              ## rc、rs会利用template.metadata.labels中的标签作为控制器的标签选择器
  replicas: 3
  template:
    metadata:
      labels:
        app: nodejs            ## pod的标签
    spec:
      containers:
      - image: art.sinovatio.com/public-docker/luksa/kubia:v1
        name: nodejs-1         ## pod内的容器名称
```

测试创建Deployment

```shell
## --record选项会记录滚动升级的原因
## 让我们可以使用kubectl rollout history查看Deployment的滚动升级记录，并在CHANGE-CAUSE列中查看
$ kubectl create -f deploy-v1.yaml --record
deployment.apps/deploy-nodejs created

$ kubectl get deploy
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deploy-nodejs           3/3     3            3           5s

## rs的名称由Deployment的名称加上hash组成
$ kubectl get rs
NAME                               DESIRED   CURRENT   READY   AGE
deploy-nodejs-6c88667799           3         3         3       11s

## 注意观察pod的名称，其实是由rs的名称加上5位hash组成的
$ kubectl get po
NAME                                     READY   STATUS    RESTARTS   AGE
deploy-nodejs-6c88667799-b4h79           1/1     Running   0          9m24s
deploy-nodejs-6c88667799-h86c9           1/1     Running   0          9m24s
deploy-nodejs-6c88667799-wzq9q           1/1     Running   0          9m24s
```

### Deployment方式的滚动升级

**两种策略**

- `RollingUpdate`（默认）：滚动更新，渐进地删除旧版本的pod，与此同时创建新版本的pod。应用程序在升级过程中都处于可用状态。
- `Recreate`：一次性删除所有旧版本的pod，然后创建新版本的pod，应用程序在升级过程中会出现短暂的不可用。

### 升级测试

为了稍微减慢滚动升级的速度，可以使用以下命令修改Deployment的描述文件。

```shell
## 语法 kubectl patch deployment <Deployment名称> -p <需要修改的参数的json表达>
## 缩写 kubectl patch deploy <Deployment名称> -p <需要修改的参数的json表达>
$ kubectl patch deploy deploy-nodejs -p '{"spec":{"minReadySeconds":10}}'
deployment.apps/deploy-nodejs patched

## minReadySeconds可以配合就绪探针阻止有故障的版本被部署
```

在窗口1循环请求NodePort类型的服务svc-nodejs

```shell
while true; do curl http://192.168.42.79:42336; done

## 一开始是v1的输出
This is v1 running in pod deploy-nodejs-6c88667799-4r4f7
This is v1 running in pod deploy-nodejs-6c88667799-ccnsh
This is v1 running in pod deploy-nodejs-6c88667799-lmkzl
...
## 升级完成后会变成v2的输出
This is v2 running in pod deploy-nodejs-566bf96c86-4s8x9
This is v2 running in pod deploy-nodejs-566bf96c86-4s8x9
This is v2 running in pod deploy-nodejs-566bf96c86-p2266
This is v2 running in pod deploy-nodejs-566bf96c86-ctgdh
...
```

在窗口2进行滚动升级

```shell
## 滚动升级前先查看原始的po和rs
$ kubectl get po | grep nodejs
NAME                                     READY   STATUS    RESTARTS   AGE
deploy-nodejs-6c88667799-4r4f7           1/1     Running   0          78m
deploy-nodejs-6c88667799-ccnsh           1/1     Running   0          78m
deploy-nodejs-6c88667799-lmkzl           1/1     Running   0          78m

$ kubectl get rs | grep nodejs
deploy-nodejs-6c88667799           3         3         3       79m

## 语法 kubectl set image deployment <Deployment名称> <容器名称>=<镜像名称>
## 缩写 kubectl set image deploy <Deployment名称> <容器名称>=<镜像名称>
$ kubectl set image deploy deploy-nodejs nodejs-1=art.sinovatio.com/public-docker/luksa/kubia:v2
deployment.apps/deploy-nodejs image updated

## 查看滚动升级状态，这样就可以不用一直kubectl get po | grep nodejs查看pod的状态
## 语法 kubectl rollout status deployment <Deployment名称>
## 缩写 kubectl rollout status deploy <Deployment名称>
$ kubectl rollout status deploy deploy-nodejs
Waiting for deployment "deploy-nodejs" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "deploy-nodejs" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "deploy-nodejs" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "deploy-nodejs" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "deploy-nodejs" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "deploy-nodejs" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "deploy-nodejs" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "deploy-nodejs" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "deploy-nodejs" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "deploy-nodejs" rollout to finish: 1 old replicas are pending termination...
deployment "deploy-nodejs" successfully rolled out

## 旧版本pod deploy-nodejs-6c88667799-ccnsh在Terminating时
## 新版本pod deploy-nodejs-566bf96c86-ctgdh在ContainerCreating
$ kubectl get po | grep nodejs
NAME                                     READY   STATUS              RESTARTS   AGE
deploy-nodejs-566bf96c86-ctgdh           0/1     ContainerCreating   0          1s
deploy-nodejs-566bf96c86-p2266           1/1     Running             0          3s
deploy-nodejs-6c88667799-4r4f7           1/1     Running             0          79m
deploy-nodejs-6c88667799-ccnsh           1/1     Terminating         0          79m
deploy-nodejs-6c88667799-lmkzl           1/1     Running             0          79m

## 过一段时间后，新版本pod都处于Running状态，所有的旧版本pod都被Terminating
$ kubectl get po | grep nodejs
NAME                                     READY   STATUS        RESTARTS   AGE
deploy-nodejs-566bf96c86-4s8x9           1/1     Running       0          7s
deploy-nodejs-566bf96c86-ctgdh           1/1     Running       0          9s
deploy-nodejs-566bf96c86-p2266           1/1     Running       0          11s
deploy-nodejs-6c88667799-4r4f7           1/1     Terminating   0          79m
deploy-nodejs-6c88667799-ccnsh           1/1     Terminating   0          79m
deploy-nodejs-6c88667799-lmkzl           1/1     Terminating   0          79m

## 再过一段时间，只保留新版本的pod
$ kubectl get po | grep nodejs
deploy-nodejs-566bf96c86-4s8x9           1/1     Running   0          44s
deploy-nodejs-566bf96c86-ctgdh           1/1     Running   0          46s
deploy-nodejs-566bf96c86-p2266           1/1     Running   0          48s

## 旧版本的rs仍然存在以供回滚
$ kubectl get rs | grep nodejs
NAME                               DESIRED   CURRENT   READY   AGE
deploy-nodejs-566bf96c86           3         3         3       16s
deploy-nodejs-6c88667799           0         0         0       79m
```

### 回滚到上个版本测试

先升级到v3版本，由于v3版本在5次请求以后会固定打印错误信息以模拟发生内部错误。所以需要回滚到v2版本。

```shell
## 升级到有问题的v3版本
$ kubectl set image deploy deploy-nodejs nodejs-1=art.sinovatio.com/public-docker/luksa/kubia:v3
deployment.apps/deploy-nodejs image updated

## 循环请求服务几次以后，会固定打印错误信息来模拟发生内部错误
...
This is v3 running in pod deploy-nodejs-5dcf676585-d2zn9
Some internal error has occurred! This is pod deploy-nodejs-5dcf676585-sxznc
Some internal error has occurred! This is pod deploy-nodejs-5dcf676585-sxznc
Some internal error has occurred! This is pod deploy-nodejs-5dcf676585-mf42p
Some internal error has occurred! This is pod deploy-nodejs-5dcf676585-sxznc
Some internal error has occurred! This is pod deploy-nodejs-5dcf676585-d2zn9
...

## 回滚
## 语法 kubectl rollout undo deployment <Deployment名称>
## 缩写 kubectl rollout undo deploy <Deployment名称>
$ kubectl rollout undo deploy deploy-nodejs
deployment.apps/deploy-nodejs rolled back

## 查看回滚状态
$ kubectl rollout status deploy deploy-nodejs
Waiting for deployment "deploy-nodejs" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "deploy-nodejs" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "deploy-nodejs" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "deploy-nodejs" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "deploy-nodejs" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "deploy-nodejs" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "deploy-nodejs" rollout to finish: 1 old replicas are pending termination...
deployment "deploy-nodejs" successfully rolled out

## 回滚成功后查看deployment滚动更新历史
## 语法 kubectl rollout history deployment <Deployment名称>
## 缩写 kubectl rollout history deploy <Deployment名称>
## 和书上的有点不一样，感觉书上的更明朗一点，这个看不出升级了什么（set image）
$ kubectl rollout history deploy deploy-nodejs
deployment.apps/deploy-nodejs 
REVISION  CHANGE-CAUSE
3         kubectl create --filename=deploy-v1.yaml --record=true
5         kubectl create --filename=deploy-v1.yaml --record=true
6         kubectl create --filename=deploy-v1.yaml --record=true
```

### 回滚到特定版本

```shell
## 回滚到特定版本
$ kubectl rollout undo deploy deploy-nodejs --to-revision=3

## 回滚成功后查看rs，会发现第一次留下的rs deploy-nodejs-6c88667799重新接管了3个pod
## 这就是为什么滚动升级和rs没有被删除的原因
$ kubectl get rs | grep nodejs
NAME                               DESIRED   CURRENT   READY   AGE
deploy-nodejs-566bf96c86           0         0         0       29m
deploy-nodejs-5dcf676585           0         0         0       21m
deploy-nodejs-6c88667799           3         3         3       131m

## 这个时候再使用curl请求服务，将由最旧版本的镜像创建的pod进行响应
curl http://192.168.42.79:42336
This is v1 running in pod deploy-nodejs-6c88667799-nq9bt
This is v1 running in pod deploy-nodejs-6c88667799-cs8xs
This is v1 running in pod deploy-nodejs-6c88667799-8v7g6
```

**注意**：使用Deployment时，不要手动删除rs。这样会丢失Deployment的历史版本记录而导致无法回滚。

**注意**：rs过多也会导致rs列表过于混乱，可以通过Deployment的`spec.revisionHistoryLimit`字段进行限制。该字段默认值为10。