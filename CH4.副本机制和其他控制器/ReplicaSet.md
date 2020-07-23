## ReplicaSet

ReplicaSet的行为与ReplicationController完全相同，但是增强了pod选择器的表达能力。

- ~~ReplicationController无法同时匹配同一个标签不同值的pod，ReplicaSet可以。~~
- ~~ReplicationController无法基于标签名的存在来匹配pod，ReplicaSet可以。~~

selector字段可以使用两种形式的标签选择器：`matchLabels`和`matchExpressions`

matchLabels示例（rs.yaml）：

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: redis-rs
spec:
  replicas: 3
  selector:                  ## 相当于ReplicationController的:
    matchLabels:             ## selector:
      app: redis             ##   app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: bitnami/redis:5.0.3
        env:
        - name: ALLOW_EMPTY_PASSWORD
          value: "yes"
```

matchExpressions示例（rs-v2.yaml）：

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: redis-rs-v2
spec:
  replicas: 3
  selector:
    matchExpressions:
      - key: app                  ## 可以是app=redis或app=mysql或app=mongodb中的一个
        operator: In
        values: 
          - redis
          - mysql
          - mongodb
      - key: version              ## version必须是stable的，不能是snapshot或beta的
        operator: NotIn
        values:
          - snapshot
          - beta
      - key: env                  ## 必须有env标签，不管是prod环境部署的还是dev环境部署的
        operator: Exists
      - key: needAuth             ## 不能有needAuth标签，代表不需要身份认证
        operator: DoesNotExist
  template:
    metadata:
      labels:
        app: redis
        version: stable
        env: dev
    spec:
      containers:
      - name: redis
        image: bitnami/redis:5.0.3
```

matchExpressions接收的每个表达式都必须包含一个key，一个operator，可能还有一个values列表（取决于operator）。

有四个有效的运算符：

- `In`：pod的标签值在values数组中。
- `NotIn`：pod的标签值不在values数组中。
- `Exists`：pod必须包含一个指定的标签名，标签值不重要。使用该运算符，不能指定values数组。
- `DoesNotExist`：pod不能包含指定的标签名，标签值不重要。使用该运算符，不能指定values数组。

**注意**：如果指定了多个表达式，所有表达式都必须为true才能使标签选择器与pod匹配。

**注意**：如果同时指定了matchLabels和matchExpressions，则所有标签都必须匹配

### 实验1：

redis-rs使用了matchLabels，redis-rs-v2使用了matchExpressions。两个ReplicaSet都创建了3个pod副本。

修改redis-rs创建的3个pod副本的标签，使他们也满足redis-rs-v2的表达式。

当两个ReplicaSet并存时，redis-rs-v2并没有接管并删除redis-rs创建的3个pod副本，尽管这些pod副本符合redis-rs-v2的表达式。而redis-rs也没有重新创建3个仅包含app=redis标签的pod副本。

当redis-rs被删除后，redis-rs创建的3个pod副本被redis-rs-v2接管并被删除。

