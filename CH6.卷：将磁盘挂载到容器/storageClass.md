## 存储类（StorageClass）

StorageClass（sc）的作用在于创建pvc的时候，sc会根据pvc的声明要求自动创建pv。

云服务器都提供了对应的provisioner，而本地使用nfs的话需要自己安装nfs对应的provisioner。具体安装方法见参考资料。

### 安装nfs-clinet-provisionser并测试nfs-sc

准备nfs服务器

```shell
## nfs服务器需要这两个依赖，默认centos7一般都安装了这两个依赖
$ rpm -qa | grep nfs-utils
$ rpm -qa | grep rpcbind

## 如果没有这两个依赖，可以使用以下方式安装
$ yum install nfs-utils -y
$ yum install rpcbind -y

## 创建nfs服务端的共享目录并设置正确的权限，这样pod内地容器才能正常读写nfs卷中的内容
$ mkdir /home/data/nfsserver
$ chmod -R 777 /home/data/nfsserver

## 编辑/etc/exports文件
$ vim /etc/exports
## 添加如下条目后保存，将该目录共享给网络上的所有机器
/home/data/nfsserver *(rw,async,no_root_squash,no_subtree_check)
## 格式
## <export> <host1>(<options>) <hostN>(<options>)...
## 类似的写法
## 共享给所有192.168.41网段的机器
## /home/data/nfsserver 192.168.41.0/24(rw,sync)

## 查看nfs-server的状态
systemctl status nfs-server
## 如果没有启动，使用以下命令启动nfs-server
systemctl start nfs-server

## nfs服务端查看
$ showmount -e
Export list for centos7-web-213:
/home/data/nfsserver *

## nfs客户端查看
$ showmount -e 192.168.41.213
Export list for 192.168.41.213:
/home/data/nfsserver *

## 使用sc创建pv的话，nfs客户端也就是k8s节点不需要做什么
## 如果是挂在nfs卷使用的话，使用以下命令将213的nfs目录挂在到本地的/mnt目录
$ mount -t nfs 192.168.41.213:/home/data/nfsserver /mnt
```

创建namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: nfs
```

创建RBAC

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: nfs
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: nfs
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
    # replace with namespace where provisioner is deployed
  namespace: nfs
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  namespace: nfs
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: nfs
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io
```

创建deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  labels:
    app: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: nfs
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs-client-provisioner
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: quay.io/external_storage/nfs-client-provisioner:latest
          imagePullPolicy: IfNotPresent
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: fuseim.pri/ifs
            - name: NFS_SERVER
              value: 192.168.41.213
            - name: NFS_PATH
              value: /home/data/nfsserver
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.168.41.213
            path: /home/data/nfsserver
```

以上内容包含RBAC，deployment等还未学习到的内容，先假定他们能正常工作。

创建sc，示例storageclass.yaml

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-nfs-storage ## sc的名称，在pvc的yaml中会使用该名称
provisioner: fuseim.pri/ifs ## or choose another name, must match deployment's env PROVISIONER_NAME'
parameters:
  archiveOnDelete: "false"
```

测试sc

```shell
## 从文件按创建sc
$ kubectl create -f storageclass.yaml

## 查看sc
$ kubectl get sc
NAME                 PROVISIONER     RECLAIMPOLICY  VOLUMEBINDINGMODE  ALLOWVOLUMEEXPANSION  AGE
managed-nfs-storage  fuseim.pri/ifs  Delete         Immediate          false                 68m
```

修改pvc的yaml文件，示例pvc-nfs-sc.yaml

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-nfs-sc                       ## pvc的名称
spec:
  storageClassName: managed-nfs-storage  ## 这个是自定义的storageClass的名称，参考storageClass.yaml
  resources:
    requests:
      storage: 100Mi
  accessModes:
    - ReadWriteOnce
```

测试pvc

```shell
## 从文件创建pvc
$ kubectl create -f pvc-nfs-sc.yaml 
persistentvolumeclaim/pvc-nfs-sc created

