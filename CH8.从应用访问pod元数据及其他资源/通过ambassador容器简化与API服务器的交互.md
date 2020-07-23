## 通过ambassador容器简化与API服务器的交互

注意这个ambassador容器并不是什么官方容器，可能是作者自行创建的。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-ambassador
spec:
  containers:
  - name: redis-1
    image: bitnami/redis:5.0.3
    command: ["sleep", "9999999"] 
  - name: ambassador
    image: art.sinovatio.com/public-docker/luksa/kubectl-proxy:latest
```

创建并验证ambassador容器

```shell
## 创建pod
$ kubectl create -f pod-with-ambassador.yaml

## 进入pod内的redis容器（另外一个是ambassador容器）
$ kubectl exec -it pod-with-ambassador -c redis-1 bash

## 可以看到直接访问/目录还是被禁止的，无法像在节点那样获得所有REST API列表
I have no name!@pod-with-ambassador:/$ curl localhost:8001/
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {
    
  },
  "status": "Failure",
  "message": "forbidden: User \"system:serviceaccount:prod:default\" cannot get path \"/\"",
  "reason": "Forbidden",
  "details": {
    
  },
  "code": 403
}

## 尝试访问别的REST API接口可以获得正确相应
I have no name!@pod-with-ambassador:/$ curl localhost:8001/apis
```

查看ambassador容器内的内容

```shell
## 可以看到
## 根据PID，第一个启动的是/目录下的kubectl-proxy.sh
$ kubectl exec -it pod-with-ambassador -c ambassador -- ps
PID   USER     TIME   COMMAND
    1 root       0:00 /bin/sh -c /kubectl-proxy.sh
    6 root       0:00 {kubectl-proxy.s} /bin/sh /kubectl-proxy.sh
    8 root       0:00 /kubectl proxy --server=https://10.43.0.1:443 --certifica
  109 root       0:00 ps
  
## 查看下kubectl-proxy.sh的脚本内容
## 首先通过环境变量组装了API_SERVER，确定从pod内的容器访问k8s API服务器的地址
## 然后定义了CA_CERT，保存了ca证书的位置
## 接着定义了TOKEN，保存将token文件内容保存在该变量中
## 最后，和在节点物理机上一样，运行kubectl proxy，只不过要指定server、ca和token选项
$ kubectl exec -it pod-with-ambassador -c ambassador -- cat /kubectl-proxy.sh
#!/bin/sh

API_SERVER="https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT"
CA_CRT="/var/run/secrets/kubernetes.io/serviceaccount/ca.crt"
TOKEN="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)"

/kubectl proxy --server="$API_SERVER" --certificate-authority="$CA_CRT" --token="$TOKEN" --accept-paths='^.*'
```

所以说，这个ambassador镜像，只要按照这个步骤也是可以自己创建的。