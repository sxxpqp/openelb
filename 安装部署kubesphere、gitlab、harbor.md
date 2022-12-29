## k8s 部署使用教程
适合多节点的安装部署 

本次使用的ubuntu18.04安装部署的

主机名和ip对应关系

master 192.168.1.201

node1 192.168.1.202

node2 192.168.1.203

### 前提条件

* 部署linux系统 如：centos7 ubuntu18、20(LTS) suse等

* 安装并启动docker/containerd(1.20+安装containerd),已经安装了会重启docker/containerd. 高版本离线包自带docker/containerd，如没安装docker/containerd会自动安装.

* 默认安装是calio网络插件

* 下载kubernetes 离线安装包.

* 务必同步服务器时间

```
apt install ntpdate -y
```

```
ntpdate time.windows.com    
```

主机名不可重复 所有节点主机名 

```shell
hostnamectl set-hostname master
```

```
ntpdate time.windows.com    
```

```
apt install -y curl openssl tar socat conntrack
```

* master节点CPU必须2C以上
* 开启ssh远程连接账号及密码。如账号：root 密码：123456

### 步骤一 安装kebernetes

在master节点执行以下命令下载 KubeKey。

```
export KKZONE=cn
```

```
curl -sfL https://get-kk.kubesphere.io | VERSION=v2.2.1 sh -
```

```
chmod +x kk
```

```
./kk create config  --with-kubernetes v1.19.16 --with-kubesphere v3.2.0
```

```
vi config-sample.yaml
```

**示例配置文件： 需要添加addons**

```
apiVersion: kubekey.kubesphere.io/v1alpha2
kind: Cluster
metadata:
  name: sample
spec:
  hosts:
  - {name: master, address: 192.168.1.201, internalAddress: 192.168.1.201, user: edge, password: "1"}
  - {name: node1, address: 192.168.1.202, internalAddress: 192.168.1.202, user: edge, password: "1"}
  - {name: node2, address: 192.168.1.203, internalAddress: 192.168.1.203, user: edge, password: "1"}
  roleGroups:
    etcd:
    - master
    control-plane: 
    - master
    worker:
    - node1
    - node2
  controlPlaneEndpoint:
    ## Internal loadbalancer for apiservers 
    # internalLoadbalancer: haproxy

    domain: lb.kubesphere.local
    address: ""
    port: 6443
  kubernetes:
    version: v1.23.7
    clusterName: cluster.local
    autoRenewCerts: true
    containerManager: docker
  etcd:
    type: kubekey
  network:
    plugin: calico
    kubePodsCIDR: 10.233.64.0/18
    kubeServiceCIDR: 10.233.0.0/18
    ## multus support. https://github.com/k8snetworkplumbingwg/multus-cni
    multusCNI:
      enabled: false
  registry:
    privateRegistry: ""
    namespaceOverride: ""
    registryMirrors: []
    insecureRegistries: []
  addons:
  - name: nfs-client
    namespace: kube-system
    sources:
      chart:
        name: nfs-client-provisioner
        repo: https://charts.kubesphere.io/main
        valuesFile: /home/edge/nfs-client.yaml
```

#### 主机

请参照上方示例在 `hosts` 下列出您的所有机器并添加详细信息。

`name`：实例的主机名。

`address`：任务机和其他实例通过 SSH 相互连接所使用的 IP 地址。根据您的环境，可以是公有 IP 地址或私有 IP 地址。例如，一些云平台为每个实例提供一个公有 IP 地址，用于通过 SSH 访问。在这种情况下，您可以在该字段填入这个公有 IP 地址。

`internalAddress`：实例的私有 IP 地址。

此外，您必须提供用于连接至每台实例的登录信息，以下示例供您参考：

- 使用密码登录示例：

  ```
  hosts:
    - {name: master, address: 192.168.0.2, internalAddress: 192.168.0.2, port: 8022, user: ubuntu, password: Testing123}
  ```

  备注

  在本教程中，端口 `22` 是 SSH 的默认端口，因此您无需将它添加至该 YAML 文件中。否则，您需要在 IP 地址后添加对应端口号，如上所示。

