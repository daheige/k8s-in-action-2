## 在pod中使用ConfigMap

一共有三种方式：

- 环境变量（valueFrom、envFrom）
- 命令行参数
- configMap卷。

该md展示**环境变量**方式。

### 一、测试pod引用的cm不存在时pod如何反应

pod示例yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-cm-notexist
spec:
  containers:
  - image: bitnami/redis:5.0.3
    name: redis-1
    env:
    - name: ALLOW_EMPTY_PASSWORD
      value: "yes"
    - name: INTERVAL
      valueFrom:
        configMapKeyRef:
          name: owls-cm-notexist    ## 该cm在创建pod时还不存在
          key: sleep.interval
```

cm示例yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: owls-cm-notexist
data:
  spring.app.name: owls
  country.code: "254"
  server.port: "8040"
  sleep.interval: "15"
```

测试创建pod

```shell
## 先创建pod
## 可以看到创建的pod失败了
$ kubectl get po
NAME                                     READY   STATUS                       RESTARTS   AGE
pod-cm-notexist                          0/1     CreateContainerConfigError   0          2s

## 创建cm后，过一段时间再次查看pod，可以看到失败pod自动重启成功，无需删除后重建
$ kubectl create -f cm-notexist.yaml

$ kubectl get po 
NAME                                     READY   STATUS    RESTARTS   AGE
pod-cm-notexist                          1/1     Running   0          115s
```

也可以将对cm的引用标记为可选的，示例yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-cm-notexist
spec:
  containers:
  - image: bitnami/redis:5.0.3
    name: redis-1
    env:
    - name: ALLOW_EMPTY_PASSWORD
      value: "yes"
    - name: INTERVAL
      valueFrom:
        configMapKeyRef:
          name: owls-cm-notexist    ## 该cm在创建pod时还不存在
          key: sleep.interval
          optional: true            ## 将对cm的依赖标记为可选的，即使cm不存在，pod也能创建成功
```

测试

```shell
## cm不存在的情况下创建pod
$ kubectl get cm
NAME              DATA   AGE
app-config-file   1      105d
vpr-audio-conf    1      105d

$ kubectl create -f pod-cm-optional.yaml 
pod/pod-cm-optional created

## pod创建成功
$ kubectl get po
NAME                                     READY   STATUS    RESTARTS   AGE
pod-cm-optional                          1/1     Running   0          4s
```

**总结**：当pod引用的cm不存在时，如果没有在pod中设置cm引用为可选，pod会创建失败；此时，将cm创建好，pod会自动重启成功，无需删除后重建。如果在pod中设置了cm引用为可选的，pod会创建成功。

### 二、使用valueFrom

valueFrom适用于设置具体的某一条或多条条目设置到pod的环境变量中

cm示例yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: owls-cm
data:
  spring.app.name: owls
  country.code: "254"
  server.port: "8040"
  sleep.interval: "15"
```

pod示例yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-env
spec:
  containers:
  - image: bitnami/redis:5.0.3
    name: redis-1
    env:
    - name: ALLOW_EMPTY_PASSWORD
      value: "yes"
    - name: INTERVAL              ## 环境变量名称
      valueFrom:                  ## 单独引用cm中的一个key，使用valueFrom
        configMapKeyRef:          
          name: owls-cm           ## cm的名称
          key: sleep.interval     ## cm中key的名称
```

创建pod并验证

```shell
## 创建cm
$ kubectl create -f cm.yaml 
configmap/owls-cm created

## 创建pod
$ kubectl create -f pod-cm.yaml 
pod/pod-cm created

## 验证通过cm设置的环境变量生效
$ kubectl exec -it pod-cm bash
I have no name!@pod-cm:/$ echo $INTERVAL
15
```



### 三、使用envFrom

envFrom适用于将所有条目设置到pod的环境变量中

修改cm的yaml如下

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: owls-cm-key
data:
  SPRING_APP_NAME: owls
  COUNTRY.CODE: "254"      ## 不合规的环境变量名称
  SERVER-PORT: "8040"      ## 不合规的环境变量名称
  sleep_interval: "15"
```

示例pod的yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-env-all
spec:
  containers:
  - image: bitnami/redis:5.0.3
    name: redis-1
    env:
    - name: ALLOW_EMPTY_PASSWORD
      value: "yes"
    envFrom:                  ## 引用真个cm，使用envFrom
    - prefix: CONFIG_         ## 给所有cm中的条目的键名加上一个CONFIG_的前缀
      configMapRef:
        name: owls-cm-key     ## cm的名称
```

创建pod并验证

```shell
$ kubectl create -f cm-key.yaml

$ kubectl create -f pod-env-all.yaml

$ kubectl exec -it pod-env-all bash

## 虽然可以看到环境变量设置成功，但有些名称无法作为合法的环境变量名，因此也无法引用
I have no name!@pod-env-all:/$ env | grep CONFIG
CONFIG_SPRING_APP_NAME=owls      ## 合法环境变量名
CONFIG_SERVER-PORT=8040          ## 非法环境变量名
CONFIG_COUNTRY.CODE=254          ## 非法环境变量名
CONFIG_sleep_interval=15         ## 合法环境变量名

## 合规的环境变量名称
I have no name!@pod-env-all:/$ echo $CONFIG_SPRING_APP_NAME
owls

## 不合规的环境变量名称
I have no name!@pod-env-all:/$ echo $CONFIG_COUNTRY.CODE
.CODE

## 不合规的环境变量名称
I have no name!@pod-env-all:/$ echo $CONFIG_SERVER-PORT
-PORT

## 合规的环境变量名称
I have no name!@pod-env-all:/$ echo $CONFIG_sleep_interval
15
```