## 查看pvc可以发现pvc已经绑定了pv
## pv的名称是自动生成的，使用的是pvc前缀+横杠+pvc的uid，这里为了对齐进行了省略
## 完成名称是pvc-b3fcbf73-3c08-42bc-a67b-b5808296d763
$ kubectl get pvc
NAME       STATUS VOLUME   CAPACITY  ACCESS MODES  STORAGECLASS         AGE
pvc-nfs-sc Bound  pvc-...  100Mi     RWO           managed-nfs-storage  3s

## 查看pv可以发现pv已经自动创建并绑定到pvc了
## pv的名称是自动生成的，使用的是pvc前缀+横杠+pvc的uid，这里为了对齐进行了省略
## 完成名称是pvc-b3fcbf73-3c08-42bc-a67b-b5808296d763
$ kubectl get pv
NAME     CAPACITY  ACCESS MODES  RECLAIM POLICY  STATUS  CLAIM            STORAGECLASS         AGE
pvc-...  100Mi     RWO           Delete          Bound   prod/pvc-nfs-sc  managed-nfs-storage  6s
```

修改pod的yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-pvc-nfs-sc
spec:
  containers:
  - image: bitnami/mongodb:4.0.6
    name: mongodb-nfs
    volumeMounts:
    - name: mongodb-data
      mountPath: /data/db
  volumes:
  - name: mongodb-data
    persistentVolumeClaim:
      claimName: pvc-nfs-sc       ## 修改pvc的名称为使用了storageClass的pvc
```

创建pod

```shell
## 从文件创建pod
$ kubectl create -f pod-pvc-nfs-sc.yaml 
pod/pod-pvc-nfs-sc created

## 查看pod状态
$ kubectl get po | grep pod-pvc-nfs-sc
NAME                                     READY   STATUS    RESTARTS   AGE
pod-pvc-nfs-sc                           1/1     Running   0          6s

## 进入pod内的容器并输出一个日志
$ kubectl exec -it pod-pvc-nfs-sc bash
I have no name!@pod-pvc-nfs-sc:/$ ls -l /data/db/
total 0
I have no name!@pod-pvc-nfs-sc:/$ echo contain log > /data/db/container.log
I have no name!@pod-pvc-nfs-sc:/$ ls -l /data/db/
total 4
-rw-r--r-- 1 1001 root 12 Jul 17  2020 container.log

## 切换到nfs服务器所在主机
## 可以看到该目录下自动创建了一个目录
## 命名规则为<namespace>-<pvc>-<pv>
## namespace: prod
## pvc: pvc-nfs-sc
## pv: pvc-b3fcbf73-3c08-42bc-a67b-b5808296d763
$ ls /home/data/nfsserver/
prod-pvc-nfs-sc-pvc-b3fcbf73-3c08-42bc-a67b-b5808296d763

## pod内的容器输出的日志文件在该目录下可见
$ ls /home/data/nfsserver/prod-pvc-nfs-sc-pvc-b3fcbf73-3c08-42bc-a67b-b5808296d763/
container.log
```

### 列出sc

```shell
$ kubectl get sc
```

### 检查默认sc

```yaml
## 由于我们是单节点的本地k8s，所以不像gce或aws有默认的sc
## 但是可以知道默认sc是由sc的yaml文件中的注解配置的
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  annotations:
    storageclass.beta.kubernetes.io/is-default-class: "true"
```

### 强制将pvc绑定到已预先配置的pv

```yaml
## 如果已经配置了符合pvc声明条件的pv，在pvc中将storageClassName设置为空字符串
## 则在创建pvc的时候会绑定到已配置好的pv上
## 但是，如果没有将storageClassName设置为空字符串
## 在创建pvc的时候，storageClass会忽略已经存在的pv，继续创建新的pv并绑定到pvc
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc
spec:
  storageClassName: ""
  resources:
    requests:
      storage: 100Mi
  accessModes:
    - ReadWriteOnce
```

### 参考：

[nfs-client-provisioner github 地址](https://github.com/kubernetes-incubator/external-storage/tree/master/nfs-client)