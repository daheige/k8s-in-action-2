基本跳过

控制平面（master节点）

- API服务器 
- Scheduler 
- Controller Manager 
- etcd分布式持久化存储

```shell
[root@rhino79 sts]# kubectl get componentstatuses
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok                  
controller-manager   Healthy   ok                  
etcd-0               Healthy   {"health":"true"}
```

工作节点

- kubelet注意与kubectl区别
- kube-proxy
- docker

客户端通过监听，API服务器通过通知（pub-sub）方式通知客户端资源变更。

调度器（scheduler）