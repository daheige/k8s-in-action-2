## Secret

与ConfigMap类似，存储是键值对映射。用法也类似，可以作为环境变量，也可以作为卷挂载。

secret只会存储在内存中，不会写入物理存储。

从`/root/test-yaml/secret/https`文件夹创建secret资源

```shell
$ kubectl create secret generic sec-nginx-https --from-file=/root/test-yaml/secret/https
```

**注意**：命令里指定了secret为`generic`，这样创建出来的secret，在yaml描述中，type为Opaque。还支持其他类型，如`docker-registry`。`docker-registry`类型的使用在这里省略。如果需要，可以回顾P227。

```shell
$ kubectl get secret sec-nginx-https -o yaml
apiVersion: v1
data:
  foo: YmFyCg==
  https.cert: LS0tLS1CRUdJTiBDRV...   ## 此处省略
  https.key: LS0tLS1CRUdJTiBSU0...    ## 此处省略
kind: Secret
metadata:
  creationTimestamp: "2020-07-20T10:53:35Z"
  name: sec-nginx-https
  namespace: prod
  resourceVersion: "37291655"
  selfLink: /api/v1/namespaces/prod/secrets/sec-nginx-https
  uid: 3bfdeb0d-b83d-4553-b12f-810851c21c01
type: Opaque                          ## 注意使用generic创建secret，type为
```

nginx配置文件my-nginx-config-https.conf文件示例

```nginx
server {
  listen              80;
  listen              443 ssl;
  server_name         www.kubia-example.com;
  ssl_certificate     certs/https.cert;          ## 读取证书的位置
  ssl_certificate_key certs/https.key;           ## 读取私钥的位置
  ssl_protocols       TLSv1 TLSv1.1 TLSV1.2;
  ssl_ciphers         HIGH:!aNULL:!MD5;
  location / {
    root   /usr/share/nginx/html;
    index  index.html index.htm;
  }
}
```

从`my-nginx-config-https.conf`文件创建ConfigMap资源

```shell
$ kubectl create cm cm-nginx-https --from-file=my-nginx-config-https.conf
```

pod示例文件pod-secret.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-secret
spec:
  containers:
  - image: art.sinovatio.com/public-docker/nginx:alpine
    name: nginx-1
    env:                                 ## 通过环境变量使用secret
    - name: FOO_SECRET                   ## 环境变量名，通过$FOO_SECRET使用
      valueFrom:
        secretKeyRef:
          name: sec-nginx-https          ## secret名称
          key: foo                       ## secret中的key
    volumeMounts:
    - name: config                       ## configMap卷名称
      mountPath: /etc/nginx/conf.d
      readOnly: true
    - name: certs                        ## secret卷名称
      mountPath: /etc/nginx/certs/
      readOnly: true
  volumes:
  - name: config                         ## configMap卷名称
    configMap:                           ## 指明是configMap卷
      name: cm-nginx-https               ## ConfigMap名称
      items:                             
      - key: my-nginx-config-https.conf  ## ConfigMap中的键
        path: https.conf                 ## 将cm cm-nginx-https中的key 挂载为容器里的https.conf文件
  - name: certs                          ## secret卷名称
    secret:                              ## 指明是secret卷
      secretName: sec-nginx-https        ## secret名称
```

**提示**：当pod一直处于创建中时，无法通过`kubectl logs pod-secret`查看日志，可以通过`kubectl describe pod-secret`查看下pod，注意最底部的Events，有些时候可以给出一些错误提示。

```shell
$ kubectl create -f pod-secret.yaml

## 当pod一直处于创建中时，无法通过kubectl logs pod-secret查看日志
## 可以通过kubectl describe pod-secret查看下pod，注意最底部的Events，有些时候可以给出一些错误提示
## 例如这里可以看到提示secret sc-nginx-secret不存在，其实是我们在pod的yaml中将secretName写错了
$ kubectl describe pod-secret
...
Events:
  Type     Reason       Age   From               Message
  ----     ------       ----  ----               -------
  Normal   Scheduled    69s   default-scheduler  Successfully assigned prod/pod-secret to voiceai
  Warning  FailedMount  5s    kubelet, voiceai   MountVolume.SetUp failed for volume "certs" : secret "sc-nginx-secret" not found
```

验证https在nginx中是否生效

```shell
kubectl port-forward pod-secret 8443:443 &
[1] 1185
[root@rhino79 secret]# Forwarding from 127.0.0.1:8443 -> 443

curl -kv https://localhost:8443

* About to connect() to localhost port 8443 (#0)
*   Trying 127.0.0.1...
* Connected to localhost (127.0.0.1) port 8443 (#0)
Handling connection for 8443
* Initializing NSS with certpath: sql:/etc/pki/nssdb
* skipping SSL peer certificate verification
* SSL connection using TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
* Server certificate:
* 	subject: CN=www.sinovatio.com
* 	start date: Jul 20 03:32:04 2020 GMT
* 	expire date: Jul 18 03:32:04 2030 GMT
* 	common name: www.sinovatio.com
* 	issuer: CN=www.sinovatio.com
> GET / HTTP/1.1
> User-Agent: curl/7.29.0
> Host: localhost:8443
> Accept: */*
> 
< HTTP/1.1 200 OK
< Server: nginx/1.19.1
< Date: Mon, 20 Jul 2020 06:21:36 GMT
< Content-Type: text/html
< Content-Length: 612
< Last-Modified: Tue, 07 Jul 2020 16:14:26 GMT
< Connection: keep-alive
< ETag: "5f049f62-264"
< Accept-Ranges: bytes
< 
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
* Connection #0 to host localhost left intact
```

验证secret存在在内存

```shell
## 由于使用的是tmpfs，存储在secret中的数据不会写入磁盘，也就无法被窃取。
$ kubectl exec pod-secret -c nginx-1 -- mount | grep certs
tmpfs on /etc/nginx/certs type tmpfs (ro,relatime)
```

通过环境变量暴露secret条目

```shell
## 在示例yaml中，我们使用了env.valueFrom.secretKeyRef字段
## 将secret sec-nginx-https中的foo键对应的值暴露为环境变量
$ kubectl exec pod-secret -- env | grep $FOO_SECRET
FOO_SECRET=bar

## 但是当我们使用echo输出这个环境变量时却发现没能按预期打印值。原因未知。
$ kubectl exec pod-secret -- echo $FOO_SECRET

## 现象和猜测
## 使用env输出不进行grep的时候可以发现FOO_SECRET下有一行空行
## 一开始以为只有FOO_SECRET前面的变量可以打印，后来发现后面的诸如HOME变量也可以打引，所以和空行可能无关
## FOO_SECRET也是合法的环境变量名，更换别的变量名如FOOBAR依然无效
$ kubectl exec -it pod-secret -- env
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=pod-secret
TERM=xterm
FOO_SECRET=bar

ASR_PORT_8970_TCP_ADDR=10.43.219.9
ADMIN_SERVER_PORT_8072_TCP_PORT=8072
...

## 新建一个全新的pod，不挂载任何secret卷和configMap卷，也有一些环境变量无法通过echo输出
## 如PKG_RELEASE，NJS_VERSION，KUBERNETES_PORT_443_TCP_PORT等
$ kubectl exec pod-test -- echo $PKG_RELEASE

$ kubectl exec pod-test -- echo $NJS_VERSION

$ kubectl exec pod-test -- echo $ADMIN_DATA_PORT_8860_TCP_ADDR

$ kubectl exec pod-test -- echo $KUBERNETES_PORT_443_TCP_PORT

## 怀疑可能和pod内容器的用户有关
```

