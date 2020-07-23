## 通过configMap卷设置

### 完整configMap卷挂载

先创建configmap-files目录，并在该目录下创建nginx的配置文件my-nginx-config.conf文件如下：

```nginx
server {
  listen              80;
  server_name         www.kubia-example.com;
  gzip on;
  gzip_types text/plain application/xml;
  location / {
    root   /usr/share/nginx/html;
    index  index.html index.htm;
  }
}
```

另外一个配置文件sleep-interval仅包含一个25的字面值。

从configmap-files目录创建cm并查看

```shell
$ kubectl create cm nginx-cm --from-file=configmap-files

$ kubectl get cm nginx-cm -o yaml
apiVersion: v1
data:
  my-nginx-config.conf: |                ## 管道符号表示后续条目值是多行字面量
    server {                                         
      listen              80;                        
      server_name         www.kubia-example.com;     
      gzip on;                                       
      gzip_types text/plain application/xml;         
      location / {                                   
        root   /usr/share/nginx/html;                
        index  index.html index.htm;                 
      }                                              
    }     
  sleep-interval: |                      ## 管道符号表示后续条目值是多行字面量
    25
kind: ConfigMap
metadata:
  creationTimestamp: "2020-07-18T03:23:59Z"
  name: nginx-cm
  namespace: prod
  resourceVersion: "36347942"
  selfLink: /api/v1/namespaces/prod/configmaps/nginx-cm
  uid: 4590c228-572c-4b7e-98f4-3e81f97f66ef
```

pod示例yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-vol
spec:
  containers:
  - image: art.sinovatio.com/public-docker/nginx:alpine
    name: nginx-1
    volumeMounts:
    - name: config                   ## 卷名称
      mountPath: /etc/nginx/conf.d   ## pod内的容器挂载路径
      readOnly: true
  volumes:
  - name: config                     ## 卷名称
    configMap:                       ## 指明是一个configMap卷
      name: nginx-cm                 ## cm名称
```

测试挂载是否生效

```shell
$ kubectl create -f pod-vol.yaml

## 设置端口转发
$ kubectl port-forward pod-val 8888:80 &

## 通过curl测试nginx是否支持gzip压缩，如果支持说明挂载的配置文件生效了
$ curl -H "Accept-Encoding: gzip" -I localhost:8888
Handling connection for 8888
HTTP/1.1 200 OK
Server: nginx/1.19.1
Date: Sat, 18 Jul 2020 03:32:26 GMT
Content-Type: text/html
Last-Modified: Tue, 07 Jul 2020 16:14:26 GMT
Connection: keep-alive
ETag: W/"5f049f62-264"
Content-Encoding: gzip                                  ## 返回gzip说明设置成功

## 查看两个配置文件是否挂载到pod内的容器
## /etc/nginx/conf.d目录为pod内的容器挂载点
$ kubectl exec -it pod-vol -c nginx-1 -- ls -l /etc/nginx/conf.d
total 0
lrwxrwxrwx    1 root     root    27 Jul 18 03:30 my-nginx-config.conf -> ..data/my-nginx-config.conf
lrwxrwxrwx    1 root     root    21 Jul 18 03:30 sleep-interval -> ..data/sleep-interval
```

### 使用items指定条目的configMap卷挂载

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-vol-item
spec:
  containers:
  - image: art.sinovatio.com/public-docker/nginx:alpine
    name: nginx-1
    volumeMounts:
    - name: config                   ## 卷名称
      mountPath: /etc/nginx/conf.d   ## pod内的容器挂载路径
      readOnly: true
  volumes:
  - name: config                     ## 卷名称
    configMap:                       ## 指明是configMap卷
      name: nginx-cm                 ## cm名称
      items:
      - key: my-nginx-config.conf    ## configMap卷中的条目
        path: gzip.conf              ## 挂载到容器内的文件名称
```

查看pod内的容器配置文件

