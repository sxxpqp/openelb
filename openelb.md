## 第 1 步：为 kube-proxy 启用 strictARP

在第 2 层模式下，您需要为 kube-proxy 启用 strictARP，以便 Kubernetes 集群中的所有 NIC 停止响应来自其他 NIC 的 ARP 请求，而由 OpenELB 处理 ARP 请求。

1. 登录 Kubernetes 集群并运行以下命令来编辑 kube-proxy ConfigMap：

   ```bash
   kubectl edit configmap kube-proxy -n kube-system
   ```

2. 在 kube-proxy ConfigMap YAML 配置中，设置`data.config.conf.ipvs.strictARP`为`true`.

   ```yaml
   ipvs:
     strictARP: true
   ```

3. 运行以下命令重启 kube-proxy：

   ```bash
   kubectl rollout restart daemonset kube-proxy -n kube-system
   ```

## 步骤 2：指定用于 OpenELB 的 NIC

如果安装OpenELB的节点有多个网卡，则需要指定OpenELB在二层模式下使用的网卡。如果节点只有一个网卡，则可以跳过此步骤。

在这个例子中，安装了 OpenELB 的 master1 节点有两个网卡（eth0 192.168.0.2 和 eth1 192.168.1.2），并且 eth0 192.168.0.2 将用于 OpenELB。

运行以下命令注释master1以指定NIC：

```bash
kubectl annotate nodes master1 layer2.openelb.kubesphere.io/v1alpha1="192.168.0.2"
```





## 步骤 3 部署openelb组件

```
kubectl apply -f openelb.yaml
```



## 第 3 步：创建 Eip 对象

Eip 对象充当 OpenELB 的 IP 地址池。

1. 运行以下命令为 Eip 对象创建 YAML 文件：

   ```bash
   vi layer2-eip.yaml
   ```

