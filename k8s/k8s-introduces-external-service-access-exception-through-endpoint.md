# k8s通过endpoint引入外部服务访问异常

## 手动创建endpoint和svc引入集群外的nginx服务

首先通过yaml在集群创建ep和svc

```
kind: Endpoints
apiVersion: v1
metadata:
  # 此处 metadata.name 的值要和 service 中的 metadata.name 的值保持一致
  # endpoint 的名称必须和服务的名称相匹配
  name: out-of-cluster-nginx
  namespace: weixnie
subsets:
  - addresses:
      # 服务将连接重定向到 endpoint 的 IP 地址
      - ip: 172.16.0.4
    ports:
      # 外部服务端口
      # endpoint 的目标端口
      - port: 8081
        name: http-8081
---
apiVersion: v1
kind: Service
metadata:
  # 此处 metadata.name 的值要和 endpoints 中的 metadata.name 的值保持一致
  name: out-of-cluster-nginx
  # 外部服务服务统一在固定的名称空间中
  namespace: weixnie
spec:
  type: NodePort
  ports:
    - protocol: TCP
      port: 8081
      targetPort: 8081
      name: tcp-8081
```


# 测试集群内通过svc访问服务

```
bash-5.1# kubectl get svc out-of-cluster-nginx -n weixnie
NAME                   TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
out-of-cluster-nginx   NodePort   10.55.253.167   <none>        8081:30128/TCP   62s
bash-5.1# curl 10.55.253.167:8081
curl: (7) Failed to connect to 10.55.253.167 port 8081: Connection refused
bash-5.1#
```

这里svc的clusterip是10.55.253.167，但是通过ip却连不上nginx，这是为什么呢？

这里确认下nginx服务本身是否正常，直接访问集群外nginx是正常的

```
bash-5.1# curl 172.16.0.4:8081
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
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
```

## 问题分析

nginx服务是正常的，但是通过svc访问就不行，这个是为什么呢？这里首先查看下svc后端的ep是否正常

```
bash-5.1# kubectl describe svc out-of-cluster-nginx -n weixnie
Name:                     out-of-cluster-nginx
Namespace:                weixnie
Labels:                   <none>
Annotations:              <none>
Selector:                 <none>
Type:                     NodePort
IP Families:              <none>
IP:                       10.55.253.167
IPs:                      10.55.253.167
Port:                     tcp-8081  8081/TCP
TargetPort:               8081/TCP
NodePort:                 tcp-8081  30128/TCP
Endpoints:                
Session Affinity:         None
External Traffic Policy:  Cluster
Events:
  Type    Reason                Age    From                Message
  ----    ------                ----   ----                -------
  Normal  EnsuringService       3m19s  service-controller  Deleted Loadbalancer
  Normal  EnsureServiceSuccess  3m19s  service-controller  Service Sync Success. RetrunCode: S2000
```

可以发下svc的后端ep是空的，为什么是空的，是不是ep关联svc有问题，这里检查下yaml，名称是一致的，端口也是一样的，唯一的区别是port的名称不一样，难道是这个导致的？先我们修改下svc的port名称。

```
bash-5.1# kubectl describe svc out-of-cluster-nginx -n weixnie
Name:                     out-of-cluster-nginx
Namespace:                weixnie
Labels:                   <none>
Annotations:              <none>
Selector:                 <none>
Type:                     NodePort
IP Families:              <none>
IP:                       10.55.253.167
IPs:                      10.55.253.167
Port:                     http-8081  8081/TCP
TargetPort:               8081/TCP
NodePort:                 http-8081  30128/TCP
Endpoints:                172.16.0.4:8081
Session Affinity:         None
External Traffic Policy:  Cluster
Events:
  Type    Reason                Age    From                Message
  ----    ------                ----   ----                -------
  Normal  EnsuringService       9m47s  service-controller  Deleted Loadbalancer
  Normal  EnsureServiceSuccess  9m47s  service-controller  Service Sync Success. RetrunCode: S2000
```

将spec.port.name改成和endpoint一样的为http-8081，再次查看svc时候，后端ep就有了，这里我们通过svc访问下看看是否正常

```
bash-5.1# curl 10.55.253.167:8081
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
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
```

通过svc也访问正常了，看来问题就是ep的port名称和svc名称不一样导致的

## 结论

如果是通过endpoint引入外部服务到k8s集群内，需要注意保证endpoint和svc下面字段都一样

* metadata.name
* endpoint的subsets.ports.name和svc的spec.port.name