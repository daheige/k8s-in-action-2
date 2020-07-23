## 服务发现（客户端pod如何发现服务）

服务发现是为了解耦IP，使用环境变量、DNS和FQDN来解耦IP。

### 一、环境变量发现

仅适用于在service创建后创建的pod。

测试在服务创建后，删除一个语音识别的pod，该pod由replicaset管理，rs会重新创建pod，新创建的pod是否能识别到环境变量中的服务

```shell
## 创建ReplicationController
$ kubectl create -f rc.yaml 
replicationcontroller/redis-rc created

## 创建服务
$ kubectl create -f svc.yaml 
service/redis-svc created

## 查看pod
$ kubectl get po
NAME                                     READY   STATUS             RESTARTS   AGE
admin-console-66c55f8f89-vgqmw           1/1     Running            6          94d
admin-data-76bb5cbc9d-gflh9              1/1     Running            10         94d
admin-resource-server-754778f6ff-dxz9n   1/1     Running            6          94d
admin-server-7d4855d89-ds72h             1/1     Running            6          94d
asr-5b846fc649-xltjc                     1/1     Running            6          94d
asv-578949548f-mccgh                     1/1     Running            6          94d
dia-dcf98bc6c-crt42                      1/1     Running            6          94d
mongodb-primary-0                        1/1     Running            6          94d
mysql-master-0                           1/1     Running            6          94d
redis-master-0                           1/1     Running            6          94d
redis-rc-7brrr                           0/1     ImagePullBackOff   0          12s
redis-rc-ncdwr                           0/1     ImagePullBackOff   0          12s
redis-rc-xfkfj                           0/1     ImagePullBackOff   0          12s
vpr-5cc48467c7-kjn87                     1/1     Running            0          3m6s
vpr-audio-784578ddcb-pzc4t               1/1     Running            6          94d
vprc-7748b6b6b4-8jhss                    1/1     Running            11         94d

## 语音识别pod vpr-5cc48467c7-kjn87是由ReplicaSet管理的，所以可以安全的删除它，ReplicaSet会重建一个pod
$ kubectl delete po vpr-5cc48467c7-kjn87
pod "vpr-5cc48467c7-kjn87" deleted

## ReplicaSet重建了一个pod vpr-5cc48467c7-fgbgl
## 该pod是在服务创建后由ReplicaSet新建的，所以理论上它可以拿到在它创建之前创建的服务的相关环境变量
$ kubectl get po
NAME                                     READY   STATUS         RESTARTS   AGE
admin-console-66c55f8f89-vgqmw           1/1     Running        6          94d
admin-data-76bb5cbc9d-gflh9              1/1     Running        10         94d
admin-resource-server-754778f6ff-dxz9n   1/1     Running        6          94d
admin-server-7d4855d89-ds72h             1/1     Running        6          94d
asr-5b846fc649-xltjc                     1/1     Running        6          94d
asv-578949548f-mccgh                     1/1     Running        6          94d
dia-dcf98bc6c-crt42                      1/1     Running        6          94d
mongodb-primary-0                        1/1     Running        6          94d
mysql-master-0                           1/1     Running        6          94d
redis-master-0                           1/1     Running        6          94d
redis-rc-7brrr                           0/1     ErrImagePull   0          62s
redis-rc-ncdwr                           0/1     ErrImagePull   0          62s
redis-rc-xfkfj                           0/1     ErrImagePull   0          62s
vpr-5cc48467c7-fgbgl                     1/1     Running        0          42s
vpr-audio-784578ddcb-pzc4t               1/1     Running        6          94d
vprc-7748b6b6b4-8jhss                    1/1     Running        11         94d

## 在pod vpr-5cc48467c7-fgbgl中执行env命令查看环境变量，并使用服务名称过滤环境变量
## 可以看到这个pod确实可以获取到服务的相关环境变量
## 环境变量名的转换规则是将：所有横杠（-）换成下划线（_），全大写
$ kubectl exec vpr-5cc48467c7-fgbgl -- env | grep REDIS_SVC
REDIS_SVC_PORT=tcp://10.43.119.76:16379
REDIS_SVC_PORT_16379_TCP_PORT=16379
REDIS_SVC_SERVICE_PORT=16379
REDIS_SVC_PORT_16379_TCP_ADDR=10.43.119.76
REDIS_SVC_PORT_16379_TCP=tcp://10.43.119.76:16379
REDIS_SVC_PORT_16379_TCP_PROTO=tcp
REDIS_SVC_SERVICE_HOST=10.43.119.76
```