2. 将以下信息添加到 YAML 文件中：

   ```yaml
   apiVersion: network.kubesphere.io/v1alpha2
   kind: Eip
   metadata:
     name: layer2-eip
   spec:
     address: 192.168.0.91-192.168.0.100
     interface: eth0
     protocol: layer2
   ```

   笔记

   - 中指定的IP地址`spec:address`必须与Kubernetes集群节点在同一网段。
   - 有关 Eip YAML 配置中的字段的详细信息，请参阅[使用 Eip 配置 IP 地址池](https://openelb.io/docs/getting-started/configuration/configure-ip-address-pools-using-eip/)。

3. 运行以下命令创建 Eip 对象：

   ```bash
   kubectl apply -f layer2-eip.yaml
   ```

## 第 4 步：创建部署

下面使用 luksa/kubia 镜像创建两个 Pod 的部署。每个 Pod 将自己的 Pod 名称返回给外部请求。

1. 运行以下命令为 Deployment 创建 YAML 文件：

   ```bash
   vi layer2-openelb.yaml
   ```

2. 将以下信息添加到 YAML 文件中：

   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: layer2-openelb
   spec:
     replicas: 2
     selector:
       matchLabels:
         app: layer2-openelb
     template:
       metadata:
         labels:
           app: layer2-openelb
       spec:
         containers:
           - image: luksa/kubia
             name: kubia
             ports:
               - containerPort: 8080
   ```

3. 运行以下命令来创建部署：

   ```bash
   kubectl apply -f layer2-openelb.yaml
   ```

## 第 5 步：创建服务

1. 运行以下命令为服务创建 YAML 文件：

   ```bash
   vi layer2-svc.yaml
   ```

2. 将以下信息添加到 YAML 文件中：

   ```yaml
   kind: Service
   apiVersion: v1
   metadata:
     name: layer2-svc
     annotations:
       lb.kubesphere.io/v1alpha1: openelb
       protocol.openelb.kubesphere.io/v1alpha1: layer2
       eip.openelb.kubesphere.io/v1alpha2: layer2-eip
   spec:
     selector:
       app: layer2-openelb
     type: LoadBalancer
     ports:
       - name: http
         port: 80
         targetPort: 8080
     externalTrafficPolicy: Cluster
   ```

   笔记

   - 您必须设置`spec:type`为`LoadBalancer`。
   - `lb.kubesphere.io/v1alpha1: openelb`注释指定服务使用 OpenELB 。
   - 该`protocol.openelb.kubesphere.io/v1alpha1: layer2`注释指定 OpenELB 用于第 2 层模式。
   - `eip.openelb.kubesphere.io/v1alpha2: layer2-eip`注解指定 OpenELB 使用的 Eip 对象。如果未配置此注解，OpenELB 会自动使用与协议匹配的第一个可用 Eip 对象。您也可以删除此注解并添加`spec:loadBalancerIP`字段（例如`spec:loadBalancerIP: 192.168.0.91`）以将特定 IP 地址分配给服务。
   - 如果`spec:externalTrafficPolicy`设置为`Cluster`（默认值），OpenELB 从所有 Kubernetes 集群节点中随机选择一个节点来处理 Service 请求。也可以通过 kube-proxy 访问其他节点上的 Pod。
   - 如果`spec:externalTrafficPolicy`设置为`Local`，OpenELB 会在 Kubernetes 集群中随机选择一个包含 Pod 的节点来处理 Service 请求。只能访问选定节点上的 Pod。

3. 运行以下命令来创建服务：

   ```bash
   kubectl apply -f layer2-svc.yaml
   ```

## 第 6 步：在第 2 层模式下验证 OpenELB

下面验证 OpenELB 是否正常运行。

1. 在 Kubernetes 集群中，运行以下命令获取 Service 的外部 IP 地址：

   ```bash
   root@master1:~# kubectl get svc
   NAME         TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)        AGE
   kubernetes   ClusterIP      10.233.0.1      <none>         443/TCP        20h
   layer2-svc   LoadBalancer   10.233.13.139   192.168.0.91   80:32658/TCP   14s
   ```

2. 在 Kubernetes 集群中，运行以下命令获取集群节点的 IP 地址：

   ```bash
   root@master1:~# kubectl get nodes -o wide
   NAME    		STATUS   ROLES		AGE		VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION       CONTAINER-RUNTIME
   master1   		Ready    master		20h		v1.17.9   192.168.0.2     <none>        Ubuntu 18.04.3 LTS   4.15.0-55-generic    docker://19.3.11
   worker-p001   	Ready    worker		20h		v1.17.9   192.168.0.3     <none>        Ubuntu 18.04.3 LTS   4.15.0-55-generic    docker://19.3.11
   worker-p002   	Ready    worker		20h		v1.17.9   192.168.0.4     <none>        Ubuntu 18.04.3 LTS   4.15.0-55-generic    docker://19.3.11
   ```

3. 在 Kubernetes 集群中，运行以下命令检查 Pod 的节点：

   ```bash
   root@master1:~# kubectl get pod -o wide
   NAME                              READY   STATUS    RESTARTS   AGE     IP             NODE    		NOMINATED NODE   READINESS GATES
   layer2-openelb-7b4fdf6f85-mnw5k   1/1     Running   0          3m27s   10.233.92.38   worker-p001   <none>           <none>
   layer2-openelb-7b4fdf6f85-px4sm   1/1     Running   0          3m26s   10.233.90.31   worker-p002   <none>           <none>
   ```

   笔记

   在此示例中，Pod 被自动分配给不同的节点。您可以手动[将 Pod 分配给不同的节点](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)。

4. 在客户端机器上，运行以下命令来 ping 服务 IP 地址并检查 IP 邻居：

   ```bash
   root@i-f3fozos0:~# ping 192.168.0.91 -c 4
   PING 192.168.0.91 (192.168.0.91) 56(84) bytes of data.
   64 bytes from 192.168.0.91: icmp_seq=1 ttl=64 time=0.162 ms
   64 bytes from 192.168.0.91: icmp_seq=2 ttl=64 time=0.119 ms
   64 bytes from 192.168.0.91: icmp_seq=3 ttl=64 time=0.145 ms
   64 bytes from 192.168.0.91: icmp_seq=4 ttl=64 time=0.123 ms
   
   --- 192.168.0.91 ping statistics ---
   4 packets transmitted, 4 received, 0% packet loss, time 3076ms
   rtt min/avg/max/mdev = 0.119/0.137/0.162/0.019 ms
   ```

   ```bash
   root@i-f3fozos0:~# ip neigh
   192.168.0.1 dev eth0 lladdr 02:54:22:99:ae:5d STALE
   192.168.0.2 dev eth0 lladdr 52:54:22:a3:9a:d9 STALE
   192.168.0.3 dev eth0 lladdr 52:54:22:3a:e6:6e STALE
   192.168.0.4 dev eth0 lladdr 52:54:22:37:6c:7b STALE
   192.168.0.91 dev eth0 lladdr 52:54:22:3a:e6:6e REACHABLE
   ```

   命令输出中`ip neigh`Service IP地址192.168.0.91的MAC地址与worker-p001 192.168.0.3的MAC地址相同。因此，OpenELB 已将 Service IP 地址映射到 worker-p001 的 MAC 地址。

5. 在客户端机器上，运行`curl`命令访问服务：

   ```bash
   curl 192.168.0.91
   ```

   如果`spec:externalTrafficPolicy`在[Service YAML 配置](https://openelb.io/docs/getting-started/usage/use-openelb-in-layer-2-mode/#step-5-create-a-service)中设置为`Cluster`，则可以访问两个 Pod。

   ```bash
   root@i-f3fozos0:~# curl 192.168.0.91
   You've hit layer2-openelb-7b4fdf6f85-px4sm
   root@i-f3fozos0:~# curl 192.168.0.91
   You've hit layer2-openelb-7b4fdf6f85-mnw5k
   
   root@i-f3fozos0:~# curl 192.168.0.91
   You've hit layer2-openelb-7b4fdf6f85-px4sm
   root@i-f3fozos0:~# curl 192.168.0.91
   You've hit layer2-openelb-7b4fdf6f85-mnw5k
   ```

   如果`spec:externalTrafficPolicy`在[Service YAML 配置](https://openelb.io/docs/getting-started/usage/use-openelb-in-layer-2-mode/#step-5-create-a-service)中设置为`Local`，则只能到达 OpenELB 选择的节点上的 Pod。

   ```bash
   root@i-f3fozos0:~# curl 192.168.0.91
   You've hit layer2-openelb-7b4fdf6f85-mnw5k
   
   root@i-f3fozos0:~# curl 192.168.0.91
   You've hit layer2-openelb-7b4fdf6f85-mnw5k
   
   root@i-f3fozos0:~# curl 192.168.0.91
   You've hit layer2-openelb-7b4fdf6f85-mnw5k
   ```