- 默认 root 用户示例：

  ```
  hosts:
    - {name: master, address: 192.168.0.2, internalAddress: 192.168.0.2, password: Testing123}
  ```

- 使用 SSH 密钥的无密码登录示例：

  ```
  hosts:
    - {name: master, address: 192.168.0.2, internalAddress: 192.168.0.2, privateKeyPath: "~/.ssh/id_rsa"}
  ```

​     roleGroups

- `etcd`：etcd 节点名称   #请和主节点名称一样
- `control-plane`：主节点名称
- `worker`：工作节点名称

### 步骤二安装及配置 *NFS* 服务器

#### 步骤 1：安装 *NFS* 服务器 (*NFS* kernel server)

#### 若要设置服务器机器，就必须在机器上安装 *NFS* 服务器。

1. 运行以下命令，使用 Ubuntu 上的最新软件包进行安装。

   ```
   sudo apt-get update
   ```

2. 安装 *NFS* 服务器。

   ```
   sudo apt install nfs-kernel-server
   ```

#### 步骤 2：创建输出目录

*NFS* 客户端将在服务器机器上挂载一个目录，该目录已由 *NFS* 服务器输出。

1. 运行以下命令来指定挂载文件夹名称（例如，`/mnt/demo`）。

   ```
   sudo mkdir -p /mnt/demo
   ```

2. 出于演示目的，请移除该文件夹的限制性权限，这样所有客户端都可以访问该目录。

   ```
   sudo chown nobody:nogroup /mnt/demo
   ```

   ```
   sudo chmod 777 /mnt/demo
   ```

#### 步骤 3：授予客户端机器访问 *NFS* 服务器的权限

1. 运行以下命令：

   ```
   sudo nano /etc/exports
   ```

2. 将客户端信息添加到文件中。

   ```
   /mnt/demo clientIP(rw,sync,no_subtree_check)
   ```

   如果您有多台客户端机器，则可以将它们的客户端信息全部添加到文件中。或者，在文件中指定一个子网，以便该子网中的所有客户端都可以访问 *NFS* 服务器。例如：

   ```
   /mnt/demo *(rw,sync,no_subtree_check)
   ```

   备注

   - `rw`：读写操作。客户端机器拥有对存储卷的读写权限。
   - `sync`：更改将被写入磁盘和内存中。
   - `no_subtree_check`：防止子树检查，即禁用客户端挂载允许的子目录所需的安全验证。

3. 编辑完成后，请保存文件。

#### 步骤 4：应用配置

1. 运行以下命令输出共享目录。

   ```
   sudo exportfs -a
   ```

2. 重启 *NFS* 服务器。

   ```
   sudo systemctl restart nfs-kernel-server
   ```

### 步骤三配置客户端机器

请在所有客户端上(所有的节点上)安装 `nfs-common`，它提供必要的 NFS 功能，而无需安装其他服务器组件。

1. 执行以下命令确保使用最新软件包。

   ```
   sudo apt-get update
   ```

2. 在所有客户端上安装 `nfs-common`。

   ```
   sudo apt-get install nfs-common
   ```

3. 访问稍后想要下载 KubeKey 到其上的一台客户端机器（任务机）。创建一个配置文件，其中包含 NFS 服务器的全部必要参数，KubeKey 将在安装过程中引用该文件。master节点执行

   ```
   vi nfs-client.yaml   
   ```

   示例配置文件：

```
nfs:
  server: "192.168.0.2"    # nfs服务器的地址，请替换。
  path: "/mnt/demo"    # Replace the exported directory with your own.
storageClass:
  defaultClass: true
```

### 步骤四 使用配置文件创建集群

```
./kk create cluster -f config-sample.yaml
```

备注

如果使用其他名称，则需要将上面的 `config-sample.yaml` 更改为您自己的文件。

整个安装过程可能需要 10 到 30 分钟，具体取决于您的计算机和网络环境。

```
kubectl get node
```

