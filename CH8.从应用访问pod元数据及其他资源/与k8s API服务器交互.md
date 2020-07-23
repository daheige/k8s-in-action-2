## 与k8s API服务器交互

### 通过proxy与k8s API服务器交互

```shell
$ kubectl cluster-info
Kubernetes master is running at https://192.168.42.79:6443
CoreDNS is running at https://192.168.42.79:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

## 使用curl访问这个https地址，会得到Unauthorized错误
$ curl -kv https://192.168.42.79:6443
* About to connect() to 192.168.42.79 port 6443 (#0)
*   Trying 192.168.42.79...
* Connected to 192.168.42.79 (192.168.42.79) port 6443 (#0)
* Initializing NSS with certpath: sql:/etc/pki/nssdb
* skipping SSL peer certificate verification
* NSS: client certificate not found (nickname not specified)
* SSL connection using TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
* Server certificate:
* 	subject: CN=kube-apiserver
* 	start date: Apr 03 11:17:53 2020 GMT
* 	expire date: Apr 25 03:28:28 2030 GMT
* 	common name: kube-apiserver
* 	issuer: CN=kube-ca
> GET / HTTP/1.1
> User-Agent: curl/7.29.0
> Host: 192.168.42.79:6443
> Accept: */*
> 
< HTTP/1.1 401 Unauthorized
< Content-Type: application/json
< Date: Wed, 22 Jul 2020 02:21:06 GMT
< Content-Length: 165
< 
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {
    
  },
  "status": "Failure",
  "message": "Unauthorized",
  "reason": "Unauthorized",
  "code": 401
* Connection #0 to host 192.168.42.79 left intact

## 这个时候可以启动一个本地proxy
$ kubectl proxy
Starting to serve on 127.0.0.1:8001

## 注意proxy是一直在前台运行的，所以需要重新打开一个shell窗口
## 然后使用curl访问本地proxy就可以得到所有API的REST地址
$ curl 127.0.0.1:8001
{
  "paths": [
    "/api",
    "/api/v1",
    "/apis",
    "/apis/",
    "/apis/acme.cert-manager.io",
    "/apis/acme.cert-manager.io/v1alpha2",
    "/apis/admissionregistration.k8s.io",
    "/apis/admissionregistration.k8s.io/v1",
    "/apis/admissionregistration.k8s.io/v1beta1",
    "/apis/apiextensions.k8s.io",
    "/apis/apiextensions.k8s.io/v1",
    "/apis/apiextensions.k8s.io/v1beta1",
    "/apis/apiregistration.k8s.io",
    "/apis/apiregistration.k8s.io/v1",
    "/apis/apiregistration.k8s.io/v1beta1",
    "/apis/apps",
    "/apis/apps/v1",
    "/apis/authentication.k8s.io",
    "/apis/authentication.k8s.io/v1",
    "/apis/authentication.k8s.io/v1beta1",
    "/apis/authorization.k8s.io",
    "/apis/authorization.k8s.io/v1",
    "/apis/authorization.k8s.io/v1beta1",
    "/apis/autoscaling",
    "/apis/autoscaling/v1",
    "/apis/autoscaling/v2beta1",
    "/apis/autoscaling/v2beta2",
    "/apis/batch",
    "/apis/batch/v1",
    "/apis/batch/v1beta1",
    "/apis/cert-manager.io",
    "/apis/cert-manager.io/v1alpha2",
    "/apis/certificates.k8s.io",
    "/apis/certificates.k8s.io/v1beta1",
    "/apis/coordination.k8s.io",
    "/apis/coordination.k8s.io/v1",
    "/apis/coordination.k8s.io/v1beta1",
    "/apis/crd.projectcalico.org",
    "/apis/crd.projectcalico.org/v1",
    "/apis/discovery.k8s.io",
    "/apis/discovery.k8s.io/v1beta1",
    "/apis/events.k8s.io",
    "/apis/events.k8s.io/v1beta1",
    "/apis/extensions",
    "/apis/extensions/v1beta1",
    "/apis/management.cattle.io",
    "/apis/management.cattle.io/v3",
    "/apis/metrics.k8s.io",
    "/apis/metrics.k8s.io/v1beta1",
    "/apis/monitoring.coreos.com",
    "/apis/monitoring.coreos.com/v1",
    "/apis/networking.k8s.io",
    "/apis/networking.k8s.io/v1",
    "/apis/networking.k8s.io/v1beta1",
    "/apis/node.k8s.io",
    "/apis/node.k8s.io/v1beta1",
    "/apis/policy",
    "/apis/policy/v1beta1",
    "/apis/project.cattle.io",
    "/apis/project.cattle.io/v3",
    "/apis/rbac.authorization.k8s.io",
    "/apis/rbac.authorization.k8s.io/v1",
    "/apis/rbac.authorization.k8s.io/v1beta1",
    "/apis/scheduling.k8s.io",
    "/apis/scheduling.k8s.io/v1",
    "/apis/scheduling.k8s.io/v1beta1",
    "/apis/storage.k8s.io",
    "/apis/storage.k8s.io/v1",
    "/apis/storage.k8s.io/v1beta1",
    "/healthz",
    "/healthz/autoregister-completion",
    "/healthz/etcd",
    "/healthz/log",
    "/healthz/ping",
    "/healthz/poststarthook/apiservice-openapi-controller",
    "/healthz/poststarthook/apiservice-registration-controller",
    "/healthz/poststarthook/apiservice-status-available-controller",
    "/healthz/poststarthook/bootstrap-controller",
    "/healthz/poststarthook/crd-informer-synced",
    "/healthz/poststarthook/generic-apiserver-start-informers",
    "/healthz/poststarthook/kube-apiserver-autoregistration",
    "/healthz/poststarthook/rbac/bootstrap-roles",
    "/healthz/poststarthook/scheduling/bootstrap-system-priority-classes",
    "/healthz/poststarthook/start-apiextensions-controllers",
    "/healthz/poststarthook/start-apiextensions-informers",
    "/healthz/poststarthook/start-cluster-authentication-info-controller",
    "/healthz/poststarthook/start-kube-aggregator-informers",
    "/healthz/poststarthook/start-kube-apiserver-admission-initializer",
    "/livez",
    "/livez/autoregister-completion",
    "/livez/etcd",
    "/livez/log",
    "/livez/ping",
    "/livez/poststarthook/apiservice-openapi-controller",
    "/livez/poststarthook/apiservice-registration-controller",
    "/livez/poststarthook/apiservice-status-available-controller",
    "/livez/poststarthook/bootstrap-controller",
    "/livez/poststarthook/crd-informer-synced",
    "/livez/poststarthook/generic-apiserver-start-informers",
    "/livez/poststarthook/kube-apiserver-autoregistration",
    "/livez/poststarthook/rbac/bootstrap-roles",
    "/livez/poststarthook/scheduling/bootstrap-system-priority-classes",
    "/livez/poststarthook/start-apiextensions-controllers",
    "/livez/poststarthook/start-apiextensions-informers",
    "/livez/poststarthook/start-cluster-authentication-info-controller",
    "/livez/poststarthook/start-kube-aggregator-informers",
    "/livez/poststarthook/start-kube-apiserver-admission-initializer",
    "/logs",
    "/metrics",
    "/openapi/v2",
    "/readyz",
    "/readyz/autoregister-completion",
    "/readyz/etcd",
    "/readyz/log",
    "/readyz/ping",
    "/readyz/poststarthook/apiservice-openapi-controller",
    "/readyz/poststarthook/apiservice-registration-controller",
    "/readyz/poststarthook/apiservice-status-available-controller",
    "/readyz/poststarthook/bootstrap-controller",
    "/readyz/poststarthook/crd-informer-synced",
    "/readyz/poststarthook/generic-apiserver-start-informers",
    "/readyz/poststarthook/kube-apiserver-autoregistration",
    "/readyz/poststarthook/rbac/bootstrap-roles",
    "/readyz/poststarthook/scheduling/bootstrap-system-priority-classes",
    "/readyz/poststarthook/start-apiextensions-controllers",
    "/readyz/poststarthook/start-apiextensions-informers",
    "/readyz/poststarthook/start-cluster-authentication-info-controller",
    "/readyz/poststarthook/start-kube-aggregator-informers",
    "/readyz/poststarthook/start-kube-apiserver-admission-initializer",
    "/readyz/shutdown",
    "/version"
  ]
}
```