```shell
## 使用rs-v2.yaml创建ReplicaSet
$ kubectl create -f rs-v2.yaml 
replicaset.apps/redis-rs-v2 created

## 查看ReplicaSet，可以看到redis-rs和redis-rs-v2并存
$ kubectl get rs
NAME          DESIRED   CURRENT   READY   AGE
redis-rs      3         3         0       173m
redis-rs-v2   3         3         0       6s

## 查看pod的状态和标签，可以看到redis-rs和redis-rs-v2都创建了3个pod副本
$ kubectl get po --show-labels
NAME                READY   STATUS             RESTARTS   AGE    LABELS
redis-rs-bh2sg      0/1     ImagePullBackOff   0          173m   app=redis
redis-rs-cqw7k      0/1     ImagePullBackOff   0          173m   app=redis
redis-rs-rfqbj      0/1     ImagePullBackOff   0          173m   app=redis
redis-rs-v2-kw6ql   0/1     ImagePullBackOff   0          19s    app=redis,env=dev,version=stable
redis-rs-v2-mj4tr   0/1     ImagePullBackOff   0          19s    app=redis,env=dev,version=stable
redis-rs-v2-qtvsb   0/1     ImagePullBackOff   0          19s    app=redis,env=dev,version=stable

## 给redis-rs管理的3个pod副本添加env和version标签，使他们也满足redis-rs-v2的表达式
$ kubectl label po redis-rs-bh2sg env=prod version=stable
pod/redis-rs-bh2sg labeled
$ kubectl label po redis-rs-cqw7k env=prod version=stable
pod/redis-rs-cqw7k labeled
$ kubectl label po redis-rs-rfqbj env=prod version=stable
pod/redis-rs-rfqbj labeled

## 再次查看pod的状态和标签，可以看到redis-rs创建的3个pod副本的标签变化了
## 预期：redis-rs-v2接管redis-rs创建的3个pod副本并删除；redis-rs重新创建3个仅包含app=redis标签的pod副本
## 但是redis-rs-v2并没有按照预期接管redis-rs创建的3个pod副本并删除，redis-rs也没有重新创建3个仅包含app=redis标签的pod副本
## 猜测原因是redis-rs仍然活跃，它创建的3个pod副本仍然处于受管理状态
$ kubectl get po --show-labels
NAME                READY   STATUS             RESTARTS   AGE     LABELS
redis-rs-bh2sg      0/1     ImagePullBackOff   0          176m    app=redis,env=prod,version=stable
redis-rs-cqw7k      0/1     ImagePullBackOff   0          176m    app=redis,env=prod,version=stable
redis-rs-rfqbj      0/1     ImagePullBackOff   0          176m    app=redis,env=prod,version=stable
redis-rs-v2-kw6ql   0/1     ImagePullBackOff   0          3m21s   app=redis,env=dev,version=stable
redis-rs-v2-mj4tr   0/1     ImagePullBackOff   0          3m21s   app=redis,env=dev,version=stable
redis-rs-v2-qtvsb   0/1     ImagePullBackOff   0          3m21s   app=redis,env=dev,version=stable

## 删除redis-rs
$ kubectl delete rs redis-rs
replicaset.apps "redis-rs" deleted

## 再次查看pod的状态和标签，可以看到redis-rs创建的3个pod副本STATUS变为Terminating
## 说明redis-rs-v2已经接管了这3个pod副本，并且正在删除他们
$ kubectl get po --show-labels
NAME                READY   STATUS             RESTARTS   AGE     LABELS
redis-rs-bh2sg      0/1     Terminating        0          179m    app=redis,env=prod,version=stable
redis-rs-cqw7k      0/1     Terminating        0          179m    app=redis,env=prod,version=stable
redis-rs-rfqbj      0/1     Terminating        0          179m    app=redis,env=prod,version=stable
redis-rs-v2-kw6ql   0/1     ImagePullBackOff   0          2m52s   app=redis,env=dev,version=stable
redis-rs-v2-mj4tr   0/1     ImagePullBackOff   0          6m34s   app=redis,env=dev,version=stable
redis-rs-v2-qtvsb   0/1     ImagePullBackOff   0          6m34s   app=redis,env=dev,version=stable

## 再次查看pod的状态和标签，可以看到redis-rs创建的3个pod副本消失了
$ kubectl get po --show-labels
NAME                READY   STATUS             RESTARTS   AGE     LABELS
redis-rs-v2-kw6ql   0/1     ImagePullBackOff   0          2m59s   app=redis,env=dev,version=stable
redis-rs-v2-mj4tr   0/1     ImagePullBackOff   0          6m41s   app=redis,env=dev,version=stable
redis-rs-v2-qtvsb   0/1     ImagePullBackOff   0          6m41s   app=redis,env=dev,version=stable
```

