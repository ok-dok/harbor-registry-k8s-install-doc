# Harbor私有仓库搭建教程
这是一个关于如何搭建Harbor私有仓库详细的说明文档，你可以参考此说明在你的k8s集群上成功搭建起一套Harbor私有仓库。

**注意：**

本教程中所有应用都将使用Helm来创建（本地PV除外），而非直接使用K8s资源定义文件。

Helm是一个k8s的应用包管理工具，如果你不是很理解，你可以想想一下CentOS的 yum 命令，Debian的 apt-get 命令，Python 的 pip 命令等，它可以让你很方便的创建一个k8s应用，而不是跟各种繁琐的k8s资源定义文件（yaml）打交道。

关于Helm的更多信息，可以查看官方文档说明来学习如何使用它：[Helm是什么？](https://helm.sh/zh/)

接下来，我将按照我自己的搭建步骤来详细介绍，如何从头开始搭建一个Harbor私有镜像仓库。

本文使用的是Helm3。

## 1. 创建PersistentVolume
首先我们要有存储来支持我们的应用，一般在云平台上，会提供各种形式的存储，比如对象存储、块存储、文件存储、高效SSD云盘等等，或者是分布式存储Ceph、Glusterfs等等，这些如何用来创建PV后面我将在单独说明。

这里主要介绍：如何挂载一个本地硬盘，并创建一个本地的PersistentVolume。

首先创建一个StorageClass，指定为本地存储类型：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

然后在每一台具有本地硬盘的机器上，将硬盘格式化：

```sh
sudo mkfs.ext4 /dev/sdb
```
使用fdisk -l可以列出机器上可用的硬盘和分区。上面命令将整块硬盘格式化，不做分区，如果你是有单独的一个分区来提供使用的话，可以直接指定分区名，如：mkfs.ext4 /dev/sdb1。

然后将该硬盘/分区挂载到目录/data/：

修改/etc/fstab文件，加入一行数据：

```sh
/dev/sdb /data  ext4    defaults        0 0
```

或者直接root用户执行命令：

```sh
cat >> /etc/fstab <<EOF
/dev/sdb /data  ext4    defaults        0 0
EOF
```

然后使挂载立即生效：

```sh
sudo mount -a
```

创建/data/k8s目录，用于提供给k8s作为存储路径：

```sh
sudo mkdir -p /data/k8s
```
每一台有硬盘存储的机器都做这么个操作，然后创建一个PV，并指定节点亲和性：

文件名：master-pv.yaml：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: master-pv
spec:
  capacity:
    storage: 180Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  # 指定所属的存储分类为本地存储
  storageClassName: local-storage
  local:
    # 设置磁盘挂载目录
    path: /data/k8s
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - master
          #- othernode
```

然后应用创建该PV：

```sh
kubectl apply -f master-pv.yaml
```


有了本地PV之后，我们就可以利用它来创建一个NFS存储了。

## 2. 搭建NFS服务以提供动态申请存储支持（PersistentVolumeClaim）

如果已经有了现成的PV，第一步可以不做，直接进行这一步操作。

如果你已经有一个NFS服务器，那么你可以直接使用nfs客户端来部署：
[nfs-subdir-external-provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/tree/master/charts/nfs-subdir-external-provisioner)。

这里介绍直接使用Pod部署一个NFS服务器：
[nfs-server-provisioner](https://github.com/kubernetes-sigs/nfs-ganesha-server-and-external-provisioner/tree/master/deploy/helm)。

首先添加仓库：
```sh
helm repo add kvaps https://kvaps.github.io/charts
```

然后下载[values.yaml](https://github.com/kubernetes-sigs/nfs-ganesha-server-and-external-provisioner/blob/master/deploy/helm/values.yaml)，对其进行修改。

我这里将此文件修改病重新命名为：[kvaps-nfs-server-provisioner-values.yaml](kvaps-nfs-server-provisioner-values.yaml)。

其中主要修改的内容是`persistence`和`storageClass`。

persistence部分指定申请使用本地存储分类`local-storage`中的存储180Gi，因为我在第一步中创建的本地存储总共180Gi，索性全部分给nfs使用了，记得访问模式要设置为`ReadWriteOnce`，表示只能被当前的nfs-server-provisioner所在的节点挂载（只能被一个节点挂载），不能被其他节点挂载，这种模式是非共享的。

```yaml
persistence:
  enabled: true
  storageClass: "local-storage"
  accessMode: ReadWriteOnce
  size: 180Gi
```

然后在storageClass部分指定创建存储分类`nfs0`，并设置为`默认的分类`，`允许`存储动态扩展，回收策略指定为`删除`，nfs版本设置为`3`：

```yaml
storageClass:
  create: true
  provisionerName: nfs
  defaultClass: true
  name: nfs0
  allowVolumeExpansion: true
  parameters: {}
  # 指定nfs版本为3
  mountOptions:
    - vers=3
  reclaimPolicy: Delete
```

最后，使用Helm创建该nfs-server-provisioner：
```yaml
helm install nfs kvaps/nfs-server-provisioner -f kvaps-nfs-server-provisioner-values.yaml
```
创建完成后，查看一下状态：

```yaml
kubectl get pods
```

## 3. 部署Ingress Controller

Ingress是K8s提供集群外部访问服务的能力的一种资源，Ingress 可以提供负载均衡、SSL 终结和基于名称的虚拟托管。

关于Ingress的介绍：[Ingress是什么？](https://kubernetes.io/zh/docs/concepts/services-networking/ingress/)

关于Ingress Controller介绍：[Ingress控制器](https://kubernetes.io/zh/docs/concepts/services-networking/ingress-controllers/)

这里使用Nginx作为Ingress Controller：[nginx-ingress-controller](https://github.com/bitnami/charts/tree/master/bitnami/nginx-ingress-controller)

首先添加`bitnami`仓库：
```sh
helm repo add bitnami https://charts.bitnami.com/bitnami
```

下载[values.yaml](https://github.com/bitnami/charts/blob/master/bitnami/nginx-ingress-controller/values.yaml)，对其进行修改并重新保存为：[bitnami-nginx-ingress-controller-values.yaml](bitnami-nginx-ingress-controller-values.yaml)

里面大部分配置都是默认的，主要修改了以下内容：

```yaml
kind: DaemonSet
daemonset:
  useHostPort: true
  hostPorts:
    http: 80
    https: 443
```

将部署方式从Deployment换成了`DaemonSet`，这样可以使得每个节点的`80`和`443`端口都绑定到nginx-ingress-controller上，这样任意一个节点IP的`80`和`443`端口都可以从外部访问了。

然后将其安装部署在k8s上：

```sh
helm install ingress bitnami/nginx-ingress-controller -f bitnami-nginx-ingress-controller-values.yaml
```

查看服务运行状态：
```yaml
kubectl get svc,pods
```

## 4. 部署Harbor服务

Harbor是Redhat开发提供的一种私有镜像仓库（registry）服务的程序，支持Helm方式安装：[harbor-helm](https://github.com/goharbor/harbor-helm)。

依然是添加Harbor仓库：

```sh
helm repo add harbor https://helm.goharbor.io
```

下载资源配置文件：[values.yaml](https://github.com/goharbor/harbor-helm/blob/master/values.yaml)。

对其进行修改并重命名为：[harbor-values.yaml](harbor-values.yaml)。

其中主要修改的内容涉及几个部分：

### 1. 暴露服务类型配置

不同的服务暴露方式配置方式不一样，这里采用ingress对外提供服务，在[第3节](#3-部署ingress-controller)中已经介绍了如何部署Ingress Controller，在其中设置了ingress类型为`nginx`，我们在这里配置harbor使用该ingress class，然后设置了harbor核心仓库服务的域名地址和notary服务的地址：

```yaml
expose:
  type: ingress
  tls:
    enable: true
  ingress:
    hosts:
      # 设置harbor核心服务的域名
      core: registry.dclingcloud.com
      # 设置notary的域名
      notary: notary.dclingcloud.com
    annotations:
      # 设置ingress类型为nginx
      kubernetes.io/ingress.class: "nginx"
      ingress.kubernetes.io/ssl-redirect: "true"
      ingress.kubernetes.io/proxy-body-size: "0"
      nginx.ingress.kubernetes.io/ssl-redirect: "true"
      nginx.ingress.kubernetes.io/proxy-body-size: "0"
```

由于我们使用了ingress方式部署，所以tls必须要开启，默认配置下，会自动生成tls的ca.crt，tls.crt， tls.key文件，这是使用自签名的方式生成的ca证书。

### 2. 设置外部服务地址

需要设置对外提供服务的仓库的地址，格式为`协议://地址:端口`，80和443端口可不写。

```yaml
externalURL: https://registry.dclingcloud.com
```

### 3. registry配置

```yaml
registry:
  # ...
  # 16位加密秘钥，默认会自动生成
  secret: ""
  credentials:
    # 设置用户名
    username: "harbor_registry_user"
    # 设置登录密码
    password: "harbor_registry_password"
    # 修改了用户名和密码后，必须使用htpasswd工具来对其进行加密(bcrypt加密)，得到htpasswd文件，只有在启用TLS时才会使用到。
    # 加密命令："htpasswd -nbBC10 $username $password".
    # htpasswd命令安装："yum install httpd"
    htpasswd: "harbor_registry_user:$2y$10$9L4Tc0DJbFFMB6RdSCunrOpTHdwhid4ktBJmLD00bYgqkkGOvll3m"
```

### 4. 数据库配置
```yaml
# 数据库必须是 PostgreSQL 类型
database:
  # 使用内置数据库
  type: internal
  internal:
    # ...
    # 设置root用户密码
    password: "db_2021@"
```

### 5. 配置动态存储

harbor各个组件需要申请存储，我们利用前面配置好的NFS存储服务来动态申请存储（PersistentVolumeClaim），指定申请存储类型（storageClass）为`nfs0`：

```yaml
persistence:
  enabled: true
  # 设置回收策略为保持，当删除服务时，保留存储的数据
  resourcePolicy: "keep"
  persistentVolumeClaim:
    # 仓库配置（需要存储镜像）
    registry:
      storageClass: "nfs0"
      subPath: ""
      accessMode: ReadWriteOnce
      # 给仓库组件100GB的存储空间，用于存储镜像
      size: 100Gi
    # 存储Helm chart
    chartmuseum:
      existingClaim: ""
      storageClass: "nfs0"
      subPath: ""
      accessMode: ReadWriteOnce
      size: 5Gi
    jobservice:
      existingClaim: ""
      storageClass: "nfs0"
      subPath: ""
      accessMode: ReadWriteOnce
      size: 1Gi
    # 由于使用内置数据库，因此必须为数据库申请存储，默认1GB
    database:
      existingClaim: ""
      storageClass: "nfs0"
      subPath: ""
      accessMode: ReadWriteOnce
      size: 1Gi
    # 使用内置的redis服务，也需要存储空间，默认1GB
    redis:
      existingClaim: ""
      storageClass: "nfs0"
      subPath: ""
      accessMode: ReadWriteOnce
      size: 1Gi
    trivy:
      existingClaim: ""
      storageClass: "nfs0"
      subPath: ""
      accessMode: ReadWriteOnce
      size: 5Gi
  imageChartStorage:
    disableredirect: false
    # image和chart存储直接使用文件系统，不使用额外的外部存储
    type: filesystem
    filesystem:
      rootdirectory: /storage
```

组件配置修改完成后，使用helm进行部署：

```sh
helm install harbor harbor/harbor -f harbor-values.yaml -n harbor
```

查看部署运行状态：

```sh
kubectl get ingress,svc,pods -n harbor
```

### 6. 设置harbor后台管理员账户登录密码

```yaml
harborAdminPassword: "Harbor12345"
```

## 5. 配置域名解析

查看harbor ingress暴露的服务地址，将域名解析到对应的node ip上即可。

```sh
kubectl get ingress -n harbor
```

然后能够看到集群的每个node都被暴露了80和443端口，如果外部有负载均衡器，可以将集群的node ip统一加入到负载均衡器中。

由于我们是在本地做测试，所以我们任取一个节点ip，在需要访问私有仓库的每一台机器上修改hosts文件，添加域名解析：

```
172.16.100.10 registry.dclingcloud.com
```

然后就可以使用浏览器访问：[registry.dclingcloud.com](https://registry.dclingcloud.com)，登录用户名：`admin`，登录密码：`Harbor12345`。

## 6. 修改集群镜像仓库配置

在完成harbor的部署之后，我们需要将每个节点的容器仓库地址配置修改，添加harbor的私有仓库。

在集群中，容器运行时可能不是Docker，也有可能是Containerd，因此，我们分两部分来分别介绍它们将如何进行配置。

在介绍如何配置Docker和Containerd的私有仓库之前，我们首先需要生成客户端证书，因为在前面harbor的部署安装时，我们采用了默认的tls自签名证书自动生成的方式，对于自签名的证书，默认提供了三个文件：

- ca.crt：CA根证书
- tls.crt：客户端证书
- tls.key：秘钥

在docker和containerd的规定中，所有`*.crt`文件都被认为是CA根证书，然后一个或多个`<filename>.key/cert`成对的文件来表示访问存储库需要自签名客户端证书和秘钥。

因此，而默认生成的客户端证书为`tls.crt`，不是`*.cert`类型，如果只是简单的改后缀名是不行的，如果只是简单的更改后缀名，那么将会报出以下错误提示：

>Missing key KEY_NAME for client certificate CERT_NAME. CA certificates should use the extension .crt.

必须按格式来生成`.cert`格式的客户端证书。此部分配置可以参考：
[Docker安全证书配置说明](https://docs.docker.com/engine/security/certificates/)。

### 1. 生成客户端证书

harbor安装之后，会自动生成证书和密钥，通过以下命令来获取：

```sh
kubectl -n harbor get secret -o jsonpath="{.data}" | jq .
```
jq命令是用来格式化输出json字符串的，如果你还没有安装，可以进行安装：

```sh
yum install jq
```

得到的输出将会含有三个文件内容，分别是：ca.crt, tls.crt, tls.key。

这三个文件都是使用base64加密的，必须经过解密才可以使用。

解密方式：

```sh
kubectl -n harbor get secret -o jsonpath="{.data.ca\.crt}" | base64 -d > ca.crt

kubectl -n harbor get secret -o jsonpath="{.data.tls\.crt}" | base64 -d > tls.crt

kubectl -n harbor get secret -o jsonpath="{.data.tls\.key}" | base64 -d > tls.key
```
这样就得到了解密后的三个文件。

接下来要利用tls.key生成客户端证书文件tls.cert：

```sh
openssl req -new -x509 -text -key tls.key -out tls.cert
```
然后会提示填写地区、国家、城市、公司、域名等信息，依次填写即可。

要注意，域名必须和仓库域名地址保持一致，或者是他的上级域名。

比如我们设置的仓库地址是：registry.dclingcloud.com，那么就可以使用registry.dclingcloud.com或者dclingcloud.com来签发客户端证书，域名填写错误将会导致无法验证证书。

### 1. Docker

配置Docker的私有仓库相对比较简单，只需要建立`/etc/docker/certs.d/<仓库地址>`目录，然后将ca.crt，tls.cert，tls.key拷贝到该目录即可。

注意：仓库地址可以是`ip:port`形式，也可以是域名地址形式，这跟harbor的部署方式有关。

我们使用的是ingress来部署，并配置了域名，所以我们需要创建对应的目录形如：

```sh
mkdir -p /etc/docker/certs.d/registry.dclingcloud.com
```
然后将三个文件拷贝进去：

```sh
cp ca.crt tls.cert tls.key /etc/docker/certs.d/registry.dclingcloud.com/
```

最终的文件目录结构如下所示：

```
/etc/docker/certs.d/          <-- 证书目录
└─ registry.dclingcloud.com   <-- 仓库地址
    ├─ client.cert            <-- 客户端证书
    ├─ client.key             <-- 客户端秘钥
    └─ ca.crt                 <-- CA根证书（证书签发机构）
```

然后执行`docker login`进行登录：

```sh
$ docker login registry.dclingcloud.com
Username: admin
password: Harbor12345

Login succeed.
```
然后就可以愉快的进行推送、拉取镜像了。

### 2. Containerd

containerd的配置文件是：`/etc/containerd/config.toml`。

修改其中部分内容如下：

```toml
[plugins."io.containerd.grpc.v1.cri".registry.mirrors]
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
    endpoint = ["https://registry-1.docker.io"]
  # 配置私有仓库地址
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."registry.dclingcloud.com"]
    endpoint = ["https://registry.dclingcloud.com"]
# 配置tls
[plugins."io.containerd.grpc.v1.cri".registry.configs]
  [plugins."io.containerd.grpc.v1.cri".registry.configs."registry.dclingcloud.com".tls]
    ca_file   = "/etc/containerd/certs.d/registry.dclingcloud.com/ca.crt"
    cert_file = "/etc/containerd/certs.d/registry.dclingcloud.com/tls.cert"
    key_file  = "/etc/containerd/certs.d/registry.dclingcloud.com/tls.key"
    # 如果不进行安全验证，可以设置忽略
    # insecure_skip_verify = true
# 设置用户登录凭证
[plugins."io.containerd.grpc.v1.cri".registry.configs."gcr.io".auth]
  username = "admin"
  password = "Harbor12345"
```
修改完配置后，添加证书文件，与docker类似，首先创建目录：

```
mkdir -p /etc/containerd/certs.d/registry.dclingcloud.com
```
然后拷贝三个证书文件到该目录：

```sh
cp ca.crt tls.cert tls.key /etc/containerd/certs.d/registry.dclingcloud.com/
```
最后重启containerd:

```sh
systemctl restart containerd
```
然后就可以愉快的拉取和推送镜像到Harbor私有镜像仓库了。

参考资料：[关于Registry部分的配置说明](https://github.com/mikebrow/containerd/blob/document-hosts.toml/docs/cri/registry.md)

**题外话：**

相对于docker，Containerd的配置相对复杂一些，不过Contaienrd官方正在改进这方面的工作，未来将会采用和Docker类似的配置方式。关于这方面的改进工作请看：

- [issue-5304](https://github.com/containerd/containerd/pull/5304)：remove mirrors from default; document the deprecation of registry.configs and registry.mirrors
- [issue-5309](https://github.com/containerd/containerd/pull/5309)：adds description for hosts.toml

在配置使用了`config_path`时，上面的registry.configs和registry.mirrors配置就会失效，config_path指定主机配置目录，使用类似docker的那种形式：

```
/etc/containerd/certs.d/          <-- 证书目录
└─ registry.dclingcloud.com   <-- 仓库地址
    ├─ hosts.toml             <-- 配置文件
    ├─ client.cert            <-- 客户端证书
    ├─ client.key             <-- 客户端秘钥
    └─ ca.crt                 <-- CA根证书（证书签发机构）
```
在这样的示例中，config_path="/etc/containerd/certs.d/"，hosts.toml提供额外的配置信息，具体可以参考issue-5309说明。

## 7. 打包和推送镜像到私有仓库

### 1. 使用docker命令
首先将本地镜像打一个tag：
```sh
docker tag <sourceRepo>:<tag> registry.dclingcloud.com/library/<repo>:<tag>
```
然后推送镜像到私有仓库:

```sh
docker push registry.dclingcloud.com/library/<repo>:<tag>
```

### 2. 使用ctr命令
对于containerd容器来说，没有docker命令，可以使用ctr命令来执行相关操作，ctr命令在docker环境中也能使用。

将本地镜像打一个tag：
```sh
ctr i tag <sourceRepo>:<tag> registry.dclingcloud.com/library/<repo>:<tag>
```
然后推送镜像到私有仓库:

```sh
ctr i push registry.dclingcloud.com/library/<repo>:<tag>
```