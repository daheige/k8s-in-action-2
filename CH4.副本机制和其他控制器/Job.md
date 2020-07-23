## Job

Job对应的yaml，job.yaml

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: redis-job
spec:
  completions: 5             ## 确保5个pod执行成功
  parallelism: 2             ## 最多2个pod可以并行
  activeDeadlineSeconds: 5   ## 限制pod时间，如果超过该事件将终止pod，并将job标记为失败
  backoffLimit: 6            ## 配置job被标记为失败之前可以重试的次数，默认为6
  template:
    metadata:
      labels:
        app: redis-job
    spec:
      restartPolicy: Never
      containers:
      - name: redis
        image: bitnami/redis
```

CronJob对应的yaml，cj.yaml

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: redis-job
spec:
  schedule: "0,15,30,45 * * * *"  ## 分 时 日 月 周，与Linux的Cron表达式一样
  startingDeadlineSeconds: 15     ## 最迟必须在预定时间后15秒开始运行
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            app: redis-job
        spec:
          restartPolicy: Never
          containers:
          - name: redis
            image: bitnami/redis
```

**注意**：任务必须是幂等的，多次运行不会得到不同的结果。