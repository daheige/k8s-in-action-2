持久化卷（PersistentVolumes） 缩写PV

持久化卷声明（PersistentVolumeClaims）缩写PVC

创建持久卷，那应该有类似的kubectl create pv ...

创建pvc清单，那应该也有类似的kubectl create pvc ... 

pvc.yaml

找到pv并将pv绑定到pvc

pvc可以当作pod中的一个卷来使用，所以pod（k8s用户）关注的是pvc，pvc绑定了pv从而独占。

k8s管理员需要关注pv和真实的底层存储

StorageClass

## emptyDir卷

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-pod
spec:
  containers:
  - image: bitnami/redis:5.0.3
    name: redis-1
    volumeMounts:                      
    - name: sharelog                  ## 与volumes.name一致，卷名
      mountPath: /var/redis-log       ## 容器1内对应的路径
    env:                              ## 设置redis容器的环境变量
    - name: ALLOW_EMPTY_PASSWORD      ## redis需要密码，通过设置ALLOW_EMPTY_PASSWORD=yes可以跳过该验证
      value: "yes"
  - image: bitnami/mongodb:4.0.6
    name: mongodb-1
    volumeMounts:
    - name: sharelog                  ## 与volumes.name一致，卷名
      mountPath: /var/mongodb-log     ## 容器1内对应的路径
  volumes:                            ## 注意volumes的位置，说明卷（volume）是属于pod的
  - name: sharelog                    ## 一个名为sharelog的emptyDir卷
    emptyDir: {}
```

创建pod并验证两个容器内能否都访问到这个emptyDir卷

```shell
## 从文件创建pod
$ kubectl create -f pod-emptyDir.yaml 
pod/emptydir-pod created

## 查看pod
$ kubectl get po emptydir-pod
NAME                                     READY   STATUS    RESTARTS   AGE
emptydir-pod                             2/2     Running   0          5s

## 进入容器1 redis-1
$ kubectl exec -it emptydir-pod -c redis-1 bash

## 进入容器1内对应的路径，查看，没有任何文件
I have no name!@emptydir-pod:/$ cd /var/redis-log/
I have no name!@emptydir-pod:/var/redis-log$ ls -l
total 0

## 在容器1内对应路径创建redis.log文件，查看，redis.log文件创建成功
I have no name!@emptydir-pod:/var/redis-log$ echo redis log info > redis.log
I have no name!@emptydir-pod:/var/redis-log$ ls -l
total 4
-rw-r--r-- 1 1001 root 15 Jul 15 08:50 redis.log

## 退出容器1
I have no name!@emptydir-pod:/var/redis-log$ exit
exit

## 进入容器2内对应的路径，查看，能够看到容器1内创建的redis.log文件
$ kubectl exec -it emptydir-pod -c mongodb-1 bash
I have no name!@emptydir-pod:/$ cd /var/mongodb-log/
I have no name!@emptydir-pod:/var/mongodb-log$ ls -l
total 4
-rw-r--r-- 1 1001 root 15 Jul 15 08:50 redis.log

## 创建mongodb.log文件，查看，mongodb.log文件创建成功
I have no name!@emptydir-pod:/var/mongodb-log$ echo mongodb logs > mongodb.log
I have no name!@emptydir-pod:/var/mongodb-log$ ls -l
total 8
-rw-r--r-- 1 1001 root 13 Jul 15 08:51 mongodb.log
-rw-r--r-- 1 1001 root 15 Jul 15 08:50 redis.log

## 退出容器2
I have no name!@emptydir-pod:/var/mongodb-log$ exit
exit

## 再次进入容器1，也能查看到容器2内创建的文件
$ kubectl exec -it emptydir-pod -c redis-1 bash
I have no name!@emptydir-pod:/$ cd /var/redis-log/
I have no name!@emptydir-pod:/var/redis-log$ ls -l
total 8
-rw-r--r-- 1 1001 root 13 Jul 15 08:51 mongodb.log
-rw-r--r-- 1 1001 root 15 Jul 15 08:50 redis.log
```

emptyDir卷默认会存储在pod所在节点的磁盘上，查看emptyDir的位置

```shell
## 首先查看pod的uid
$ kubectl get po emptydir-pod -o yaml | grep uid
  uid: 19c3832a-17f5-436d-9e47-092e200ab0e0

## k8s相关资源存放在/var/lib/kubelet目录下
## pod相关的资源存放在/var/lib/kubelet/pod/<uid>目录下
$ cd /var/lib/kubelet/pods/19c3832a-17f5-436d-9e47-092e200ab0e0/volumes/kubernetes.io~empty-dir/log/

