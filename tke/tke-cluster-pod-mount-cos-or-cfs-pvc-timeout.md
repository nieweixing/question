# TKE集群内pod挂载cos/cfs类型pvc超时

## 问题现象

cfs/cos类型的pvc卷挂载超时

报错内容：
timed out waiting for the condition; skipping pod

# 问题原因

* 业务负载中指定了 fsGroup，导致 kubelet 在完成 cos 的挂载后，会把挂载目录下所有文件进行一次权限修改，修改为 fsGroup 指定权限。

* cos 桶中文件过多，导致 kubelet 存储准备工作卡在了权限修改这一步。（用户的cos桶内有5万多文件）（[https://github.com/kubernetes/kubernetes/issues/69699](https://github.com/kubernetes/kubernetes/issues/69699) ）
* 后续旧 pod 的删除以及新 pod 的创建的存储相关工作都因上一步而卡住。cbs刷写速度较快，不容易出现该问题，cfs/cos这种会延时很厉害，容易超时

## 解决方案


有四种解决方案：

1. （推荐）选择文件数不多的子目录进行挂载，需要修改 pv 中的 path 参数。
2. （推荐）若可以保证每次修改文件权限一致的话，可以在 pod 中添加如下参数
   spec.securityContext.fsGroupChangePolicy: OnRootMismatch
   这样只要目录下文件权限已匹配，就不会去刷权限了。
   加上这个参数后，pod第一次启动还是会刷所有的文件权限，有超时问题。 但是后续pod重建就不需要刷权限，所以设置了该参数后如果仍有超时现象，请让用户耐心等待文件权限刷写完成
3. 修改cfs csidriver中fsGroupPolicy设置成非File,或者删除掉[None]。 这样无论客户是否有配置fsgroup 都会直接skip掉。 pod会立即启动
4. 当然，也可以不指定 fsGroup，这个是最简单的修复方式。