### 二、DNS发现

```shell
## 如何判断pod是否使用了k8s提供的dns服务，查看pod的yaml定义中spec.dnsPolicy
## 这说明我们挑选的语音识别的pod使用了k8s提供的dns服务
$ kubectl get po vpr-5cc48467c7-fgbgl --namespace=prod -o yaml
...
spec:
  dnsPolicy: ClusterFirst
...

## 查看pod容器的/etc/resolv.conf
## 可以看到DNS服务器的地址是10.43.0.10
$ kubectl exec vpr-5cc48467c7-fgbgl -- cat /etc/resolv.conf
nameserver 10.43.0.10
search prod.svc.cluster.local svc.cluster.local cluster.local
options ndots:5

## 找一下kube-system命名空间（Namespace）下的所有资源，看哪个资源的IP是10.43.0.10
## 结合这张学习的内容，pod要被内部或外部访问，依托服务，所以其实只要查看服务就可以了，没有必要查看all
## 可以看到service kube-dns的CLUSTER-IP为10.43.0.10
$ kubectl get all --namespace kube-system
...
NAME                     TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                  AGE
service/kube-dns         ClusterIP   10.43.0.10     <none>        53/UDP,53/TCP,9153/TCP   94d
service/metrics-server   ClusterIP   10.43.139.72   <none>        443/TCP                  94d
...

## 根据本章前面学到的内容，可以做一些推测
## 推测：服务背后应该有pod，只是通过这个服务暴露一个CLUSTER-IP，供集群内其他pod访问
## 目的：找到这个背后的pod
## 验证步骤：
## 1.查看service的定义，主要关注selector
$ kubectl get svc kube-dns --namespace=kube-system -o yaml
...
selector:
    k8s-app: kube-dns
...
## 2.查找有k8s-app=kube-dns标签的pod，确实有这么一个pod
$ kubectl get po --namespace=kube-system -l k8s-app=kube-dns
NAME                       READY   STATUS    RESTARTS   AGE
coredns-56b747df5f-r5ghv   1/1     Running   6          95d
```

### 三、FQDN发现

```shell
## 在pod的容器内部使用curl测试FQDN
## 我们使用prod命名空间（Namespace）下的redis-master-0这个pod，在这个pod下执行交互bash
$ kubectl exec -it redis-master-0 bash

## 先使用完整的FQDN
$ curl http://rancher.cattle-system.svc.cluster.local
<a href="https://rancher.cattle-system.svc.cluster.local/">Found</a>.

## 省略svc.cluster.local
$ curl http://rancher.cattle-system
<a href="https://rancher.cattle-system/">Found</a>.

## 由于redis-master-0和名为rancher的服务不在同一个命名空间（Namespace）下
## 所以无法省略cattle-system这个命名空间（Namespace）名称，如果在相同命名空间（Namespace），可以这样访问
## $ curl http://rancher

## 也可以不登录到pod的容器内部，使用--来执行需要在pod的容器内部执行的命令
## 先使用完整的FQDN
$ kubectl exec redis-master-0 -- curl http://rancher.cattle-system.svc.cluster.local
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    71  100    71    0     0  14552      0 --:--:-- --:--:-- --:--:-- 17750
<a href="https://rancher.cattle-system.svc.cluster.local/">Found</a>.

## 省略svc.cluster.local
$ kubectl exec redis-master-0 -- curl http://rancher.cattle-system
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    53  100    53    0     0  10408      0 --:--:-- --:--:-- --:--:-- 13250
<a href="https://rancher.cattle-system/">Found</a>.

## 由于redis-master-0和名为rancher的服务不在同一个命名空间（Namespace）下
## 所以无法省略cattle-system这个命名空间（Namespace）名称，如果在相同命名空间（Namespace），可以这样访问
## $ kubectl exec redis-master-0 -- curl http://rancher
```