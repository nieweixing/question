# 无法通过负载均衡类型ingress访问后端服务

## 问题现象

tke集群部署了一个nginx服务，并通过一个负载均衡类型的ingress暴露域名提供访问，但是通过域名无法访问到后端nginx服务。

## 排查思路

* 首先查看了ingress对应的clb监听后端是否正常，发现后端监听没有创建。

![upload-image](image/Snipaste_2021-11-05_18-38-47.JPG) 

* 没有成功创建监听说明ingress同步规则到clb失败，这时候查看ingress事件是否有报错，看了下事件也没有具体报错。

* 事件不能看出来，这时候再去查看下ingress-controller日志，对应的pod是kube-system下的l7-lb-controller，查看日志发现有报错，从下面报错基本可以确定问题了，后端的Service类型不匹配导致的，负载均衡类型ingress，需要后端service为odePort或者LoadBalancer才行。

```
sync ingress(coding01/nginx) error!err:Ingress Sync ClientError. ErrorCode: E4047 Details: Ingress: coding01/nginx. Service(coding01/nginx) is no able to be the backend of ingress. NodePort service or LoadBalancer service is support for ingress. OriginError: 
```

## 解决方案

将后端service类型从ClusterIP改成nodeport或者LoadBalancer。这个问题只会出现在用yaml创建ingress，如果是控制创建的ingress是无法选择后端ClusterIP类型的service。