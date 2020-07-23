## configMap配置热更新

尝试修改ConfigMap，关闭gzip，看pod内的挂载的configMap卷里的配置能否更新。

```shell
$ kubectl edit cm nginx-cm
## 将gzip on该为gzip off后保存
configmap/nginx-cm edited

## 直接查看pod内的容器的配置文件
## 发现gzip仍然为打开状态
$ kubectl exec pod-vol -- cat /etc/nginx/conf.d/my-nginx-config.conf
server {
  listen              80;
  server_name         www.kubia-example.com;
  gzip on;                                     ## gzip仍然为on
  gzip_types text/plain application/xml;    
  location / {
    root   /usr/share/nginx/html;
    index  index.html index.htm;
  }
}

## 过一段时间（测试环境大约1分钟内）再查看发现gzip已经被关闭
$ kubectl exec pod-vol -- cat /etc/nginx/conf.d/my-nginx-config.conf
server {
  listen              80;
  server_name         www.kubia-example.com;
  gzip off;                                     ## gzip仍然为off                                  
  gzip_types text/plain application/xml;    
  location / {
    root   /usr/share/nginx/html;
    index  index.html index.htm;
  }
}

## 但是配置的变更是没法通知到nginx的，要nginx应用变更，需要使用nginx的命令
## -c 容器名称 在pod中存在多个容器时可以通过该选项进入指定的容器
$ kubectl exec pod-vol -c nginx-1 -- nginx -s reload

## 使用curl测试nginx的gzip是否关闭
## 先配置端口转发
$ kubectl port-forward pod-vol 8888:80 &
[1] 35606
$ Forwarding from 127.0.0.1:8888 -> 80

## 再使用curl进行测试，可以看到响应头里没有包含Content-Encoding: gzip
## 说明gzip已经被关闭
$ curl -H "Accept-Encoding: gzip" -I localhost:8888
Handling connection for 8888
HTTP/1.1 200 OK
Server: nginx/1.19.1
Date: Sat, 18 Jul 2020 07:35:00 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 07 Jul 2020 16:14:26 GMT
Connection: keep-alive
ETag: "5f049f62-264"
Accept-Ranges: bytes
```

**注意**：如果不是挂载的完整卷，而是通过items指定条目的configMap卷挂载，那自动更新不会发生。