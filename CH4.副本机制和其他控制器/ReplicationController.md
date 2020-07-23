## ReplicationController

ReplicationController有三个主要部分：

- 标签选择器（label selector），用来确定ReplicationController作用域内有哪些pod
- 副本数（replica count），指定应运行的pod数量
- pod模板（pod template），用于创建新的pod副本

三个要素中，只有修改副本数会影响（新增或删除）已存在的pod，修改标签选择器和pod模板对已存在的pod没有影响。

- 修改标签选择器，已存在的pod可能会（也可能不会）被移出ReplicationController作用域，但不会被删除。
- 增大副本数，ReplicationController会自动新增pod，减小副本数，ReplicationController会删除已存在的pod。
- 修改pod模板，只在ReplicationController创建新的pod时才会应用修改后的新pod模板

### 删除ReplicationController而保留pod

通过将`--cascade`选项设置为false可以在删除ReplicationController的同时保留pod

```shell
$ kubectl delete rc redis-rc --cascade=false
```

### 实验1：更改pod标签

rc.yaml

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: redis-rc
spec:
  replicas: 3                 ## 会创建3的pod
  template:
    metadata:
      labels:
        app: redis            ## pod标签是app=redis
    spec:
      containers:
      - name: redis
        image: bitnami/redis:5.0.3
        ports:
        - containerPort: 6379
        env:
        - name: ALLOW_EMPTY_PASSWORD
          value: "yes"
```

```shell
## 通过yaml创建一个三副本的ReplicationController
$ kubectl create -f rc.yaml 
replicationcontroller/redis-rc created

## 查看ReplicationController
$ kubectl get rc
NAME       DESIRED   CURRENT   READY   AGE
redis-rc   3         3         0       6s

## 查看pod及pod的标签，可以看到ReplicationController创建了三个pod副本，标签都是app=redis
$ kubectl get pod --show-labels
NAME             READY   STATUS             RESTARTS   AGE   LABELS
redis-rc-f5zff   0/1     ImagePullBackOff   0          83s   app=redis
redis-rc-jtj54   0/1     ImagePullBackOff   0          83s   app=redis
redis-rc-wm2vr   0/1     ImagePullBackOff   0          83s   app=redis

## 手动将pod redis-rc-wm2vr的标签修改掉
$ kubectl label po redis-rc-wm2vr app=not-redis --overwrite
pod/redis-rc-wm2vr labeled

## 再次查看pod及pod的标签，可以看到ReplicationController重新创建了一个标签为app=redis的pod redis-rc-x7gbv
$ kubectl get pod --show-labels
NAME             READY   STATUS             RESTARTS   AGE    LABELS
redis-rc-f5zff   0/1     ImagePullBackOff   0          116s   app=redis
redis-rc-jtj54   0/1     ImagePullBackOff   0          116s   app=redis
redis-rc-wm2vr   0/1     ImagePullBackOff   0          116s   app=not-redis
redis-rc-x7gbv   0/1     ImagePullBackOff   0          4s     app=redis

## 手动将pod redis-rc-wm2vr的标签修改为原来的app=redis
$ kubectl label po redis-rc-wm2vr app=redis --overwrite
pod/redis-rc-wm2vr labeled

## 再次查看pod及pod的标签，可以看到后创建的pod被移除
$ kubectl get pod --show-labels
NAME             READY   STATUS             RESTARTS   AGE    LABELS
redis-rc-f5zff   0/1     ImagePullBackOff   0          2m9s   app=redis
redis-rc-jtj54   0/1     ImagePullBackOff   0          2m9s   app=redis
redis-rc-wm2vr   0/1     ImagePullBackOff   0          2m9s   app=redis
```

### 实验2：修改ReplicationController的标签选择器和pod模板

```shell
## 修改ReplicationController，将标签选择器和pod模板的标签修改为app=foo并保存
$ kubectl edit rc redis-rc
replicationcontroller/redis-rc edited

## 查看pod及pod的标签，可以看到新创建了3个标签为app=foo的pod
$ kubectl get po --show-labels
NAME             READY   STATUS             RESTARTS   AGE   LABELS
redis-rc-88x5f   0/1     ImagePullBackOff   0          10s   app=foo
redis-rc-f5zff   0/1     ImagePullBackOff   0          31m   app=redis
redis-rc-glsnn   0/1     ImagePullBackOff   0          10s   app=foo
redis-rc-jtj54   0/1     ImagePullBackOff   0          31m   app=redis
redis-rc-kvsfc   0/1     ImagePullBackOff   0          10s   app=foo
redis-rc-wm2vr   0/1     ImagePullBackOff   0          31m   app=redis

## 查看标签为app=foo的pod redis-rc-f5zff
## 可以看到metadata中没有ownerReferences字段了，说明该pod已经移出ReplicationController作用域了。
$ kubectl get pod redis-rc-f5zff -o yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    cni.projectcalico.org/podIP: 172.42.0.206/32
  creationTimestamp: "2020-06-30T01:56:29Z"
  generateName: redis-rc-
  labels:
    app: redis
  name: redis-rc-f5zff
  namespace: test-ns
  resourceVersion: "29691894"
  selfLink: /api/v1/namespaces/test-ns/pods/redis-rc-f5zff
  uid: 6350e33c-f746-460c-94ba-b69dc531de70
...redis

