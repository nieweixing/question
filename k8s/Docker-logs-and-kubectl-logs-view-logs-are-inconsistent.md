# docker logs和kubectl logs查看日志不一致

## 问题现象

节点docker logs查看容器日志和命令kubectl logs查看日志有点不一致，kubectl logs查看的日志会少一部分。

## 排查思路

docker logs和kubectl logs查看日志都是查看/var/lib/docker/containers/[containerd_id]这个目录下的日志文件，既然是查看的结果不一致，说明是日志文件存在差异，进入这个目录查看了下

```
[root@node1 e1a3e821aa55c9a77419a4bbb063c084f337336e4c46c3008be6752fbb8f8554]# ll
total 906480
drwx------ 2 root root      4096 Nov  1 17:33 checkpoints
-rw------- 1 root root     16564 Nov  1 17:33 config.v2.json
-rw-r----- 1 root root  28125126 Nov  5 15:47 e1a3e821aa55c9a77419a4bbb063c084f337336e4c46c3008be6752fbb8f8554-json.log
-rw-r----- 1 root root 100000382 Nov  5 15:45 e1a3e821aa55c9a77419a4bbb063c084f337336e4c46c3008be6752fbb8f8554-json.log.1
-rw-r----- 1 root root 100000267 Nov  5 15:40 e1a3e821aa55c9a77419a4bbb063c084f337336e4c46c3008be6752fbb8f8554-json.log.2
-rw-r----- 1 root root 100000279 Nov  5 15:35 e1a3e821aa55c9a77419a4bbb063c084f337336e4c46c3008be6752fbb8f8554-json.log.3
-rw-r----- 1 root root 100000078 Nov  5 15:28 e1a3e821aa55c9a77419a4bbb063c084f337336e4c46c3008be6752fbb8f8554-json.log.4
-rw-r----- 1 root root 100000330 Nov  5 15:23 e1a3e821aa55c9a77419a4bbb063c084f337336e4c46c3008be6752fbb8f8554-json.log.5
-rw-r----- 1 root root 100000788 Nov  5 15:18 e1a3e821aa55c9a77419a4bbb063c084f337336e4c46c3008be6752fbb8f8554-json.log.6
-rw-r----- 1 root root 100000187 Nov  5 15:11 e1a3e821aa55c9a77419a4bbb063c084f337336e4c46c3008be6752fbb8f8554-json.log.7
-rw-r----- 1 root root 100000916 Nov  5 15:06 e1a3e821aa55c9a77419a4bbb063c084f337336e4c46c3008be6752fbb8f8554-json.log.8
-rw-r----- 1 root root 100003481 Nov  5 15:00 e1a3e821aa55c9a77419a4bbb063c084f337336e4c46c3008be6752fbb8f8554-json.log.9
-rw-r--r-- 1 root root      2336 Nov  1 17:33 hostconfig.json
drwx------ 2 root root      4096 Nov  1 17:33 mounts
```

从上面可以发现，容器的标准输出日志是有进行轮转的，这个是在/etc/docker/daemon.json的log-opts字段配置，既然kubectl logs查看的日志少了一部分，看了下kubectl logs日志最早时间是哪个，查看后发现和最早时间是e1a3e821aa55c9a77419a4bbb063c084f337336e4c46c3008be6752fbb8f8554-json.log这个文件的最开始的日志时间对应上，排查到这里，问题已经明确了，kubectl logs和docker logs查看的日志文件是不一样的，导致了日志有不同。

## 问题结论

kubectl logs查看的日志是xxxx-json.log这个日志文件的，而docker logs查看的是所有日志文件的日志，因此查看存在一定差异。