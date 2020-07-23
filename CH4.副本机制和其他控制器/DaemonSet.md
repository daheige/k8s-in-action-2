## DaemonSet

在集群中的每个节点上都运行一个pod实例。

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: redis-ds
spec:
  selecor:
    matchLabels:
      app: redis-monitor
  template:
    metadata:
      labels:
        app: redis-monitor
    spec:
      nodeSelector:
        disk: ssd
      containers:
      - name: redis
        image: bitnami/redis
```

通过yaml文件创建DaemonSet并查看相应DaemonSet和pod状态

```shell
## 从yaml文件创建DaemonSet
$ kubectl create -f ds.yaml 
daemonset.apps/redis-ds created

## 查看DaemonSet状态
$ kubectl get ds
NAME       DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
redis-ds   1         1         0       1            0           disk=ssd        6s

## 单独查看pod所属的node
## custom-column语法可以参考http://kubernetes.io/docs/user-guide/kubectl-overview/#custom-columns
$ kubectl get po -o custom-columns=NODE:.spec.nodeName
NODE
voiceai

## 通过-o wide选项查看，也可以看到pod redis-ds-949p9在node voiceai上进行了部署
## 这说明DaemonSet生效了
$ kubectl get po -o wide
NAME             READY   STATUS             RESTARTS   AGE   IP             NODE      NOMINATED NODE   READINESS GATES
redis-ds-949p9   0/1     ImagePullBackOff   0          52m   172.42.0.251   voiceai   <none>           <none>
```

删除node的标签后pod也消失

```shell
## 先查看node的标签，里面有disk=ssd标签
$ kubectl get no --show-labels
NAME      STATUS   ROLES                                                                                  AGE   VERSION   LABELS
voiceai   Ready    controlplane,etcd,mongodb-node-1,mysql-node-1,nfs,redis-node-1,redis-sentinel,worker   87d   v1.17.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,disk=ssd,kubernetes.io/arch=amd64,kubernetes.io/hostname=voiceai,kubernetes.io/os=linux,node-role.kubernetes.io/controlplane=true,node-role.kubernetes.io/etcd=true,node-role.kubernetes.io/mongodb-node-1=true,node-role.kubernetes.io/mysql-node-1=true,node-role.kubernetes.io/nfs=true,node-role.kubernetes.io/redis-node-1=true,node-role.kubernetes.io/redis-sentinel=true,node-role.kubernetes.io/worker=true

## 删除node的disk=ssd标签
$ kubectl label no voiceai disk-
node/voiceai labeled

## 再次查看node的标签，确认disk=ssd标签已被删除
$ kubectl get no --show-labels
NAME      STATUS   ROLES                                                                                  AGE   VERSION   LABELS
voiceai   Ready    controlplane,etcd,mongodb-node-1,mysql-node-1,nfs,redis-node-1,redis-sentinel,worker   87d   v1.17.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=voiceai,kubernetes.io/os=linux,node-role.kubernetes.io/controlplane=true,node-role.kubernetes.io/etcd=true,node-role.kubernetes.io/mongodb-node-1=true,node-role.kubernetes.io/mysql-node-1=true,node-role.kubernetes.io/nfs=true,node-role.kubernetes.io/redis-node-1=true,node-role.kubernetes.io/redis-sentinel=true,node-role.kubernetes.io/worker=true

## 查看pod，发现已经没有pod了
$ kubectl get po
No resources found in test-ns namespace.
```

