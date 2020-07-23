## ConfigMap & Secret

配置应用程序

- 向容器传递命令行参数（**命令行参数**）
- 为每个容器设置自定义环境变量（**环境变量**）
- 通过特殊类型的卷将配置文件挂载到容器中（**特殊类型卷**）

### 为容器设置环境变量

新建使用环境变量的pod示例yaml

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
    - name: INTERVAL               ## 添加了另外一个环境变量INTERVAL
      value: "15"                  ## 环境变量的值是15
    - name: FIRST_VAR              ## 定义一个变量，以在第二个变量中引用它
      value: "foo"
    - name: LAST_VAR
      value: "$(FIRST_VAR) bar"    ## 第二个变量中使用$()引用第一个变量
```

创建pod并验证环境变量

```shell
## 从文件创建pod
$ kubectl create -f pod-env.yaml 
pod/pod-env created

## 进入pod内的容器验证，能够输出环境变量
$ kubectl exec -it pod-env bash
I have no name!@pod-env:/$ echo $INTERVAL
15

## 验证拼接的环境变量可以输出
I have no name!@pod-env:/$ echo $LAST_VAR
foo bar
```

### 使用ConfigMap

ConfigMap是一种资源。configMap卷和之前的emptyDir卷，hostPath卷，nfs卷一样，是一种卷资源。

使用ConfigMap的好处在于，可以在不同的namespace下创建同名的ConfigMap，这样在pod从开发测试环境部署到生产环境时，可以不用修改pod或pod中的应用，因为ConfigMap名字是一样的（举个例子，开发和生产环境存在ConfigMap中的数据库连接键值（value）是不一样的，但键名（key）是一样的）。

命令行创建cm

```shell
## 创建一个名为owls-cm-dev的configMap，包含一个style.default=white的键值对
$ kubectl create cm owls-cm-dev --from-literal=style.default=white
configmap/owls-cm-dev created

## 创建一个名为owls-cm-prod的configMap，包含一个server.port=8040的键值对和一个country.code=254的键值对
$ kubectl create cm owls-cm-prod --from-literal=server.port=8040 --from-literal=country.code=254
configmap/owls-cm-prod created
```

从yaml文件创建cm，示例yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: owls-cm           ## cm本身的名称
data:
  spring.app.name: owls   ## cm包含的键值对1
  country.code: "254"     ## cm包含的键值对2
  server.port: "8040"     ## cm包含的键值对3
```

从已有的配置文件创建cm

```shell
## 假定有以下配置文件app.properties
server.port=8040
country.code=254
spring.app.name=owls

## 从已有配置文件创建cm
$ kubectl create cm owls-cm-file --from-file=app.properties
configmap/owls-cm-file created

## 查看cm内容
$ kubectl get cm owls-cm-file -o yaml
apiVersion: v1
data:
  app.properties: |
    server.port=8040
    country.code=254
    spring.app.name=owls
kind: ConfigMap
metadata:
  creationTimestamp: "2020-07-17T08:47:08Z"
  name: owls-cm-file
  namespace: prod
  resourceVersion: "36027478"
  selfLink: /api/v1/namespaces/prod/configmaps/owls-cm-file
  uid: 66c50c0d-45fc-4ac9-b44c-9c4b56a4ba72
```

从已有的文件夹创建cm

```shell
## 假定有以下配置文件目录
conf
  |-- app.properties
  |-- dataserv.conf

## app.properties内容如下
server.port=8040
country.code=254
spring.app.name=owls

## dataserv.conf内容如下
dataservice.url=http://192.168.42.13:8899/dataservice
dataservice.url.new=http://192.168.42.13:8899/dataservice

## 从文件夹创建cm
$ kubectl create owls-cm-dir --from-file=conf

## 查看cm内容
$ kubectl get cm owls-cm-dir -o yaml
apiVersion: v1
data:
  app.properties: |
    server.port=8040
    country.code=254
    spring.app.name=owls
  dataserv.conf: |
    dataservice.url=http://192.168.42.13:8899/dataservice
    dataservice.url.new=http://192.168.42.13:8899/dataservice
kind: ConfigMap
metadata:
  creationTimestamp: "2020-07-17T08:51:16Z"
  name: owls-cm-dir
  namespace: prod
  resourceVersion: "36028667"
  selfLink: /api/v1/namespaces/prod/configmaps/owls-cm-dir
  uid: 79ec374f-665c-4d60-bed9-04f6d4a343b4
```

总结几种创建方式

```shell
$ kubectl create cm owls-cm-composite 
    --from-file=foo.json          ## 从配置文件创建，配置文件中的键值对都会放入cm
    --from-file=bar=foobar.conf   ## 自定义键名条目下的文件
    --from-file=config-opts/      ## 完整的文件夹
    --from-literal=some=thing     ## 字面量
```

