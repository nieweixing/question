# k8s部署hostport模式pod报错端口冲突

最近有客户在使用hostport的模式部署deployment的时候，有出现这个错误0/4 nodes are available: 4 node(s) didn't have free ports for the requested pod ports，导致deployment部署失败，从报错看是端口冲突了。

因为hostport是用的占用的node节点的端口，既然是端口冲突，我们到节点上查看下对应端口是否有监听，登录节点用netstat查看对应端口的监听，发现并没有被占用，端口没有被占用，为什么会出现部署失败报错端口被占用的问题。

后面我根据报错的yaml进行测试下，具体现象如下

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: hostport-test
    qcloud-app: hostport-test
  name: hostport-test
  namespace: tke-test
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: hostport-test
      qcloud-app: hostport-test
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        k8s-app: hostport-test
        qcloud-app: hostport-test
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: hostport-test
        ports:
        - containerPort: 80
          hostPort: 80
          name: http
          protocol: TCP
        resources:
          limits:
            cpu: 500m
            memory: 1Gi
          requests:
            cpu: 250m
            memory: 256Mi
        securityContext:
          privileged: false
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      imagePullSecrets:
      - name: qcloudregistrykey
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
```

部署了hostPort的pod后，我们登录节点查看80端口的监听，发现并没有被监听

```
[root@VM-0-10-centos ~]# netstat -an | grep tcp | grep 80
tcp        0      0 0.0.0.0:30080           0.0.0.0:*               LISTEN
tcp        0      0 0.0.0.0:32580           0.0.0.0:*               LISTEN
tcp        0      0 10.0.0.10:46548         169.254.0.71:80         ESTABLISHED
tcp        0      0 10.0.0.10:33762         169.254.0.71:80         ESTABLISHED
```

接下来我们接着往hostport-test所在的节点部署一个hostport模式的pod看看

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: hostport-new
    qcloud-app: hostport-new
  name: hostport-new
  namespace: tke-test
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: hostport-new
      qcloud-app: hostport-new
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        k8s-app: hostport-new
        qcloud-app: hostport-new
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/hostname
                operator: In
                values:
                - 10.0.0.10
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: hostport-new
        ports:
        - containerPort: 80
          hostPort: 80
          name: http
          protocol: TCP
        resources:
          limits:
            cpu: 500m
            memory: 1Gi
          requests:
            cpu: 250m
            memory: 256Mi
        securityContext:
          privileged: false
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      hostNetwork: true
      imagePullSecrets:
      - name: qcloudregistrykey
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
```

我们看下hostport-new的事件，看下是否有出现报错

```
[root@VM-0-10-centos ~]# kubectl describe  pod -n tke-test hostport-new-86786bc46f-kl8ph
Name:           hostport-new-86786bc46f-kl8ph
Namespace:      tke-test
.................
Events:
  Type     Reason                        Age                   From                  Message
  ----     ------                        ----                  ----                  -------
  Normal   FailedSchedulingOnEkletNodes  7m49s (x16 over 12m)  admission-controller  node(s) didn't match node selector
  Warning  FailedScheduling              65s (x10 over 12m)    default-scheduler     0/5 nodes are available: 1 Too many pods, 2 node(s) didn't have free ports for the requested pod ports, 2 node(s) didn't match node selector.
```

从事件日志看，部署报错，也是报错端口没法使用，但是我们到节点查看80端口的确没有被监听，这是怎么回事呢？

hostport-test配置的hostport，但是没有配置hostNetwork，k8s的hostport，并不是直接在节点启动对应的监听端口，但是这里还是可以通过节点ip+端口进行访问，这是因为节点的iptables规则会将节点hostport的请求转发到pod上，所以到节点上查看80端口没有被监听，但是实际可以访问通。

所以hostport实际是生效的，如果你已经有pod通过hostport占用了端口，后续再配置同样的端口，则会提示端口冲突。

# 参考文档

https://cloud.tencent.com/developer/article/2008146