### 研究batch API组的REST API

```shell
## 先将jobs API返回的数据保存到jobs.json文件中
## API会返回跨命名空间的所有job清单
$ curl localhost:8001/apis/batch/v1/jobs > jobs.json

## 查看所有namespace下的job，有四个
$ kubectl get jobs -A
NAMESPACE     NAME                                COMPLETIONS   DURATION   AGE
kube-system   rke-coredns-addon-deploy-job        1/1           3s         109d
kube-system   rke-ingress-controller-deploy-job   1/1           3s         109d
kube-system   rke-metrics-addon-deploy-job        1/1           3s         109d
kube-system   rke-network-plugin-deploy-job       1/1           2s         109d
```

到jobs.json文件中验证，`items`下也有4个同名job，部分信息省略。

```json
{
  "kind": "JobList",
  "apiVersion": "batch/v1",
  "metadata": {
    "selfLink": "/apis/batch/v1/jobs",
    "resourceVersion": "37975716"
  },
  "items": [
    {
      "metadata": {
        "name": "rke-coredns-addon-deploy-job",
        "namespace": "kube-system",
        "selfLink": "/apis/batch/v1/namespaces/kube-system/jobs/rke-coredns-addon-deploy-job",
        "uid": "5ea4b118-ec9f-40ba-b03d-6df454668682",
        "resourceVersion": "468",
        "creationTimestamp": "2020-04-03T11:31:13Z",
        "labels": {
          "controller-uid": "5ea4b118-ec9f-40ba-b03d-6df454668682",
          "job-name": "rke-coredns-addon-deploy-job"
        }
      }
    },
    {
      "metadata": {
        "name": "rke-ingress-controller-deploy-job",
        "namespace": "kube-system",
        "selfLink": "/apis/batch/v1/namespaces/kube-system/jobs/rke-ingress-controller-deploy-job",
        "uid": "6679cfde-a452-4f1e-9f61-966f932e8906",
        "resourceVersion": "626",
        "creationTimestamp": "2020-04-03T11:31:23Z",
        "labels": {
          "controller-uid": "6679cfde-a452-4f1e-9f61-966f932e8906",
          "job-name": "rke-ingress-controller-deploy-job"
        }
      }
    },
    {
      "metadata": {
        "name": "rke-metrics-addon-deploy-job",
        "namespace": "kube-system",
        "selfLink": "/apis/batch/v1/namespaces/kube-system/jobs/rke-metrics-addon-deploy-job",
        "uid": "27adb75e-1b78-4e98-82e4-5504cec9c504",
        "resourceVersion": "532",
        "creationTimestamp": "2020-04-03T11:31:18Z",
        "labels": {
          "controller-uid": "27adb75e-1b78-4e98-82e4-5504cec9c504",
          "job-name": "rke-metrics-addon-deploy-job"
        }
      }
    },
    {
      "metadata": {
        "name": "rke-network-plugin-deploy-job",
        "namespace": "kube-system",
        "selfLink": "/apis/batch/v1/namespaces/kube-system/jobs/rke-network-plugin-deploy-job",
        "uid": "441f5b1a-d1fa-4990-a2d9-856e2fd4c868",
        "resourceVersion": "383",
        "creationTimestamp": "2020-04-03T11:31:08Z",
        "labels": {
          "controller-uid": "441f5b1a-d1fa-4990-a2d9-856e2fd4c868",
          "job-name": "rke-network-plugin-deploy-job"
        }
      }
    }
  ]
}
```

