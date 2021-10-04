# pod采用组件挂载cfs报错

## 问题现象

pod通过cfs组件挂载cfs一直pending，pod起不来，查看事件报错MountVolume.MountDevice failed for volume "xxxxxxx" : kubernetes.io/csi: attacher.MountDevice failed to create newCsiDriverClient: driver name com.tencent.cloud.csi.cfs not found in the list of registered CSI drivers。

## 排查思路

1. 根据事件日志报错是因为cfs组件agent异常导致，检查cfs组件的agent是正常运行。
2. 查看事件发现有一条日志Starting pod sandbox eks-xxxxx，这个说明是调度到虚拟节点的，但是cfs组件的agent是csi-nodeplugin-cfsplugin(kube-system)，这个负载是DaemonSet类型，虚拟节点是无法运行DaemonSet，所以组件的agent无法在虚拟节点上运行。导致pod挂载cfs异常。

## 解决方案

将pod调度到正常节点上，不要调度到虚拟节点上，或者采用nfs的方式进行挂载。