```shell
$ kubectl create -f pod-vol-item.yaml 
pod/pod-vol-item created

$ kubectl exec -it pod-vol-item -c nginx-1 -- ls -l /etc/nginx/conf.d
total 0
lrwxrwxrwx    1 root     root            16 Jul 18 06:00 gzip.conf -> ..data/gzip.conf
```

**注意**：configMap卷挂载某一文件夹会隐藏该文件夹中已存在的文件。在第一个示例中，`/etc/nginx/conf.d`目录下是`my-nginx-config.conf`文件和`sleep-interval`文件；在第二个示例中，`/etc/nginx/conf.d`目录下仅包含`gzip.conf`文件。我们可以创建一个不挂载任何configMap的nginx容器进行比较。

不挂载任何configMap卷的nginx，示例pod-nginx.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-nginx
spec:
  containers:
  - image: art.sinovatio.com/public-docker/nginx:alpine
    name: nginx-1
```

查看pod-nginx下的etc/nginx/conf.d文件夹下的内容

```shell
$ kubectl create -f pod-nginx.yaml 
pod/pod-nginx created

$ kubectl exec pod-nginx -c nginx-1 -- ls -l /etc/nginx/conf.d
total 4
-rw-r--r--    1 root     root          1114 Jul 18 06:11 default.conf
```

可以看到默认nginx的`/etc/nginx/conf.d`目录下应该包含`default.conf`文件，但是挂载configMap卷后（不论是完整挂载还是使用`items`进行具体条目挂载），这个文件都被隐藏了。

那么，假设我们要覆盖`/etc`目录下的`resolv.conf`文件，而`/etc`目录下有很多系统运行需要使用的重要文件，他们不能被隐藏，如何挂载configMap卷对应的文件到`/etc/resolv.conf`文件呢？可以通过`pod.spec.containers.volumeMounts.subPath`来配置。

### configMap独立条目作为文件被挂载且不隐藏文件夹中的其他文件

示例pod的yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-vol-item
spec:
  containers:
  - image: art.sinovatio.com/public-docker/nginx:alpine
    name: nginx-1
    volumeMounts:
    - name: config                   ## 卷名称
      mountPath: /etc/resolv.conf    ## pod内的容器挂载路径
      subPath: my-nginx-config.conf  ## 仅挂载指定的条目
  volumes:
  - name: config                     ## 卷名称
    configMap:                       ## 指明是configMap卷
      name: nginx-cm                 ## cm名称
```

验证是否可行

```shell
$ kubectl create -f pod-subpath.yaml 
pod/pod-subpath created

## 省略了一部分文件列表，但可以看到，/etc目录下的文件没有被隐藏，而且resolv.conf文件存在
$ kubectl exec pod-subpath -- ls -l /etc
total 140
drwxr-xr-x    2 root     root            17 Apr 23 06:25 crontabs
-rw-r--r--    1 root     root            89 Nov 29  2019 fstab
-rw-r--r--    1 root     root            12 Jul 18 06:29 hostname
-rw-r--r--    1 root     root           207 Jul 18 06:29 hosts
drwxr-xr-x    2 root     root            36 Jul 10 20:27 init.d
-rw-r--r--    1 root     root           570 Nov 29  2019 inittab
drwxr-xr-x    8 root     root           114 Apr 23 06:25 network
drwxr-xr-x    3 root     root          4096 Jul 10 20:27 nginx
drwxr-xr-x    2 root     root             6 Apr 23 06:25 opt
-rw-r--r--    1 root     root           238 Nov 29  2019 profile
drwxr-xr-x    2 root     root            38 Apr 23 06:25 profile.d
-rw-r--r--    1 root     root           258 Jul 18 06:29 resolv.conf
-rw-r--r--    1 root     root         36141 Nov 29  2019 services
-rw-r--r--    1 root     root            38 Nov 29  2019 shells
drwxr-xr-x    5 root     root           148 Apr 23 06:25 ssl
-rw-r--r--    1 root     root            53 Nov 29  2019 sysctl.conf
drwxr-xr-x    2 root     root            27 Apr 23 06:25 sysctl.d

## 查看resolv.conf文件，内容也是cm中的内容
$ kubectl exec pod-subpath -- cat /etc/resolv.conf
server {
  listen              80;
  server_name         www.kubia-example.com;
  gzip on;                                  
  gzip_types text/plain application/xml;    
  location / {
    root   /usr/share/nginx/html;
    index  index.html index.htm;
  }
}
```

