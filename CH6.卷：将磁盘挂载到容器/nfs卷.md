## nfs卷

nfs卷需要先在别的机器上创建好nfs服务器，并共享出一个nfs目录。

创建使用nfs卷的pod的示例yaml，pod-nfs.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-nfs
spec:
  containers:
  - image: bitnami/mongodb:4.0.6
    name: mongodb-nfs
    volumeMounts:
    - name: mongodb-data           ## 卷名称
      mountPath: /data/db          ## pod内的容器挂载路径（pod内的容器访问路径）
  volumes:
  - name: mongodb-data             ## 卷名称
    nfs:                           ## 卷类型为nfs卷
      server: 192.168.41.213       ## nfs服务器地址（k8s所在服务器为192.168.42.79，跨网段）
      path: /home/data             ## nfs服务器提供的共享路径
```

和hostPath卷一样，使用nfs卷之前，也要在文件系统中配置好nfs卷的权限。否则pod内的容器没有权限进行读写操作。

验证不提前设置权限，pod内的容器无法往nfs卷写入内容，设置权限后，即可从pod内的容器往nfs卷写入。

```shell
## 从yaml文件创建pod
$ kubectl create -f pod-nfs.yaml 
pod/pod-nfs created

## 进入pod内的容器查看挂载的nfs卷
## 可以读取到nfs卷中的内容
$ kubectl exec -it pod-nfs bash
I have no name!@pod-nfs:/$ ls -l /data/db/ | grep nfs-volume
drwxrwxrwx  2 nobody nogroup      4096 Jul 16  2020 nfs-volume

## 尝试从pod内的容器写文件到nfs卷，提示无权限
I have no name!@pod-nfs:/$ echo nfs log > /data/db/nfs-volume/container.log
bash: /data/db/nfs-volume/container.log: Permission denied

## 在节点上更改该目录的权限
$ chmod -R 777 /mnt/nfs-volume/
$ ll /mnt/ | grep nfs-volume
drwxrwxrwx  2 nfsnobody nfsnobody      4096 Jul 16  2020 nfs-volume

## 进入pod内的容器查看对应目录下，再次尝试创建文件可以成功
$ kubectl exec -it pod-nfs bash
I have no name!@pod-nfs:/$ echo nfs log > /data/db/nfs-volume/container.log
```