## 可以看到容器内创建的两个log文件
$ ll
total 8
-rw-r--r-- 1 rhino root 13 Jul 15 16:51 mongodb.log
-rw-r--r-- 1 rhino root 15 Jul 15 16:50 redis.log
```

删除pod时，对应的文件会被删除

```shell
## 删除pod
$ kubectl delete po emptydir-pod
pod "emptydir-pod" deleted

## 切换到k8s的pod目录
$ cd /var/lib/kubelet/pods/

## 先前uid对应的目录已不存在
$ ll | grep 19c3832a-17f5-436d-9e47-092e200ab0e0
```

可以配置emptyDir卷使用内存而不是磁盘

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-pod-mem
spec:
  containers:
  - image: bitnami/redis:5.0.3
    name: redis-1
    volumeMounts:                      
    - name: sharelog                  ## 与volumes.name一致，卷名
      mountPath: /var/redis-log       ## 容器1内对应的路径
    env:                              ## 设置redis容器的环境变量
    - name: ALLOW_EMPTY_PASSWORD      ## redis需要密码，通过设置ALLOW_EMPTY_PASSWORD=yes可以跳过该验证
      value: "yes"
  - image: bitnami/mongodb:4.0.6
    name: mongodb-1
    volumeMounts:
    - name: sharelog                  ## 与volumes.name一致，卷名
      mountPath: /var/mongodb-log     ## 容器1内对应的路径
  volumes:                            ## 注意volumes的位置，说明卷（volume）是属于pod的
  - name: sharelog                    ## 一个名为sharelog的emptyDir卷
    emptyDir: 
      medium: Memory                  ## emptyDir的文件将会存储在内存中
```

验证文件是否被创建到内存中

```shell
## 从文件创建
$ kubectl create -f pod-emptyDir-mem.yaml
pod/emptydir-pod-mem created

## 查看pod
$ kubectl get po emptydir-pod-mem
NAME                                     READY   STATUS    RESTARTS   AGE
emptydir-pod-mem                         2/2     Running   0          11s

## 首先查看pod的uid
$ kubectl get po emptydir-pod-mem -o yaml | grep uid
  uid: 558e52e6-9edd-4aa9-b138-7abda53b01a1

## 进入容器1内对应路径查看没有文件后创建redis日志文件并退出
$ kubectl exec -it emptydir-pod-mem -c redis-1 bash
I have no name!@emptydir-pod-mem:/$ ls -l /var/redis-log/
total 0
I have no name!@emptydir-pod-mem:/$ echo mem log > /var/redis-log/redis-mem.log
I have no name!@emptydir-pod-mem:/$ ls -l /var/redis-log/
total 4
-rw-r--r-- 1 1001 root 8 Jul 15 09:12 redis-mem.log
I have no name!@emptydir-pod-mem:/$ exit
exit

## 进入容器2内对应路径查看有redis日志文件后创建mongodb日志文件并退出
$ kubectl exec -it emptydir-pod-mem -c mongodb-1 bash
I have no name!@emptydir-pod-mem:/$ ls -l /var/mongodb-log/
total 4
-rw-r--r-- 1 1001 root 8 Jul 15 09:12 redis-mem.log
I have no name!@emptydir-pod-mem:/$ echo mongodb mem logs > /var/mongodb-log/mongodb.log
I have no name!@emptydir-pod-mem:/$ ls -l /var/mongodb-log/
total 8
-rw-r--r-- 1 1001 root 17 Jul 15 09:15 mongodb.log
-rw-r--r-- 1 1001 root  8 Jul 15 09:12 redis-mem.log
I have no name!@emptydir-pod-mem:/$ exit
exit

## 再次进入容器1内对应路径查看redis日志和mongodb日志均存在
$ kubectl exec -it emptydir-pod-mem -c redis-1 bash
I have no name!@emptydir-pod-mem:/$ ls -l /var/redis-log/
total 8
-rw-r--r-- 1 1001 root 17 Jul 15 09:15 mongodb.log
-rw-r--r-- 1 1001 root  8 Jul 15 09:12 redis-mem.log
I have no name!@emptydir-pod-mem:/$ exit
exit

## 退出容器查看对应磁盘位置，文件还是会存放到磁盘上，原因未知
$ ll /var/lib/kubelet/pods/558e52e6-9edd-4aa9-b138-7abda53b01a1/volumes/kubernetes.io~empty-dir/log/
total 8
-rw-r--r-- 1 rhino root 17 Jul 15 17:15 mongodb.log
-rw-r--r-- 1 rhino root  8 Jul 15 17:12 redis-mem.log
```