**总结**：

| 方式    | 是否隐藏挂载点内容 | 是否具体到文件 |
| ------- | ------------------ | -------------- |
| items   | 是                 | 是             |
| subPath | 否                 | 是             |



### configMap卷中的文件权限

默认权限被设置为644（rw-r--r--），可以通过`pod.spec.containers.volumes.configMap.defaultMode`进行修改。

使用了defaultMode的yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-vol-defaultmode
spec:
  containers:
  - image: art.sinovatio.com/public-docker/nginx:alpine
    name: nginx-1
    volumeMounts:
      - name: config
        mountPath: /etc/nginx/conf.d
        readOnly: true
  volumes:
    - name: config
      configMap:
        name: nginx-cm
        defaultMode: 0666     ## 将两个配置文件均设置为666权限（rw-rw-rw-）
```

验证是否可以通过defaultMode更改configMap卷中的文件权限

```shell
$ kubectl create -f pod-vol-defaultmode.yaml

## 可以看到my-nginx-config.conf和sleep-interval的权限是777（rwxrwxrwx），并不是我们设置的666（rw-rw-rw-）
## 也不是默认的644（rw-r--r--）
$ kubectl exec pod-vol-defaultmode -- ls -l /etc/nginx/conf.d
total 0
lrwxrwxrwx    1 root     root    27 Jul 18 07:05 my-nginx-config.conf -> ..data/my-nginx-config.conf
lrwxrwxrwx    1 root     root    21 Jul 18 07:05 sleep-interval -> ..data/sleep-interval

## 但是注意这两个文件的属性是link（第一位是l，不是d（文件夹），也不是-（文件）），所以找原始文件看下
## 找到原始文件的目录为/etc/nginx/conf.d/..2020_07_18_07_05_27.299514810/
$ kubectl exec pod-vol-defaultmode -- find / -name sleep-interval
/etc/nginx/conf.d/..2020_07_18_07_05_27.299514810/sleep-interval
/etc/nginx/conf.d/sleep-interval

## 查看原始文件的权限确实如我们在defaultMode中设置的一样为666（rw-rw-rw-）
$ kubectl exec pod-vol-defaultmode -- ls -l /etc/nginx/conf.d/..2020_07_18_07_05_27.299514810/
total 8
-rw-rw-rw-    1 root     root           258 Jul 18 07:05 my-nginx-config.conf
-rw-rw-rw-    1 root     root             3 Jul 18 07:05 sleep-interval

## 再验证下没有使用defaultMode的pod，pod-vol中的文件权限是否为默认的644（rw-r--r--）
$ kubectl exec pod-vol -- ls -l /etc/nginx/conf.d
total 0
lrwxrwxrwx    1 root     root    27 Jul 18 03:30 my-nginx-config.conf -> ..data/my-nginx-config.conf
lrwxrwxrwx    1 root     root    21 Jul 18 03:30 sleep-interval -> ..data/sleep-interval

$ kubectl exec pod-vol -- find / -name sleep-interval
/etc/nginx/conf.d/..2020_07_18_03_30_28.984625473/sleep-interval
/etc/nginx/conf.d/sleep-interval

## 原始文件的权限确实为644（rw-r--r--）
$ kubectl exec pod-vol -- ls -l /etc/nginx/conf.d/..2020_07_18_03_30_28.984625473/
total 8
-rw-r--r--    1 root     root           258 Jul 18 03:30 my-nginx-config.conf
-rw-r--r--    1 root     root             3 Jul 18 03:30 sleep-interval
```

