---
layout: post
title: 记一次 Kubernetes 集群 Pod Eviction 问题排查过程
category: 技术
---

现象：一个普通的 k8s 集群，3 个 worker node，k8s 版本 v1.19.0。发现 worker 1 上运行的某些 pod 被 evicted。pod 中的 `status.message` 显示：

![](/images/2021-06-03-k8s-pod-eviction/evicted_pod.png)

可以看到很多 pod 都是 Evicted 状态，并且错误信息也比较明确，The node was low on resource: ephemeral-storage，即 node 上的本地存储空间不足，导致 kubelet 触发了 eviction 流程，关于 Kubernetes 中的 Node-pressure Eviction 介绍参见[这里](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/)。

因此，对用户提出建议：

* 扩容，可以手动向集群增加一个 worker 或者部署 [cluster-autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler) 根据设定的条件自动扩容。当然，也可以只增加 worker 的本地存储空间，这里涉及虚拟机的 resize，会导致 worker node 暂时不可用。
* 针对一些关键 app，在其 manifest 中指定资源的 requests & limits，给 pod 赋予 Guaranteed QoS class，这样至少保证在 kubelet 触发 eviction 时尽量不影响这些 pod 的运行。参见[这里](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/)。

上述措施能够在一定程度上保证一些关键应用的可用性。但用户反馈说，eviction 流程是**周期性触发**。

我的第一个猜测是，worker 上某些容器不停的写日志，而又没有妥善配置 docker 的日志处理，所以是日志文件把存储填满了。

在不集成第三方日志管理工具的情况下，Kubernetes 中的 pod 日志是直接由 worker 上的容器 runtime 接管，用户集群用的是 docker，所以是由 docker 来处理容器日志。而 docker 提供的默认日志 driver 是 json-file，这个 driver 默认不提供 log rotation 功能，需要自己配置 (所以 docker 文档中推荐使用 local driver)。

检查了用户集群里的 docker 配置，

```
{
  "log-driver": "json-file",
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-opts": {
    "max-size": "50m"
  }
}
```

虽然使用的确实是 json-file，但这里配置了 max-size 来限制日志文件的大小，所以在 pod 数目稳定的情况下，理论上不会出现因为容器日志导致存储空间使用不断增加的情况。

难道是 pod 太多并且不停的在部署新 pod？于是，想跟踪检查一下 docker 的系统存储信息：

```
$ docker system df
TYPE                TOTAL               ACTIVE              SIZE                RECLAIMABLE
Images              23                  22                  4.198GB             493.2MB (11%)
Containers          66                  52                  253.9MB             9.066kB (0%)
Local Volumes       14                  13                  33.08GB             6.891MB (0%)
Build Cache         0                   0                   0B                  0B
```

一看吓一跳，发现 `Local Volumes` 竟然占用了 33G，`df -h` 显示该节点的本地存储空间总共只有 59G，看来确实是容器的本地存储占用了比较多的空间。那直接到 docker 存储目录(`/var/lib/docker/`)下按文件大小排序看看：

