## 持久卷（PV）&持久卷声明（PVC）

### hostPath类型的持久卷（pv）和持久卷声明（pvc）

通过pv和pvc的yaml文件，可以看到pv和pvc是没有任何强关联的。

创建pvc的时候会根据pvc的声明要求去寻找有没有符合条件的pv。

持久卷（pv）对应的yaml，pv-hostpath.yaml

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-hostpath
spec:
  capacity:
    storage: 100Mi
  accessModes:
  - ReadWriteOnce
  - ReadOnlyMany
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /tmp/k8s/pv
```

创建pv并查看

```shell
$ kubectl create -f pv-hostpath.yaml 
persistentvolume/pv-hostpath created

## 注意CLAIM和STORAGECLASS列没有任何值，说明还没有和pvc/storageClass关联
$ kubectl get pv
NAME         CAPACITY  ACCESS MODES  RECLAIM POLICY  STATUS     CLAIM  STORAGECLASS  AGE
pv-hostpath  100Mi     RWO,ROX       Retain          Available                       10s
```

持久卷声明（pvc）对应的yaml，pvc-hostpath.yaml

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-hostpath
spec:
  storageClassName: ""
  resources:
    requests:
      storage: 100Mi
  accessModes:
    - ReadWriteOnce
```

创建pvc并查看

```shell
$ kubectl create -f pvc-hostpath.yaml 
persistentvolumeclaim/pvc-hostpath created

## VOLUME列已经绑定了先前创建的pv
$ kubectl get pvc
NAME          STATUS  VOLUME       CAPACITY  ACCESS MODES  STORAGECLASS  AGE
pvc-hostpath  Bound   pv-hostpath  100Mi     RWO,ROX                     4s

## 再次查看pv，CLAIM列已经绑定了pvc
$ kubectl get pv
NAME         CAPACITY  ACCESS MODES  RECLAIM POLICY  STATUS  CLAIM              STORAGECLASS  AGE
pv-hostpath  100Mi     RWO,ROX       Retain          Bound   prod/pvc-hostpath                24s
```

在pod中使用持久卷声明（pvc）

示例pod-pvc-hostpath.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-pvc-hostpath
spec:
  containers:
  - image: bitnami/mongodb:4.0.6
    name: mongodb
    volumeMounts:
    - name: mongodb-data
      mountPath: /data/db
  volumes:
  - name: mongodb-data
    persistentVolumeClaim:        ## 使用pvc
      claimName: pvc-hostpath     ## pvc的yaml文件中metadata.name对应的值
```

验证删除pod和pvc后，pv的状态

```shell
$ kubectl delete po pod-pvc-hostpath
pod "pod-pvc-hostpath" deleted

$ kubectl delete pvc pvc-hostpath
persistentvolumeclaim "pvc-hostpath" deleted

## 注意删除pod和pvc后，pv的STATUS变成了Released状态，而不是一开始的Available状态
## 这是因为pv的RECLAIM POLICY为Retain，pvc被删除后，k8s会保留pv中的数据
## 也可以在pv的yaml中指定persistentVolumeReclaimPolicy为Delete或Recycle
$ kubectl get pv
NAME         CAPACITY  ACCESS MODES  RECLAIM POLICY  STATUS    CLAIM              STORAGECLASS  AGE
pv-hostpath  100Mi     RWO,ROX       Retain          Released  prod/pvc-hostpath                24m

## 再次创建pvc查看绑定状态
## 可以看到由于没有可用的符合条件的pv，pvc的VOLUME不会绑定
$ kubectl create -f pvc-hostpath.yaml 
persistentvolumeclaim/pvc-hostpath created
$ kubectl get pvc
NAME           STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-hostpath   Pending                                                     5s
```

验证`persistentVolumeReclaimPolicy`为`Delete`的pv

```shell
## 创建pv
$ kubectl create -f pv-hostpath.yaml 
persistentvolume/pv-hostpath created

## 创建pvc
$ kubectl create -f pvc-hostpath.yaml 
persistentvolumeclaim/pvc-hostpath created

## 创建pod
$ kubectl create -f pod-pvc-hostpath.yaml 
pod/pod-pvc-hostpath created

## 查看pod
$ kubectl get po pod-pvc-hostpath
NAME                                     READY   STATUS    RESTARTS   AGE
pod-pvc-hostpath                         1/1     Running   0          5s

## 查看pvc，已绑定pv
$ kubectl get pvc
NAME           STATUS   VOLUME        CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-hostpath   Bound    pv-hostpath   100Mi      RWO,ROX                       34s

## 查看pv，已绑定pvc
$ kubectl get pv
NAME         CAPACITY  ACCESS MODES  RECLAIM POLICY  STATUS  CLAIM              STORAGECLASS  AGE
pv-hostpath  100Mi     RWO,ROX       Delete          Bound   prod/pvc-hostpath                42s

## 删除pod
$ kubectl delete po pod-pvc-hostpath
pod "pod-pvc-hostpath" deleted

## 删除pvc
$ kubectl delete pvc pvc-hostpath
persistentvolumeclaim "pvc-hostpath" deleted

## 可以看到pv被删除了
$ kubectl get pv
No resources found in prod namespace.
```

经过测试，hostPath类型的卷不支持`persistentVolumeReclaimPolicy`为`Recycle`