### 获取特定job

```shell
## 格式
$ curl localhost:8001/apis/batch/v1/namespaces/<namespace-name>/jobs/<job-name>

## 示例
## 将通过REST API获取的内容保存到json文件
$ curl localhost:8001/apis/batch/v1/namespaces/kube-system/jobs/rke-network-plugin-deploy-job > rke-network-plugin-deploy-job.json

## 将通过kubectl get获取的内容也保存到json文件
$ kubectl get jobs rke-network-plugin-deploy-job -n kube-system -o json > get-job-rke-network-plugin-deploy-job.json

## 比较两个文件，可知，除了字段数序不一样外，基本没有差异
```

在pod内部与k8s API服务器通信

创建一个pod，示例yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-k8s-api
spec:
  containers:
  - name: redis-1
    image: bitnami/redis:5.0.3
    command: ["sleep", "9999999"]     ## 运行一个长时间延迟的sleep命令来保持容器处于运行状态
```



```shell
## 名为kubernetes的服务会在default命名空间暴露，并被配置为指向k8s API服务器
## 通过命令查看default命名空间下的服务
$ kubectl get svc -n default
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.43.0.1    <none>        443/TCP   109d

## 进入pod中的容器，并在容器内执行交互命令行
$ kubectl exec -it pod-k8s-api bash

