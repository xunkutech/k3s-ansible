# 在宝x云ECS服务器上离线部署高可用k3s集群

## 系统要求

- 一个k3s集群可以访问的MySQL实例，用于保存etcd数据
- 一个k3s集群可以访问的Harbor实例，用于提供构建k3s集群需要的docker镜像。镜像可以通过代理进行缓存，也可以手工进行上传
- 一个可以运行ansible的控制台，用于执行集群构建脚本（可以和Harbor按照在同一台机器上）
- 所有集群节点都可以通过publickey方式进行ssh登录，并且ssh登录用户可以无需密码sudo
- 至少4台安装了Redhat 7.6 Enterprise Linux的ECS主机。其中两台为controller节点，其它为worker节点

## 准备磁盘分区

ECS都上额外准备了一块200G的本地硬盘（/dev/sdb）。可以通过以下命令进行磁盘的格式化并挂载到/data目录：

```bash
lsblk

sudo parted -s -a optimal -- /dev/sdb mklabel gpt
sudo parted -s -a optimal -- /dev/sdb mkpart primary 0% 100%
sudo parted -s -- /dev/sdb align-check optimal 1
sudo pvcreate /dev/sdb1
sudo vgcreate vg0 /dev/sdb1
sudo vgs
sudo lvcreate -n data -Zn -l +100%FREE vg0
sudo lvs
sudo mkfs.xfs -f /dev/mapper/vg0-data
sudo mkdir /data
echo "/dev/mapper/vg0-data /data xfs defaults 0 0" | sudo tee -a /etc/fstab
df -hT /data/
```

## 修改内核启动参数并重启

```bash
sudo grubby --args="namespace.unpriv_enable=1 user_namespace.enable=1" --update-kernel=`grubby --default-kernel`
sudo reboot
```

## 构建k3s集群

### 安装ansible及组件

```bash
python3 -m pip install --user -r requirements.txt
ansible-galaxy collection install -r ./collections/requirements.yml --force
```

### k3s集群的初始化

首先，将inventory/sample复制一份到一个新的目录。比如：

```bash
cp -R inventory/sample inventory/my-cluster
```

然后，修改 `inventory/my-cluster/hosts.ini` 文件，使它对应我们需要管理的集群节点。

比如：

```ini
[master]
10.60.80.12
10.60.80.13

[node]
10.60.80.14
10.60.80.15

[k3s_cluster:children]
master
node
```

最后，修改 `inventory/my-cluster/group_vars/all.yml` 文件：

- 集群主机登录用户名（ansible_user: hgbadmin）
- 数据库地址（eg: --datastore-endpoint "mysql://k3s:k3spassword@tcp(10.60.80.3:3306)/k3s_cluster）
- harbor服务器的地址（eg: registry_server_location: http://10.60.80.11），
- apiserver vip地址（eg: apiserver_endpoint: "10.60.80.130"）
- LoadBalancer地址池（eg: metal_lb_ip_range: "10.60.80.131-10.60.80.135"）等配置。

执行集群初始化脚本，并等待其结束

```bash
ansible-playbook site.yml -i ./inventory/my-cluster/hosts.ini
```

将集群的配置文件保存到本地

```bash
mkdir -p ~/.kube; scp 10.60.80.12:/root/.kube/config ~/.kube

# 测试集群
kubectl get nodes
kubectl get all -A
```

### k3s集群的重置

重置所有的节点，保存集群配置：

```bash
ansible-playbook reset.yml -i ./inventory/my-cluster/hosts.ini
ansible-playbook site.yml -i ./inventory/my-cluster/hosts.ini
```

重置所有的节点，删除集群配置：

```bash
ansible-playbook reset.yml -i ./inventory/my-cluster/hosts.ini

# 删除数据库中的`kine`表
docker exec -ti mysql-server /bin/sh -c  'echo "drop table kine;" | mysql -uroot -p$MYSQL_ROOT_PASSWORD k3s_cluster'

ansible-playbook site.yml -i ./inventory/my-cluster/hosts.ini
```

### 登录kubernetes dashboard控制台

将kubernetes-dashboard映射到节点上

```bash
kubectl patch svc kubernetes-dashboard -n kubernetes-dashboard -p '{"spec":{"type":"NodePort", "ports"[0]:{"nodePort": "30300"}}}'
```

获得登录的token

```bash
kubectl -n kubernetes-dashboard create token admin-user
```

## 安装Harbor实例

我们将通过Docker运行Harbor实例，并且将harbor的镜像仓库保存在由lvm管理的本地磁盘卷上。如果在离线环境下，需要在rhel7主机上安装以下软件包：

- container-selinux
- fuse-overlayfs
- slirp4netns
- docker-ce
- docker-ce-cli
- containerd.io
- docker-compose-plugin

安装harbor服务

```bash
# Download harbor offline package
wget https://github.com/goharbor/harbor/releases/download/v2.7.0/harbor-offline-installer-v2.7.0.tgz

# Extract install package
tar zxf harbor-offline-installer-v2.7.0.tgz

# 
cd harbor
cp harbor.yml.tmpl harbor.yml

# vi harbor.yml to set values
sudo ./install.sh
sudo docker compose up -d
```

安装完成后，既可以通过admin/Harbor12345登录访问harbor

## 安装rancher控制台

```bash
helm upgrade --install cert-manager ./charts/cert-manager-v1.11.0.tgz \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true

helm upgrade --install rancher ./charts/rancher-2.7.1.tgz \
  --namespace cattle-system \
  --create-namespace \
  --set hostname=rancher \
  --set bootstrapPassword=rancher \
  --set ingress.tls.source=rancher

kubectl get ing -n cattle-system
```

rancher已经通过ingress映射，请通过ingress中提供的地址进行访问

## 安装longhorn分布式存储卷

```bash
helm upgrade --install longhorn-crd charts/longhorn-crd-101.1.0+up1.3.2.tgz \
  --namespace longhorn-system \
  --create-namespace

helm upgrade --install longhorn charts/longhorn-101.1.0+up1.3.2.tgz \
  --namespace longhorn-system \
  --create-namespace
```

在rancher控制台中，打开longhorn管理界面进行分布式存储卷的管理

## 部署hello-app测试应用

### 创建deployment

### 创建ingress

## 安装集群监控

### 安装ServiceMonitor和Prometheus

```bash
# Install service monitor - nodeExporter
cd monitoring
kubectl apply -f node-exporter/
kubectl get pods -n monitoring

# Install service monitor - kube-state-metrics
cd monitoring
kubectl apply -f kube-state-metrics/
kubectl get pods -n monitoring

# Install service monitor - kubelet
cd monitoring
kubectl apply -f kubelet/
kubectl get ServiceMonitor -n monitoring

# Install service monitor - longhorn-prometheus-servicemonitor
cd monitoring
kubectl apply -f longhorn-servicemonitor.yaml

# Install prometheus 
cd monitoring
kubectl apply -f prometheus/
kubectl get svc -n monitoring
```

### 安装Grafana和图表

```bash
cd monitoring
kubectl apply -f grafana/
kubectl get svc -n monitoring
```