```
NAME     STATUS   ROLES                  AGE   VERSION
master   Ready    control-plane,master   21h   v1.23.7
node1    Ready    worker                 21h   v1.23.7
node2    Ready    worker                 21h   v1.23.7
```

都出现ready表示正常

## harbor部署教程

### 步骤一 harbor部署教程

主机点执行

添加harbor仓库

```
helm repo add harbor https://helm.goharbor.io
```

```
helm pull harbor/harbor --version 1.7.2 --untar
```

查看harbor所有版本

```
 helm search repo harbor/harbor --versions
```

```
helm repo update
```

拉取指定版本chart包并解压缩

```
 helm pull harbor/harbor --version 1.7.2  --untar
```

```
cd harbor
```

```
vi values.yaml
```

修改自己对应值 有三处位置

```
expose:
  # Set the way how to expose the service. Set the type as "ingress",
  # "clusterIP", "nodePort" or "loadBalancer" and fill the information
  # in the corresponding section
  type: nodePort #修改为nodePort 注意大小写
  tls:
    # Enable the tls or not.
    # Delete the "ssl-redirect" annotations in "expose.ingress.annotations" when TLS is disabled and "expose.type" is "ingress"
    # Note: if the "expose.type" is "ingress" and the tls
    # is disabled, the port must be included in the command when pull/push
    # images. Refer to https://github.com/goharbor/harbor/issues/5291
    # for the detail.
    enabled: false #修改为false 关系tsl
    # The source of the tls certificate. Set it as "auto", "secret"
    # or "none" and fill the information in the corresponding section
    # 1) auto: generate the tls certificate automatically
    # 2) secret: read the tls certificate from the specified secret.
    # The tls certificate can be generated manually or by cert manager
    # 3) none: configure no tls certificate for the ingress. If the default
    # tls certificate is configured in the ingress controller, choose this option
    certSource: auto
    auto:
      # The common name used to generate the certificate, it's necessary
      # when the type isn't "ingress"
      commonName: ""
    secret:
      # The name of secret which contains keys named:
      # "tls.crt" - the certificate
      # "tls.key" - the private key
      secretName: ""
      # The name of secret which contains keys named:
      # "tls.crt" - the certificate
      # "tls.key" - the private key
      # Only needed when the "expose.type" is "ingress".
      notarySecretName: ""
  ingress:
    hosts:
      core: core.harbor.domain
      notary: notary.harbor.domain
    # set to the type of ingress controller if it has specific requirements.
    # leave as `default` for most ingress controllers.
    # set to `gce` if using the GCE ingress controller
    # set to `ncp` if using the NCP (NSX-T Container Plugin) ingress controller
    controller: default
    annotations:
      # note different ingress controllers may require a different ssl-redirect annotation
      # for Envoy, use ingress.kubernetes.io/force-ssl-redirect: "true" and remove the nginx lines below
      ingress.kubernetes.io/ssl-redirect: "true"
      ingress.kubernetes.io/proxy-body-size: "0"
      nginx.ingress.kubernetes.io/ssl-redirect: "true"
      nginx.ingress.kubernetes.io/proxy-body-size: "0"
    notary:
      # notary-specific annotations
      annotations: {}
    harbor:
      # harbor ingress-specific annotations
      annotations: {}
  clusterIP:
    # The name of ClusterIP service
    name: harbor
    # Annotations on the ClusterIP service
    annotations: {}
    ports:
      # The service port Harbor listens on when serving with HTTP
      httpPort: 80
      # The service port Harbor listens on when serving with HTTPS
      httpsPort: 443
      # The service port Notary listens on. Only needed when notary.enabled
      # is set to true
      notaryPort: 4443
  nodePort:
    # The name of NodePort service
    name: harbor
    ports:
      http:
        # The service port Harbor listens on when serving with HTTP
        port: 80
        # The node port Harbor listens on when serving with HTTP
        nodePort: 30002
      https:
        # The service port Harbor listens on when serving with HTTPS
        port: 443
        # The node port Harbor listens on when serving with HTTPS
        nodePort: 30003
      # Only needed when notary.enabled is set to true
      notary:
        # The service port Notary listens on
        port: 4443
        # The node port Notary listens on
        nodePort: 30004
  loadBalancer:
    # The name of LoadBalancer service
    name: harbor
    # Set the IP if the LoadBalancer supports assigning IP
    IP: ""
    ports:
      # The service port Harbor listens on when serving with HTTP
      httpPort: 80
      # The service port Harbor listens on when serving with HTTPS
      httpsPort: 443
      # The service port Notary listens on. Only needed when notary.enabled
      # is set to true
      notaryPort: 4443
    annotations: {}
    sourceRanges: []

# The external URL for Harbor core service. It is used to
# 1) populate the docker/helm commands showed on portal
# 2) populate the token service URL returned to docker/notary client
#
# Format: protocol://domain[:port]. Usually:
# 1) if "expose.type" is "ingress", the "domain" should be
# the value of "expose.ingress.hosts.core"
# 2) if "expose.type" is "clusterIP", the "domain" should be
# the value of "expose.clusterIP.name"
# 3) if "expose.type" is "nodePort", the "domain" should be
# the IP address of k8s node
#
# If Harbor is deployed behind the proxy, set it as the URL of proxy
externalURL: http://zkturing.imwork.net:30002    #修改自己出口对应域名或者ip   需要出口ip地址映射到master或node节点端口为30002
```

