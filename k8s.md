### 📋 1. 环境规划与准备

假设你有 3 台机器（1 Master, 2 Worker）：

| 角色   | 主机名       | IP 地址        | 配置要求         |
| :----- | :----------- | :------------- | :--------------- |
| Master | `k8s-master` | `192.168.1.10` | 2C/2G+ (建议 4G) |
| Worker | `k8s-node1`  | `192.168.1.11` | 2C/2G+           |
| Worker | `k8s-node2`  | `192.168.1.12` | 2C/2G+           |

> **注意**：以下命令需要在**所有节点**（Master + Worker）上执行，除非特别标注。

------

### 🛠️ 2. 系统初始化 (所有节点)

#### 2.1 关闭防火墙、Swap 和 SELinux

```bash
1# 关闭防火墙
2systemctl stop firewalld && systemctl disable firewalld
3# 如果是 Ubuntu: ufw disable
4
5# 关闭 Swap (K8s 默认要求关闭，除非配置了 Kubelet 支持)
6swapoff -a
7sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
8
9# 临时关闭 SELinux (生产环境建议配置为 permissive)
10setenforce 0
11sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

#### 2.2 配置内核参数 (开启桥接流量监听)

```bash
1cat <<EOF | tee /etc/sysctl.d/k8s.conf
2net.bridge.bridge-nf-call-ip6tables = 1
3net.bridge.bridge-nf-call-iptables = 1
4net.ipv4.ip_forward = 1
5EOF
6
7sysctl --system
```

#### 2.3 配置主机名解析

在所有节点修改 `/etc/hosts`，添加集群节点信息：

```bash
1cat <<EOF >> /etc/hosts
2192.168.1.10 k8s-master
3192.168.1.11 k8s-node1
4192.168.1.12 k8s-node2
5EOF
```

------

### 📦 3. 安装容器运行时 (Containerd)

K8s v1.24+ 已移除 Docker Shim，推荐使用 Containerd。

#### 3.1 安装 Containerd

**CentOS:**

```bash
1yum install -y yum-utils
2yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
3yum install -y containerd.io
```

**Ubuntu:**

```bash
1apt-get update
2apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release
3curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
4echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
5apt-get update
6apt-get install -y containerd.io
```

#### 3.2 配置 Containerd (关键步骤)

生成默认配置并修改，以支持 `systemd` cgroup 驱动（与 K8s 一致）。

```bash
1mkdir -p /etc/containerd
2containerd config default > /etc/containerd/config.toml
3
4# 修改配置：将 SystemdCgroup 改为 true，并配置镜像加速器
5sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
6sed -i 's|registry.k8s.io/pause:3.9|registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.9|g' /etc/containerd/config.toml
7
8# 重启 containerd
9systemctl daemon-reload
10systemctl enable containerd
11systemctl restart containerd
```

------

### 🚀 4. 安装 Kubernetes 组件 (kubeadm, kubelet, kubectl)

#### 4.1 添加阿里云 Kubernetes 源

**CentOS:**

```bash
1cat <<EOF | tee /etc/yum.repos.d/kubernetes.repo
2[kubernetes]
3name=Kubernetes
4baseurl=https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.28/rpm/
5enabled=1
6gpgcheck=1
7gpgkey=https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.28/rpm/repodata/repomd.xml.key
8exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
9EOF
```

**Ubuntu:**

```bash
1curl -fsSL https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.28/deb/Release.key | gpg --dearmor -o /usr/share/keyrings/kubernetes-archive-keyring.gpg
2echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.28/deb/ /" | tee /etc/apt/sources.list.d/kubernetes.list
```

#### 4.2 安装指定版本

```bash
1# CentOS
2yum install -y kubelet-1.28.6 kubeadm-1.28.6 kubectl-1.28.6 --disableexcludes=kubernetes
3
4# Ubuntu
5apt-get update
6apt-get install -y kubelet=1.28.6-00 kubeadm=1.28.6-00 kubectl=1.28.6-00
7apt-mark hold kubelet kubeadm kubectl
```

#### 4.3 启动 Kubelet

```bash
1systemctl enable --now kubelet
```

*(此时 kubelet 会报错无法连接 apiserver，这是正常的，因为还没初始化)*

------

### 🎯 5. 初始化集群 (仅 Master 节点)

#### 5.1 执行初始化命令

我们指定 Pod 网段为 `10.244.0.0/16` (Calico 默认)，并使用阿里云镜像仓库。

```bash
1kubeadm init \
2  --image-repository registry.cn-hangzhou.aliyuncs.com/google_containers \
3  --kubernetes-version v1.28.6 \
4  --service-cidr=10.96.0.0/12 \
5  --pod-network-cidr=10.244.0.0/16 \
6  --apiserver-advertise-address=192.168.1.10  # 替换为你的 Master IP
```

#### 5.2 配置 kubectl 权限

初始化成功后，复制输出中的 `kubeadm join` 命令（稍后用于 Worker 节点加入），然后执行：

```bash
1mkdir -p $HOME/.kube
2sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
3sudo chown $(id -u):$(id -g) $HOME/.kube/config
4
5# 验证 master 状态 (此时 Node 是 NotReady，因为还没装网络插件)
6kubectl get nodes
```

------

### 🌐 6. 安装 Calico 网络插件 (关键)

K8s v1.28 推荐搭配 Calico v3.26+。我们将直接使用修改过镜像地址的 YAML，避免再次出现拉取失败。

#### 6.1 下载并修改 Calico YAML

```bash
1# 下载 Calico v3.26.1 配置文件
2curl https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml -O
3
4# 修改镜像地址为国内源 (使用 dockerproxy 或 阿里云代理)
5# 这里我们将 docker.io/calico 替换为 quay.m.daocloud.io/calico (DaoCloud 代理，速度快且全)
6sed -i 's|docker.io/calico/|quay.m.daocloud.io/calico/|g' calico.yaml
7
8# 【重要】修改 Pod 网段，必须与 kubeadm init 时一致 (10.244.0.0/16)
9# 找到 CALICO_IPV4POOL_CIDR 环境变量，默认通常是 192.168.0.0/16，需改为 10.244.0.0/16
10sed -i 's|192.168.0.0/16|10.244.0.0/16|g' calico.yaml
```

#### 6.2 应用配置

```bash
1kubectl apply -f calico.yaml
```

#### 6.3 观察状态

等待约 1-2 分钟，直到所有 Pod 运行：

```bash
1watch kubectl get pods -n kube-system
```

当 `calico-node-xxxxx` 和 `coredns-xxxxx` 都变为 `Running` 后，按 `Ctrl+C` 退出。

再次查看节点状态，应该变为 `Ready`：

```bash
1kubectl get nodes
```

------

### ➕ 7. 加入 Worker 节点 (在所有 Worker 节点执行)

回到 **Master 节点**，获取加入命令（如果在 5.1 步忘了复制，可以重新生成）：

```bash
1# 在 Master 执行
2kubeadm token create --print-join-command
```

复制输出的命令（类似 `kubeadm join 192.168.1.10:6443 --token ...`），在 **所有 Worker 节点** 执行。

------

### ✅ 8. 验证集群

部署一个测试 Nginx：

```bash
1kubectl create deployment nginx --image=nginx
2kubectl expose deployment nginx --port=80 --type=NodePort
3kubectl get pods,svc
```

如果 Nginx Pod 状态为 `Running` 且有 IP 地址，恭喜你！全新的 **K8s v1.28 + Calico v3.26** 集群搭建成功！🎉

### 💡 常见问题提示

1. **Pod 一直 Pending**: 检查 `calico-node` 是否正常运行。
2. **Worker 节点 NotReady**: 检查 Worker 节点上的 `kubelet` 日志 (`journalctl -u kubelet`)，通常是因为镜像拉取失败或网络不通。
3. **镜像拉取失败**: 本教程已配置国内源，如果仍失败，请检查服务器是否能访问外网，或尝试手动 `crictl pull` 对应镜像。