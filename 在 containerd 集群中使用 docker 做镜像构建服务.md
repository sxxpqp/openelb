# 在 containerd 集群中使用 docker 做镜像构建服务

## 背景

在 K8S 集群中，某些 CI/CD 流水线业务可能需要使用 docker 来提供镜像打包服务，一些同学会想到利用宿主机上的 docker，具体做法是把宿主机上 docker 的 unix socket (`/var/run/docker.sock`) 作为 hostPath 挂载到 CI/CD 的业务 Pod 中，之后在容器里通过 unix socket 来调用宿主机上的 docker 进行构建。这种方法固然简单，照比真正意义上的 Docker in Docker 甚至还能节省资源，但这种做法可能会遇到一些问题：

- 无法运行在 runtime 是 containerd 的集群中。
- 如果不加以控制可能会覆盖掉节点上已有的镜像。
- 在需要修改 docker daemon 配置文件的情况下，可能会影响到其他业务。
- 这种方式并不安全，特别是在多租户的场景下，当有特权的 Pod 拿到了 docker 的 unix socket 之后，Pod 中的容器不只可以调用宿主机的 docker 构建镜像，甚至可以删除已有镜像或容器，更有甚者可以通过 `docker exec` 接口操作其他容器。

第一个问题最近很多同学都会提到，因为 Kubernetes 在官方博客中宣布要在 1.22 版本之后弃用 docker，这之后可能部分同学就会把业务转投到 containerd 上。对于某些想用 containerd 集群，而不想改变 CI/CD 业务流程仍使用 docker 做构建构成一部分的话，可以通过在原有 Pod 上添加 dind 容器作为 sidecar 或者使用 DaemonSet 在节点上部署专门用来构建镜像的 docker 服务。

## 使用 DinD 作为 Pod 的 Sidecar

DinD（Docker in Docker）具体的原理可以参照 [DinD](https://hub.docker.com/_/docker)，下面的例子中是给 clean-ci 容器添加一个 sidecar，并通过 emptyDir 让 clean-ci 容器可以通过 unix socket 访问到 DinD 容器。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: clean-ci
spec:
  containers:
  - name: dind
    image: 'docker:stable-dind'
    command:
    - dockerd
    - --host=unix:///var/run/docker.sock
    - --host=tcp://0.0.0.0:8000
    securityContext:
      privileged: true
    volumeMounts:
    - mountPath: /var/run
      name: cache-dir
  - name: clean-ci
    image: 'docker:stable'
    command: ["/bin/sh"]
    args: ["-c", "docker info >/dev/null 2>&1; while [ $? -ne 0 ] ; do sleep 3; docker info >/dev/null 2>&1; done; docker pull library/busybox:latest; docker save -o busybox-latest.tar library/busybox:latest; docker rmi library/busybox:latest; while true; do sleep 86400; done"]
    volumeMounts:
    - mountPath: /var/run
      name: cache-dir
  volumes:
  - name: cache-dir
    emptyDir: {}
```

## 使用 DaemonSet 在每个 containerd 节点上部署 Docker

这种方式比较简单，直接在 containerd 集群中下发 DaemonSet 即可(挂载 hostPath)，当然不想污染节点上 `/var/run` 路径的同学们可以指定其他路径，使用下面的 yaml 部署来 DaemonSet:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: docker-ci
spec:
  selector:
    matchLabels:
      app: docker-ci
  template:
    metadata:
      labels:
        app: docker-ci
    spec:
      containers:
      - name: docker-ci
        image: 'docker:stable-dind'
        command:
        - dockerd
        - --host=unix:///var/run/docker.sock
        - --host=tcp://0.0.0.0:8000
        securityContext:
          privileged: true
        volumeMounts:
        - mountPath: /var/run
          name: host
      volumes:
      - name: host
        hostPath:
          path: /var/run
```

然后让业务 Pod 与 DaemonSet 共享同一个 hostPath，示例:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: clean-ci
spec:
  containers:
  - name: clean-ci
    image: 'docker:stable'
    command: ["/bin/sh"]
    args: ["-c", "docker info >/dev/null 2>&1; while [ $? -ne 0 ] ; do sleep 3; docker info >/dev/null 2>&1; done; docker pull library/busybox:latest; docker save -o busybox-latest.tar library/busybox:latest; docker rmi library/busybox:latest; while true; do sleep 86400; done"]
    volumeMounts:
    - mountPath: /var/run
      name: host
  volumes:
  - name: host
    hostPath:
      path: /var/run
```