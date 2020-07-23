## 持久卷（PV）&持久卷声明（PVC）

### nfs类型的持久卷（pv）和持久卷声明（pvc）

pv示例yaml

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nfs
spec:
  capacity:                              ## 定义持久卷的大小
    storage: 100Mi
  accessModes:
  - ReadWriteOnce                        ## 被单个客户端挂载为读写模式
  - ReadOnlyMany                         ## 被多个客户端挂载为只读模式
  persistentVolumeReclaimPolicy: Retain  ## 如果pvc被释放了，pv保留。合法值为Retain/Delete/Recyle
  nfs:
    server: 192.168.41.213               ## nfs服务器的IP地址
    path: /home/data                     ## nfs服务器的共享目录路径
```

创建pv

```shell
$ kubectl create -f pv-nfs.yaml 
persistentvolume/pv-nfs created

## 查看pv，注意CLAIM列是空白的
$ kubectl get pv
NAME    CAPACITY  ACCESS MODES  RECLAIM POLICY  STATUS     CLAIM  STORAGECLASS  AGE
pv-nfs  100Mi     RWO,ROX       Retain          Available                       4s
```

pvc示例yaml

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-nfs
spec:
  resources:
    requests:
      storage: 100Mi
  accessModes:
  - ReadWriteOnce
  storageClassName: ""    
```

创建pvc

```shell
$ kubectl create -f pvc.yaml 
persistentvolumeclaim/pvc-nfs created

## 可以看到pvc的VOLUME列已经绑定了之前创建的pv
$ kubectl get pvc
NAME     STATUS  VOLUME  CAPACITY  ACCESS MODES  STORAGECLASS  AGE
pvc-nfs  Bound   pv-nfs  100Mi     RWO,ROX                     3s

## 再次查看pv，注意CLAIM列绑定了pvc
[root@rhino79 test-yaml]# kubectl get pv
NAME    CAPACITY  ACCESS MODES  RECLAIM POLICY  STATUS  CLAIM         STORAGECLASS AGE
pv-nfs  100Mi     RWO,ROX       Retain          Bound   prod/pvc-nfs               16m
```

在pod中使用持久卷声明（pvc）

示例pod-pvc-nfs.yaml

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
    - name: mongodb-data
      mountPath: /data/db
  volumes:
  - name: mongodb-data
    persistentVolumeClaim:
      claimName: pvc-nfs
```

当`persistentVolumeReclaimPolicy`为`Retain`时，删除pod和pvc后，pv的`STATUS`会变成`Released`

而`persistentVolumeReclaimPolicy`为`Recycle`和`Delete`时，删除pod和pvc后，pv的`STATUS`会变成`Failed`