## 可以通过curl命令访问FQDN
## 因为pod和svc不在同一个namespace下，所以在使用FQDN的时候，必须在服务名后加上namespace的名称
## 由于是https连接，还需要使用ca证书或在curl命令中使用-k选项绕开。
## 会返回Unauthorized错误
## 这是因为crul已经信任了k8s API服务器的证书，但k8s API服务器还不能确认访问者的身份，所以没有授权允许访问
I have no name!@pod-k8s-api:/$ curl -k https://kubernetes.default
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {
    
  },
  "status": "Failure",
  "message": "Unauthorized",
  "reason": "Unauthorized",
  "code": 401
}

## 正确的途径应该是使用ca证书
## 在学习secret章节的时候可知，ca证书，私钥，token作为一个secret卷中的三个key，是自动创建并挂载到容器的
## 通过查看pod可以看到，secret卷被挂载到pod内容器的/var/run/secrets/kubernetes.io/serviceaccount目录下
## 名称为default-token-xxxxx
$ kubectl get po pod-k8s-api -o yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-k8s-api
  namespace: prod
spec:
  containers:
    image: bitnami/redis:5.0.3
    name: redis-1
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: default-token-cdkm6
      readOnly: true
  volumes:
  - name: default-token-cdkm6
    secret:
      defaultMode: 420
      secretName: default-token-cdkm6

## 在pod内的容器中查看该路径下的内容可以看到ca证书，私钥和token
I have no name!@pod-k8s-api:/$ ls -l /var/run/secrets/kubernetes.io/serviceaccount
total 0
lrwxrwxrwx 1 root root 13 Jul 22 06:11 ca.crt -> ..data/ca.crt
lrwxrwxrwx 1 root root 16 Jul 22 06:11 namespace -> ..data/namespace
lrwxrwxrwx 1 root root 12 Jul 22 06:11 token -> ..data/token

## 使用ca证书访问https连接
## 一样会返回Unauthorized错误
I have no name!@pod-k8s-api:/$ curl --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt https://kubernetes.default    
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {
    
  },
  "status": "Failure",
  "message": "Unauthorized",
  "reason": "Unauthorized",
  "code": 401
}

## 也可以将ca证书的位置配置到环境变量CURL_CA_BUNDLE中
## 这样crul可以通过CURL_CA_BUNDLE找到ca证书，而无需在每次运行curl时都指定--cacert选项
I have no name!@pod-k8s-api:/$ export CURL_CA_BUNDLE=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

## 简化后的访问方式
I have no name!@pod-k8s-api:/$ curl https://kubernetes.default/apis

## k8s API服务器之所以返回了Unauthorized错误，是因为还不能确认访问者的身份，所以没有授权允许访问
## 可以通过在访问时带上secret卷中的token，来获得k8s API服务器的授权
## 首先将token导入为环境变量
I have no name!@pod-k8s-api:/$ TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
## 然后在curl请求中带上Authorization头，并将token内容作为头消息体发送给k8s API服务器
## 需要注意的是：
## 因为我们环境的k8s API服务器禁止直接请求/目录
## 所以无法像访问proxy那样获取全部REST API列表，只能访问具体的REST API接口
## 仍然查看batch这个REST API，与在节点物理机上使用curl访问proxy获得的响应一样
I have no name!@pod-k8s-api:/$ curl -H "Authorization: Bearer $TOKEN" https://kubernetes.default/apis/batch/v1
{
  "kind": "APIResourceList",
  "apiVersion": "v1",
  "groupVersion": "batch/v1",
  "resources": [
    {
      "name": "jobs",
      "singularName": "",
      "namespaced": true,
      "kind": "Job",
      "verbs": [
        "create",
        "delete",
        "deletecollection",
        "get",
        "list",
        "patch",
        "update",
        "watch"
      ],
      "categories": [
        "all"
      ],
      "storageVersionHash": "mudhfqk/qZY="
    },
    {
      "name": "jobs/status",
      "singularName": "",
      "namespaced": true,
      "kind": "Job",
      "verbs": [
        "get",
        "patch",
        "update"
      ]
    }
  ]
}
```

