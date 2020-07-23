## hostPath卷

hostPath卷是一种持久性存储。emptyDir和gitRepo不是，他们在pod删除时会被删除。而hostPath不会。

查看hostPath的描述

```shell
$ kubectl explain po.spec.volumes.hostPath
KIND:     Pod
VERSION:  v1

RESOURCE: hostPath <Object>

## 说明hostPath是将原先节点上已预先存在（pre-existing）的目录暴露给pod内的容器
## 所以要使用hostPath应当先在节点上创建好目录，并给定对应的权限
## 如果该目录不存在，那么创建pod的时候也会创建，但对应目录的权限仅为drwxr-xr-x，非root用户没有写入权限
DESCRIPTION:
     HostPath represents a pre-existing file or directory on the host machine
     that is directly exposed to the container. This is generally used for
     system agents or other privileged things that are allowed to see the host
     machine. Most containers will NOT need this. More info:
     https://kubernetes.io/docs/concepts/storage/volumes#hostpath

     Represents a host path mapped into a pod. Host path volumes do not support
     ownership management or SELinux relabeling.

FIELDS:
   path	<string> -required-
     Path of the directory on the host. If the path is a symlink, it will follow
     the link to the real path. More info:
     https://kubernetes.io/docs/concepts/storage/volumes#hostpath

   type	<string>
     Type for HostPath Volume Defaults to "" More info:
     https://kubernetes.io/docs/concepts/storage/volumes#hostpath
```

创建一下yaml文件，测试使用hostPath来读取节点磁盘上的内容

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-hostpath
spec:
  containers:
  - image: bitnami/mongodb:4.0.6
    name: mongodb-hostpath
    volumeMounts:
    - name: mongodb-data             ## 卷名称
      mountPath: /data/db            ## pod内的容器挂载卷的路径（pod内的容器访问的路径）
  volumes:
  - name: mongodb-data               ## 卷名称
    hostPath:                        ## 表明是hostPath类型的卷
      path: /tmp/mongodb             ## 节点的磁盘路径（节点物理机访问的路径）
```

演示创建pod时自动创建hostPath对应目录，但该目录权限受限

```shell
## 从文件创建
$ kubectl create -f pod-hostpath.yaml 
pod/pod-hostpath created

## /tmp/mongodb目录原始不存在，pod创建后会自动创建该目录
## 注意该文件的权限目前是不支持非root用户写操作的
$ ll /tmp/ | grep mongodb
drwxr-xr-x  2 root      root       6 Jul 16 16:30 mongodb

## 进入pod内的容器，查看对应目录下，并尝试写文件，提示权限受限
$ kubectl exec -it pod-hostpath bash
I have no name!@pod-hostpath:/$ ls -l /data/db/
total 0
I have no name!@pod-hostpath:/$ echo 'log content from pod container' > /data/db/container.log
bash: container.log: Permission denied

## 在节点上往hostPath对应路径写入文件
$ echo 'log content from pod container' > /tmp/mongodb/container.log

## 从pod内的容器读取该文件，内容与节点磁盘一致
I have no name!@pod-hostpath:/$ cat /data/db/container.log 
log content from pod container

## 在节点上更改该目录的权限
$ chmod -R 777 /tmp/mongodb
$ ll /tmp | grep mongodb
drwxrwxrwx  2 root      root      44 Jul 16 16:14 mongodb

## 进入pod内的容器查看对应目录下，再次尝试创建文件可以成功
$ kubectl exec -it pod-hostpath bash
I have no name!@pod-hostpath:/$ echo mongodb log from pod container > mongodb.log
```

演示创建pod前手动创建好hostPath对应目录并赋予对应权限，则pod内的容器可以直接使用

```shell
## 提前创建/tmp/predir文件夹并赋权
$ mkdir -p /tmp/predir
$ chmod -R 777 /tmp/predir/
$ ll /tmp/ | grep predir
drwxrwxrwx  2 root      root       6 Jul 16 16:39 predir

## 将pod的yaml文件中spec.volumes.hostPath.path修改为/tmp/predir
...
spec:
  volumes:
  - name: mongodb-data
    hostPath:
      path: /tmp/predir

## 创建pod
$ kubectl create -f pod-hostpath.yaml

## 进入pod内的容器并往hostPath对应挂载目录写入文件，写入成功
$ kubectl exec -it pod-hostpath bash
I have no name!@pod-hostpath:/$ echo 'log content from pod container' > /data/db/container.log
```