创建harbor命名空间

```
kubectl create ns harbor
```

安装harbor

```
helm install harbor -f values.yaml . -n harbor
```

查看harbor的pod是否在运行

```
kubectl get  pod  -w -n harbor
```

全部 Running为成功安装

浏览器访问 http://zkturing.imwork.net:30002

默认账号为admin 密码：Harbor12345

### 步骤二：修改节点docker配置

请在所有的节点上执行

```
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
"registry-mirrors": ["https://egkr0rl5.mirror.aliyuncs.com"],
"insecure-registries":["zkturing.imwork.net:30002"]    
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

["zkturing.imwork.net:30002"]    #修改安装使用的出口地址

请稍微等一下 查看pod是否正常启动

```
kubectl get pod -n harbor
```

检测一下是否成功

```
docker login -u admin -p Harbor12345 zkturing.imwork.net:30002
```

```
WARNING! Using --password via the CLI is insecure. Use --password-stdin.
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```

出现 Succeeded 表示成功

## gitlab部署教程

### 步骤一 gitlab部署教程

添加harbor仓库

```
helm repo add gitlab-jh https://charts.gitlab.cn/
helm repo update
```

查看harbor所有版本

```
 helm search repo gitlab-jh/gitlab --versions
```

```
helm repo update
```

```
helm pull gitlab-jh/gitlab --version 6.1.0 --untar
cd gitlab
```

```
kubectl create ns gitlab
```

vi values.yaml 修改对应位置

```
global:
  hosts:
    domain: test.sxxpqp.top  #需要gitlab.test.sxxpqp.top、minio.test.sxxpqp.top、registry.test.sxxpqp.top域名映射到对应的公网ip，公网映射到node节点上。
gitlab-runner:
  install: false  

certmanager-issuer:
   email: sxxpqp@gmail.com
```

安装gitlab

```
helm install gitlab -f values.yaml .  -n gitlab
```

```
kubectl get svc  -n gitlab
```

```
kubectl get svc  -n gitlab|grep ingress
gitlab-nginx-ingress-controller           LoadBalancer   10.233.23.249   <pending>     80:32528/TCP,443:30794/TCP,22:30473/TCP   35m
gitlab-nginx-ingress-controller-metrics   ClusterIP      10.233.46.190   <none>        10254/TCP                                 35m
gitlab-nginx-ingress-defaultbackend       ClusterIP      10.233.48.131   <none>        80/TCP
```

30794端口需要域名后面加的，或者通过nat转化到443端口也行。

查看账号：root 

密码为：

```
kubectl -n gitlab get secret gitlab-gitlab-initial-root-password -ojsonpath='{.data.password}' | base64 --decode ; echo
```

```
kubectl get pod -n gitlab #是否都运行running
```



登陆域名为：https://gitlab.test.sxxpqp.top:30794