```
$ du -aSh /var/lib/docker/ 2>/dev/null | sort -h -r | head -n 10
22G     /var/lib/docker/volumes/c8c813e938fb5ea57f9033e0b7c49ed21edd5b273c99a4ef094fd7c71b881714/_data/log/prod.deprecations.log
22G     /var/lib/docker/volumes/c8c813e938fb5ea57f9033e0b7c49ed21edd5b273c99a4ef094fd7c71b881714/_data/log
9.8G    /var/lib/docker/volumes/3703bf4f269173f5e0fcb268135579d8eb1d22b4123a4f1249b13e18c2ef9add/_data/log/prod.deprecations.log
9.8G    /var/lib/docker/volumes/3703bf4f269173f5e0fcb268135579d8eb1d22b4123a4f1249b13e18c2ef9add/_data/log
448M    /var/lib/docker/overlay2/bfa437019c0d35783e65f21e218235e9d33a888d798a76c06633d2b53ce9c632/merged/usr/local/bin
448M    /var/lib/docker/overlay2/4181cdd990635ddb2edc0134c3f667c28315c2011af2291d4168fc9289e3cdff/diff/usr/local/bin
192M    /var/lib/docker/overlay2/129aed4465959cf137248cb1ad5564f8af21c9822283aa932c49b537a6a79432/merged/usr/bin
174M    /var/lib/docker/overlay2/db8e9aa188c8dfe6ca641e31e2a71680f03a3f22a74308fda14be74fda6407d1/diff/usr/bin
116M    /var/lib/docker/overlay2/bfa437019c0d35783e65f21e218235e9d33a888d798a76c06633d2b53ce9c632/merged/usr/local/bin/kube-apiserver
116M    /var/lib/docker/overlay2/4181cdd990635ddb2edc0134c3f667c28315c2011af2291d4168fc9289e3cdff/diff/usr/local/bin/kube-apiserver
```

确实看到了两个可疑的容器 volume。

但这里值得注意的是，占用大空间的并不是之前怀疑的容器日志文件，而是 docker local volume。

- docker 容器日志存储路径：`/var/lib/docker/containers/<container_id>/<container_id>-json.log`
- docker 容器 volume 存储路径：`/var/lib/docker/volumes/<volume_id>`
- docker 容器的可写层存储路径：`/var/lib/docker/overlay2/`

在 worker 节点上找到使用这个 docker volume 的容器：

```
$ docker ps -a --filter volume=c8c813e938fb5ea57f9033e0b7c49ed21edd5b273c99a4ef094fd7c71b881714
CONTAINER ID        IMAGE                                                  COMMAND                  CREATED             STATUS              PORTS               NAMES
7effca15b519        xxxxxxxxxxxxxxxxxxxxxxxxxx                            "docker-entrypoint p…"   44 hours ago        Up 44 hours                             k8s_api-php_production-api-php-67f675cd44-jdvnt_gdr-gear-24805605-production_02e5f2f4-1e0d-4384-9925-c74c151debff_0
```

从容器的名字可以倒推这个容器在 Kubernetes 中的 pod 名称、pod 所属的资源名字以及 namespace，命名规则参见下图：

![](/images/2021-06-03-k8s-pod-eviction/pod_naming.jpg)

所以 namespace 是 gdr-gear-24805605-production，deployment 名称是 production-api-php，container 名称是 api-php。

接下来我们要弄明白的是：这个 deployment 里的 pod 为什么使用了这么大的本地存储空间？

通过查看该 deployment 的 yaml spec (`kubectl -n gdr-gear-24805605-production get deploy production-api-php -o yaml`)，看到 pod 确实挂载了两个 PVC，但并没有使用 emptyDir 或者 hostPath (关于 emptyDir 是什么参见[这里](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir))，即这个 pod 的定义中并没有使用本地存储。

带着疑问，再次 google 了一下 `/var/lib/docker/volumes/` 这个路径 (因为毕竟占用空间的文件来自这个路径)，发现了Stack Overflow 上的[这个](https://stackoverflow.com/questions/53062547/docker-volume-vs-kubernetes-persistent-volume)问答，大致意思是，如果生成容器镜像的 Dockerfile 文件中包含 `VOLUME` 语句，并且不在 Kubernetes Pod 定义里映射这个 volume 路径的话，docker 就会在路径 `/var/lib/docker/volumes/` 下创建一个对应的目录。

至此，问题原因基本浮出水面，为了进一步确认，找用户检查了 Dockerfile 文件，果然有 `VOLUME` 定义并且 Pod spec 中并没有 volumeMounts 映射，这才导致了改 pod 不断的往这个路径下写数据，慢慢蚕食本地存储，迫使 kubelet 启动 eviction 流程。

---

我开通了知识星球(https://t.zsxq.com/MfMfqvJ)付费答疑，欢迎加入：

![](/images/2021-05-13-zhishixingqiu/1.png)