## 查看标签为app=redis的pod redis-rc-88x5f
## 可以看到metadata中有ownerReferences字段，说明该pod在ReplicationController作用域中。
$ kubectl get pod redis-rc-88x5f -o yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    cni.projectcalico.org/podIP: 172.42.0.210/32
  creationTimestamp: "2020-06-30T02:27:34Z"
  generateName: redis-rc-
  labels:
    app: foo
  name: redis-rc-88x5f
  namespace: test-ns
  ownerReferences:
  - apiVersion: v1
    blockOwnerDeletion: true
    controller: true
    kind: ReplicationController
    name: redis-rc
    uid: 5d0aaba3-7e07-4444-a681-ae8652a0d0b9
  resourceVersion: "29692589"
  selfLink: /api/v1/namespaces/test-ns/pods/redis-rc-88x5f
  uid: 800afb39-5604-4290-8c69-f7221b9f1dd3
...

```

### 实验3：通过yaml水平伸缩pod

```shell
## 修改ReplicationController，将replicas修改为10
$ kubectl edit rc redis-rc

## 查看ReplicationController状态，DESIRED和CURRENT目前都是10
$ kubectl get rc
NAME       DESIRED   CURRENT   READY   AGE
redis-rc   10        10        0       57m

## 可以看到ReplicationController增加了7个pod
$ kubectl get po --show-labels
NAME             READY   STATUS              RESTARTS   AGE   LABELS
redis-rc-9klnj   0/1     Pending             0          3s    app=redis
redis-rc-bk45k   0/1     Pending             0          3s    app=redis
redis-rc-brclr   0/1     ContainerCreating   0          3s    app=redis
redis-rc-d6cll   0/1     Pending             0          3s    app=redis
redis-rc-f5zff   0/1     ImagePullBackOff    0          51m   app=redis
redis-rc-jtj54   0/1     ImagePullBackOff    0          51m   app=redis
redis-rc-wm2vr   0/1     ImagePullBackOff    0          51m   app=redis
redis-rc-xm7tr   0/1     ContainerCreating   0          3s    app=redis
redis-rc-xplsd   0/1     Pending             0          3s    app=redis
redis-rc-zwm5t   0/1     Pending             0          3s    app=redis

## 修改ReplicationController，将replicas还原为3
$ kubectl edit rc redis-rc

## 可以看到ReplicationController增加的7个pod的状态都在变为Terminating
$ kubectl get po --show-labels
NAME             READY   STATUS             RESTARTS   AGE   LABELS
redis-rc-f5zff   0/1     ImagePullBackOff   0          51m   app=redis
redis-rc-9klnj   0/1     Terminating        0          39s   app=redis
redis-rc-bk45k   0/1     Terminating        0          39s   app=redis
redis-rc-brclr   0/1     Terminating        0          39s   app=redis
redis-rc-jtj54   0/1     ImagePullBackOff   0          51m   app=redis
redis-rc-d6cll   0/1     Terminating        0          39s   app=redis
redis-rc-xm7tr   0/1     Terminating        0          39s   app=redis
redis-rc-xplsd   0/1     Terminating        0          39s   app=redis
redis-rc-zwm5t   0/1     Terminating        0          39s   app=redis
redis-rc-wm2vr   0/1     ImagePullBackOff   0          51m   app=redis

## 最后剩下的还是一开始创建的3个pod
$ kubectl get po --show-labels
NAME             READY   STATUS             RESTARTS   AGE   LABELS
redis-rc-f5zff   0/1     ImagePullBackOff   0          51m   app=redis
redis-rc-jtj54   0/1     ImagePullBackOff   0          51m   app=redis
redis-rc-wm2vr   0/1     ImagePullBackOff   0          51m   app=redis
```

### 实验4：通过命令水平伸缩pod

```shell
## 通过scale命令将副本数变为5
$ kubectl scale rc redis-rc --replicas=5
replicationcontroller/redis-rc scaled

## 查看ReplicationController状态，DESIRED和CURRENT目前都是5
$ kubectl get rc redis-rc
NAME       DESIRED   CURRENT   READY   AGE
redis-rc   5         5         0       58m

## 可以看到ReplicationController增加了2个pod
$ kubectl get po --show-labels
NAME             READY   STATUS             RESTARTS   AGE   LABELS
redis-rc-f5zff   0/1     ImagePullBackOff   0          58m   app=redis
redis-rc-hj95x   0/1     ImagePullBackOff   0          9s    app=redis
redis-rc-jtj54   0/1     ImagePullBackOff   0          58m   app=redis
redis-rc-kzsvg   0/1     ImagePullBackOff   0          9s    app=redis
redis-rc-wm2vr   0/1     ImagePullBackOff   0          58m   app=redis

## 通过scale命令将副本数还原为3
$ kubectl scale rc redis-rc --replicas=3
replicationcontroller/redis-rc scaled

## 查看ReplicationController状态，DESIRED和CURRENT目前都是3
$ kubectl get rc redis-rc
NAME       DESIRED   CURRENT   READY   AGE
redis-rc   3         3         0       58m

## 可以看到ReplicationController新增的2个pod消失了
$ kubectl get po --show-labels
NAME             READY   STATUS             RESTARTS   AGE   LABELS
redis-rc-f5zff   0/1     ImagePullBackOff   0          59m   app=redis
redis-rc-jtj54   0/1     ImagePullBackOff   0          59m   app=redis
redis-rc-wm2vr   0/1     ImagePullBackOff   0          59m   app=redis
```

