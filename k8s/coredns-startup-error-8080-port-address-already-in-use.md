#  coredns服务pod启动报错listen tcp :8080: bind: address already in use

## 问题现象

业务pod无法通过域名访问外网，在pod里面解析域名也失败，解析域名失败，应该是coredns服务出现了异常，检查coredns的pod，发现coredns的pod一直在重启，kubectl logs查看pod日志报错listen tcp :8080: bind: address already in use。

## 排查思路

从pod的启动日志报错来看，是端口被占用了，查看了下coredns的yaml，不是hostnetwork模式，正常容器里面怎么会出现监听端口被占用呢。怀疑是coredns的配置哪里有问题，检查了下coredns的configmaq配置，发现有在corefile里面自定义上游的dns配置。

```
a.com {
    forword . 1.1.1.1 2.2.2.2
    health 
}
```

从上面自定义的coredns配置看，有加上health这个插件，那么这个插件需要怎么配置呢？到官网文档查了下<https://coredns.io/plugins/health/>

发现文档里面有这个说明

If you have multiple Server Blocks, health can only be enabled in one of them (as it is process wide). If you really need multiple endpoints, you must run health endpoints on different ports:

```
com {
    whoami
    health :8080
}

net {
    erratic
    health :8081
}
```

从上面的说明看，如果需要使用health插件，需要指定检查端口才行，否则就用默认的8080，会导致端口被占用。

## 解决方案

既然知道问题原因了，这里解决方案有2个

1. 去掉health插件
2. 在health插件检查健康检查的端口配置，注意不能和coredns的8080冲突。