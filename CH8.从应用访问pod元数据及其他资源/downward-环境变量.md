## Downward-环境变量

示例yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-downward-env
spec:
  containers:
  - name: busybox-1
    image: art.sinovatio.com/public-docker/busybox:latest
    command: ["sleep", "9999999"]     ## 运行一个长时间延迟的sleep命令来保持容器处于运行状态
    resources:
      requests:
        cpu: 15m
        memory: 100Ki
      limits:
        cpu: 1000m                    ## 资源上限可以适当放大，申请不到所需的资源会导致pod创建失败
        memory: 1000Mi                ## 资源上限可以适当放大，申请不到所需的资源会导致pod创建失败
    env:
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: POD_NAMESPACE
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace
    - name: POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
    - name: NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName
    - name: SERVICE_ACCOUNT
      valueFrom:
        fieldRef:
          fieldPath: spec.serviceAccountName
    - name: CONTAINER_CPU_REQUEST_MILLICORES
      valueFrom:
        resourceFieldRef:
          resource: requests.cpu
          divisor: 1m
    - name: CONTAINER_MEMORY_LIMIT_KIBIBYTES
      valueFrom:
        resourceFieldRef:
          resource: limits.memory
          divisor: 1Ki
```

验证环境变量

```shell
$ kubectl exec pod-downward-env -- env | grep POD_NAME
POD_NAME=pod-downward
POD_NAMESPACE=prod

$ kubectl exec pod-downward-env -- env | grep POD_IP
POD_IP=172.42.0.115

$ kubectl exec pod-downward-env -- env | grep NODE_NAME
NODE_NAME=voiceai

$ kubectl exec pod-downward-env -- env | grep SERVICE_ACCOUNT
SERVICE_ACCOUNT=default

$ kubectl exec pod-downward-env -- env | grep CONTAINER_CPU_REQUEST_MILLICORES
CONTAINER_CPU_REQUEST_MILLICORES=15

$ kubectl exec pod-downward-env -- env | grep CONTAINER_MEMORY_LIMIT_KIBIBYTES
CONTAINER_MEMORY_LIMIT_KIBIBYTES=1024000
```

**注意**：使用echo仍然无法输出这部分环境变量

**注意**：使用环境变量方式暴露标签和注解，一旦标签和注解被修改，pod内无法通过环境变量获得到新值