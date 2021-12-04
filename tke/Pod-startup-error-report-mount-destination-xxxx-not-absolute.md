# pod启动报错mount destination xxxx not absolute

## 问题现象

部署镜像到tke集群上，pod启动报错。

```
Error: failed to start container "volume-test": Error response from daemon: OCI runtime create failed: invalid mount {Destination:data/test Type:bind Source:/var/lib/docker/volumes/afb1908160a4aab18f754325b4df4968fcb4846cb4c9367c326d2e3f70d59f3b/_data Options:[rbind]}: mount destination data/test not absolute: unknown
```

对应的Dockerfile如下

```
FROM nginx
VOLUME data/test
```

## 排查思路

查看pod事件报错，提示是因为data/test不是绝对路径导致的，这里是dockerfile编写不规范导致的。


## 解决方案

将volume字段的路径改成绝对路径即可。