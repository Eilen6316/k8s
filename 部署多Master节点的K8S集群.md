# 多Master节点的k8s集群部署

## 一、准备工作

##### 1.准备五台主机（三台Master节点，一台Node节点，一台普通用户）如下：

| 角色         | IP                  | 内存   | 核心    | 磁盘    |
| ------------ | ------------------- | ------ | ------- | ------- |
| Master01     | 192.168.116.141     | 4G     | 4个     | 55G     |
| Master02     | 192.168.116.142     | 4G     | 4个     | 55G     |
| Master03     | 192.168.116.143     | 4G     | 4个     | 55G     |
| Node         | 192.168.116.144     | 4G     | 4个     | 55G     |
| ~~普通用户~~ | ~~192.168.116.150~~ | ~~4G~~ | ~~4个~~ | ~~55G~~ |

##### 2.关闭SElinux，因为SElinux会影响K8S部分组件无法正常工作：

```shell
sed -i '1,$s/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
# reboot
```

##### 3.四台主机分别配置主机名（不包括普通用户），如下：

 控制节点Master01：

```shell
hostnamectl set-hostname master01 && bash
```

 控制节点Master02：

```shell
hostnamectl set-hostname master02 && bash
```

 控制节点Master03：

```shell
hostnamectl set-hostname master03 && bash
```

工作节点Node：

```shell
hostnamectl set-hostname node && bash
```

##### 4.四台主机（不包括普通用户）分别配置host文件：

- 进入hosts文件：

  ```shell
  vim /etc/hosts
  ```

- 修改文件内容，添加四台主机以及IP：

  ```bash
  127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
  ::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
  
  192.168.116.141 master01
  192.168.116.142 master02
  192.168.116.143 master03
  192.168.116.144 node
  ```

- 修改完可以四台主机用ping命令检查是否连通：

  ```shell
  ping -c1 -W1 master01
  ping -c1 -W1 master02
  ping -c1 -W1 master03
  ping -c1 -W1 node
  ```

  

#### 5.四台主机（不包括普通用户）分别下载所需意外组件包和相关依赖包：

```shell
yum install -y yum-utils device-mapper-persistent-data lvm2 wget net-tools nfs-utils lrzsz gcc gcc-c++ make cmake libxml2-devel openssl-devel curl curl-devel unzip autoconf automake zlib-devel epel-release openssh-server libaio-devel vim ncurses-devel socat conntrack telnet ipvsadm
```

所需相关意外组件包解释如下：

​	**yum-utils**：提供了一些辅助工具用于 `yum` 包管理器，比如 `yum-config-manager`，`repoquery` 等。

​	**device-mapper-persistent-data**：与 Linux 的设备映射功能相关，通常与 LVM（逻辑卷管理）和容器存储（如 Docker）有关。

​	**lvm2**：逻辑卷管理器，用于管理磁盘上的逻辑卷，允许灵活的磁盘分区管理。

​	**wget**：一个非交互式网络下载工具，支持 HTTP、HTTPS 和 FTP 协议，常用于下载文件。

​	**net-tools**：提供一些经典的网络工具，如 `ifconfig`，`netstat` 等，用于查看和管理网络配置。

​	**nfs-utils**：支持 NFS（网络文件系统）的工具包，允许客户端挂载远程文件系统。

​	**lrzsz**：`lrz` 和 `lsz` 是 Linux 系统下用于 X/ZMODEM 文件传输协议的命令行工具，常用于串口传输数据。

​	**gcc**：GNU C 编译器，用于编译 C 语言程序。

​	**gcc-c++**：GNU C++ 编译器，用于编译 C++ 语言程序。

​	**make**：用于构建和编译程序，通常与 `Makefile` 配合使用，控制程序的编译和打包过程。

​	**cmake**：跨平台的构建系统生成工具，用于管理项目的编译过程，特别适用于大型复杂项目。

​	**libxml2-devel**：开发用的 `libxml2` 库头文件，`libxml2` 是一个用于解析 XML 文件的 C 库。

​	**openssl-devel**：用于 OpenSSL 库开发的头文件和开发库，OpenSSL 是用于 SSL/TLS 加密的库。

​	**curl**：一个用于传输数据的命令行工具，支持多种协议（HTTP、FTP 等）。

​	**curl-devel**：开发用的 `curl` 库和头文件，支持在代码中使用 `curl` 相关功能。

​	**unzip**：用于解压缩 `.zip` 文件。

​	**autoconf**：自动生成配置脚本的工具，常用于生成软件包的 `configure` 文件。

​	**automake**：自动生成 `Makefile.in` 文件，结合 `autoconf` 使用，用于构建系统。

​	**zlib-devel**：`zlib` 库的开发头文件，`zlib` 是一个用于数据压缩的库。

​	**epel-release**：用于启用 EPEL（Extra Packages for Enterprise Linux）存储库，提供大量额外的软件包。

​	**openssh-server**：OpenSSH 服务器，用于通过 SSH 远程登录和管理系统。

​	**libaio-devel**：异步 I/O 库的开发头文件，提供异步文件 I/O 支持，常用于数据库和高性能应用。

​	**vim**：一个强大的文本编辑器，支持多种语言和扩展功能。

​	**ncurses-devel**：开发用的 `ncurses` 库，提供终端控制和用户界面的构建工具。

​	**socat**：一个多功能的网络工具，用于双向数据传输，支持多种协议和地址类型。

​	**conntrack**：连接跟踪工具，显示和操作内核中的连接跟踪表，常用于网络防火墙和 NAT 配置。

​	**telnet**：用于远程登录的一种简单网络协议，允许通过命令行与远程主机进行通信。

​	**ipvsadm**：用于管理 IPVS（IP 虚拟服务器），这是一个 Linux 内核中的负载均衡模块，常用于高可用性负载均衡集群。

#### 6.配置主机之间免密登录

##### 四台节点同时执行：

​	1）配置三台Master主机到另外一台Node主机免密登录：

```shell
ssh-keygen # 遇到问题不输入任何内容，直按回车
```

​	2）把刚刚生成的公钥文件传递到其他Master和node节点，输入yes后，在输入主机对应的密码：

```shell
ssh-copy-id master01
ssh-copy-id master02
ssh-copy-id master03
ssh-copy-id node
```

#### 7.关闭所有主机的firewall防火墙

​	如果不想关闭防火墙可以添加firewall-cmd规则进行过滤筛选，相关内容查询资料，不做演示。

**关闭防火墙：**

```shell
systemctl stop firewalld && systemctl disable firewalld
systemctl status firewalld # 查询防火墙状态，关闭后应为	Active: inactive (dead)
```

**添加防火墙规则**：

6443：Kubernetes Api Server			2379、2380：Etcd数据库

10250、10255：kubelet服务	 		10257：kube-controller-manager 服务

10259：kube-scheduler 服务		 	30000-32767：在物理机映射的 NodePort端口

179、473、4789、9099：Calico 服务	   9090、3000：Prometheus监控+Grafana面板

8443：Kubernetes Dashboard控制面板				

```shell
# Kubernetes API Server
firewall-cmd --zone=public --add-port=6443/tcp --permanent

# Etcd 数据库
firewall-cmd --zone=public --add-port=2379-2380/tcp --permanent

# Kubelet 服务
firewall-cmd --zone=public --add-port=10250/tcp --permanent
firewall-cmd --zone=public --add-port=10255/tcp --permanent

# Kube-Controller-Manager 服务
firewall-cmd --zone=public --add-port=10257/tcp --permanent

# Kube-Scheduler 服务
firewall-cmd --zone=public --add-port=10259/tcp --permanent

# NodePort 映射端口
firewall-cmd --zone=public --add-port=30000-32767/tcp --permanent

# Calico 服务
firewall-cmd --zone=public --add-port=179/tcp --permanent  # BGP
firewall-cmd --zone=public --add-port=473/tcp --permanent  # IP-in-IP
firewall-cmd --zone=public --add-port=4789/udp --permanent  # VXLAN
firewall-cmd --zone=public --add-port=9099/tcp --permanent  # Calico 服务

#Prometheus监控+Grafana面板
firewall-cmd --zone=public --add-port=9090/tcp --permanent
firewall-cmd --zone=public --add-port=3000/tcp --permanent

# Kubernetes Dashboard控制面板	
firewall-cmd --zone=public --add-port=8443/tcp --permanent

# 重新加载防火墙配置以应用更改
firewall-cmd --reload
```

#### 8.四台主机关闭swap交换分区

swap 分区的读写速度远低于物理内存。如果 Kubernetes 工作负载依赖于 swap 来补偿内存不足，会导致性能显著下降，尤其是在资源密集型的容器应用中。Kubernetes 更倾向于让节点直接面临内存不足的情况，而不是依赖 swap，从而促使调度器重新分配资源。

​	Kubernetes 默认会在 `kubelet` 启动时检查 `swap` 的状态，并要求其关闭。如果 `swap` 未关闭，Kubernetes 可能无法正常启动并报出错误。例如：

> [!WARNING]
>
> kubelet: Swap is enabled; production deployments should disable swap.

​	为了让 Kubernetes 正常工作，建议在所有节点上永久关闭 swap，同时调整系统的内存管理：

```shell
swapoff -a 	# 关闭当前swap

sed -i '/swap/s/^/#/' /etc/fstab 	# swap前添加注释

grep swap /etc/fstab # 成功关闭会这样：#/dev/mapper/rl-swap     none              swap    defaults        0 0
```

#### 9.修改内核参数

四台主机（不包括普通用户）分别执行:

```shell
modprobe br_netfilter
```

- `modprobe`：用于加载或卸载内核模块的命令。

- `br_netfilter`：该模块允许桥接的网络流量被 iptables 规则过滤，通常在启用网络桥接的情况下使用。
- 该模块主要在 Kubernetes 容器网络环境中使用，确保 Linux 内核能够正确处理网络流量的过滤和转发，特别是在容器间的通信中。

​	四台主机分别执行:

```shell
cat > /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

sysctl -p /etc/sysctl.d/k8s.conf # 使配置生效
```

- net.bridge.bridge-nf-call-ip6tables = 1：允许 IPv6 网络流量通过 Linux 网络桥接时使用 `ip6tables` 进行过滤。
- net.bridge.bridge-nf-call-iptables = 1：允许 IPv4 网络流量通过 Linux 网络桥接时使用 `iptables` 进行过滤。
- net.ipv4.ip_forward = 1：允许 Linux 内核进行 IPv4 数据包的转发（路由）。

​	这些设置确保在 Kubernetes 中，网络桥接流量可通过 `iptables` 和 `ip6tables` 过滤，并启用 IPv4 数据包转发，提升网络安全性和通信能力。

#### 10.配置安装Docker和Containerd的yum源

​	四台主机分别安装docker-ce源(任选其一，只安装一个)，后续操作只演示阿里源的。

```shell
# 阿里源
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# 清华大学开源软件镜像站
yum-config-manager --add-repo https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/centos/docker-ce.repo
# 中国科技大学开源镜像站
yum-config-manager --add-repo https://mirrors.ustc.edu.cn/docker-ce/linux/centos/docker-ce.repo
# 中科大镜像源
yum-config-manager --add-repo https://mirrors.ustc.edu.cn/docker-ce/linux/centos/docker-ce.repo
# 华为云源
yum-config-manager --add-repo https://repo.huaweicloud.com/docker-ce/linux/centos/docker-ce.repo
```

#### 11.配置K8S命令行工具所需要的yum源

```shell
cat > /etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
       https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

yum makecache
```

#### 12.四台主机进行时间同步

​	Chrony 和 NTPD都是用于时间同步的工具，但 Chrony 在许多方面有其独特的优点。以下是 Chrony 相较于 NTPD 的一些主要优点，并基于此，进行chrony时间同步的部署：

| 快速同步     | 在网络延迟较大或连接不稳定时，Chrony 可以更快地同步时间。 | 通常需要更长的时间来达到时间同步。     |
| ------------ | --------------------------------------------------------- | -------------------------------------- |
| 优点         | Chrony                                                    | NTPD                                   |
| 适应性强     | 在移动设备或虚拟环境中表现良好，能够快速适应网络变化。    | 在这些环境中的性能较差。               |
| 时钟漂移修正 | 能够更好地处理系统时钟漂移，通过频率调整来实现。          | 对系统时钟漂移的处理能力较弱。         |
| 配置简单     | 配置相对简单直观，易于理解和使用。                        | 配置选项较多，可能需要更多时间来熟悉。 |

​	1) 四台主机安装Chrony

```shell
yum -y install chrony
```

​	2）四台主机修改配置文件，添加国内 NTP 服务器 

```shell
echo "server ntp1.aliyun.com iburst" >> /etc/chrony.conf
echo "server ntp2.aliyun.com iburst" >> /etc/chrony.conf
echo "server ntp3.aliyun.com iburst" >> /etc/chrony.conf
echo "server ntp.tuna.tsinghua.edu.cn iburst" >> /etc/chrony.conf

tail -n 4 /etc/chrony.conf
systemctl restart chronyd 
```

​	3) 可以设置定时任务，每分钟重启chrony服务，进行时间校准（非必须）

```
echo "* * * * * /usr/bin/systemctl restart chronyd" | tee -a /var/spool/cron/root 
```

​	建议手动进行添加，首先执行`crontab -e`命令，在将如下内容添加至定时任务中

```
* * * * * /usr/bin/systemctl restart chronyd
```

- 这五个星号表示时间调度，每个星号代表一个时间字段，从左到右分别是：
  - 第一个星号：分钟（0-59）
  - 第二个星号：小时（0-23）
  - 第三个星号：日期（1-31）
  - 第四个星号：月份（1-12）
  - 第五个星号：星期几（0-7，0 和 7 都代表星期天）
- 在这里，每个字段都用 `*` 表示“每一个”，因此 `* * * * *` 的意思是“每分钟的每一秒”。
- `/usr/bin/systemctl` 是 `systemctl` 命令的完整路径，用于管理系统服务。

#### 13.安装Containerd

Containerd 是一个高性能的容器运行时，在 Kubernetes 中它负责容器的生命周期管理，包括创建、运行、停止和删除容器，同时支持从镜像仓库拉取和管理镜像。Containerd 提供容器运行时接口 (CRI)，与 Kubernetes 无缝集成，确保高效的资源利用和快速的容器启动时间。除此之外，它还支持事件监控和日志记录，方便运维和调试，是实现容器编排和管理的关键组件。

​	四台主机安装containerd1.6.22版本

```shell
yum -y install containerd.io-1.6.22
yum -y install containerd.io-1.6.22 --allowerasing # 如果安装有问题选择这个，默认用第一个
```

​	创建containerd的配置文件目录并修改自带的`config.toml`。

```shell
mkdir -pv /etc/containerd
vim /etc/containerd/config.toml
```

​	修改内容如下：

```toml
disabled_plugins = []
imports = []
oom_score = 0
plugin_dir = ""
required_plugins = []
root = "/var/lib/containerd"
state = "/run/containerd"
temp = ""
version = 2

[cgroup]
  path = ""

[debug]
  address = ""
  format = ""
  gid = 0
  level = ""
  uid = 0

[grpc]
  address = "/run/containerd/containerd.sock"
  gid = 0
  max_recv_message_size = 16777216
  max_send_message_size = 16777216
  tcp_address = ""
  tcp_tls_ca = ""
  tcp_tls_cert = ""
  tcp_tls_key = ""
  uid = 0

[metrics]
  address = ""
  grpc_histogram = false

[plugins]

  [plugins."io.containerd.gc.v1.scheduler"]
    deletion_threshold = 0
    mutation_threshold = 100
    pause_threshold = 0.02
    schedule_delay = "0s"
    startup_delay = "100ms"

  [plugins."io.containerd.grpc.v1.cri"]
    device_ownership_from_security_context = false
    disable_apparmor = false
    disable_cgroup = false
    disable_hugetlb_controller = true
    disable_proc_mount = false
    disable_tcp_service = true
    drain_exec_sync_io_timeout = "0s"
    enable_selinux = false
    enable_tls_streaming = false
    enable_unprivileged_icmp = false
    enable_unprivileged_ports = false
    ignore_deprecation_warnings = []
    ignore_image_defined_volumes = false
    max_concurrent_downloads = 3
    max_container_log_line_size = 16384
    netns_mounts_under_state_dir = false
    restrict_oom_score_adj = false
    sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.9"
    selinux_category_range = 1024
    stats_collect_period = 10
    stream_idle_timeout = "4h0m0s"
    stream_server_address = "127.0.0.1"
    stream_server_port = "0"
    systemd_cgroup = false
    tolerate_missing_hugetlb_controller = true
    unset_seccomp_profile = ""

    [plugins."io.containerd.grpc.v1.cri".cni]
      bin_dir = "/opt/cni/bin"
      conf_dir = "/etc/cni/net.d"
      conf_template = ""
      ip_pref = ""
      max_conf_num = 1

    [plugins."io.containerd.grpc.v1.cri".containerd]
      default_runtime_name = "runc"
      disable_snapshot_annotations = true
      discard_unpacked_layers = false
      ignore_rdt_not_enabled_errors = false
      no_pivot = false
      snapshotter = "overlayfs"

      [plugins."io.containerd.grpc.v1.cri".containerd.default_runtime]
        base_runtime_spec = ""
        cni_conf_dir = ""
        cni_max_conf_num = 0
        container_annotations = []
        pod_annotations = []
        privileged_without_host_devices = false
        runtime_engine = ""
        runtime_path = ""
        runtime_root = ""
        runtime_type = ""

        [plugins."io.containerd.grpc.v1.cri".containerd.default_runtime.options]

      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]

        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
          base_runtime_spec = ""
          cni_conf_dir = ""
          cni_max_conf_num = 0
          container_annotations = []
          pod_annotations = []
          privileged_without_host_devices = false
          runtime_engine = ""
          runtime_path = ""
          runtime_root = ""
          runtime_type = "io.containerd.runc.v2"

          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            BinaryName = ""
            CriuImagePath = ""
            CriuPath = ""
            CriuWorkPath = ""
            IoGid = 0
            IoUid = 0
            NoNewKeyring = false
            NoPivotRoot = false
            Root = ""
            ShimCgroup = ""
            SystemdCgroup = false

      [plugins."io.containerd.grpc.v1.cri".containerd.untrusted_workload_runtime]
        base_runtime_spec = ""
        cni_conf_dir = ""
        cni_max_conf_num = 0
        container_annotations = []
        pod_annotations = []
        privileged_without_host_devices = false
        runtime_engine = ""
        runtime_path = ""
        runtime_root = ""
        runtime_type = ""

        [plugins."io.containerd.grpc.v1.cri".containerd.untrusted_workload_runtime.options]

    [plugins."io.containerd.grpc.v1.cri".image_decryption]
      key_model = "node"

    [plugins."io.containerd.grpc.v1.cri".registry]
      config_path = "/etc/containerd/certs.d"  # 新版本推荐使用 config_path

    [plugins."io.containerd.grpc.v1.cri".x509_key_pair_streaming]
      tls_cert_file = ""
      tls_key_file = ""

  [plugins."io.containerd.internal.v1.opt"]
    path = "/opt/containerd"

  [plugins."io.containerd.internal.v1.restart"]
    interval = "10s"

  [plugins."io.containerd.internal.v1.tracing"]

  [plugins."io.containerd.metadata.v1.bolt"]
    content_sharing_policy = "shared"

  [plugins."io.containerd.monitor.v1.cgroups"]
    no_prometheus = false

  [plugins."io.containerd.runtime.v1.linux"]
    no_shim = false
    runtime = "runc"
    runtime_root = ""
    shim = "containerd-shim"
    shim_debug = false

  [plugins."io.containerd.runtime.v2.task"]
    platforms = ["linux/amd64"]
    sched_core = false

  [plugins."io.containerd.service.v1.diff-service"]
    default = ["walking"]

  [plugins."io.containerd.service.v1.tasks-service"]
    rdt_config_file = ""

  [plugins."io.containerd.snapshotter.v1.aufs"]
    root_path = ""

  [plugins."io.containerd.snapshotter.v1.devmapper"]
    async_remove = false
    base_image_size = ""
    discard_blocks = false
    fs_options = ""
    fs_type = ""
    pool_name = ""
    root_path = ""

  [plugins."io.containerd.snapshotter.v1.native"]
    root_path = ""

  [plugins."io.containerd.snapshotter.v1.overlayfs"]
    mount_options = []
    root_path = ""
    sync_remove = false
    upperdir_label = false

  [plugins."io.containerd.snapshotter.v1.zfs"]
    root_path = ""

  [plugins."io.containerd.tracing.processor.v1.otlp"]

[proxy_plugins]

[stream_processors]

  [stream_processors."io.containerd.ocicrypt.decoder.v1.tar"]
    accepts = ["application/vnd.oci.image.layer.v1.tar+encrypted"]
    args = ["--decryption-keys-path", "/etc/containerd/ocicrypt/keys"]
    env = ["OCICRYPT_KEYPROVIDER_CONFIG=/etc/containerd/ocicrypt/ocicrypt_keyprovider.conf"]
    path = "ctd-decoder"
    returns = "application/vnd.oci.image.layer.v1.tar"

  [stream_processors."io.containerd.ocicrypt.decoder.v1.tar.gzip"]
    accepts = ["application/vnd.oci.image.layer.v1.tar+gzip+encrypted"]
    args = ["--decryption-keys-path", "/etc/containerd/ocicrypt/keys"]
    env = ["OCICRYPT_KEYPROVIDER_CONFIG=/etc/containerd/ocicrypt/ocicrypt_keyprovider.conf"]
    path = "ctd-decoder"
    returns = "application/vnd.oci.image.layer.v1.tar+gzip"

[timeouts]
  "io.containerd.timeout.bolt.open" = "0s"
  "io.containerd.timeout.shim.cleanup" = "5s"
  "io.containerd.timeout.shim.load" = "5s"
  "io.containerd.timeout.shim.shutdown" = "3s"
  "io.containerd.timeout.task.state" = "2s"

[ttrpc]
  address = ""
  gid = 0
  uid = 0
```

**sandbox 镜像源**：设置 Kubernetes 使用的沙箱容器镜像，支持高效管理容器。

- sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.9"

**hugeTLB 控制器**：禁用 hugeTLB 控制器，减少内存管理复杂性，适合不需要的环境。

- disable_hugetlb_controller = true

**网络插件路径**：指定 CNI 网络插件的二进制和配置路径，确保网络功能正常。

- bin_dir = "/opt/cni/bin" 
- conf_dir = "/etc/cni/net.d"  

**垃圾回收调度器**：调整垃圾回收阈值和启动延迟，优化容器资源管理和性能。

- pause_threshold = 0.02
- startup_delay = "100ms"  

**流媒体服务器**：配置流媒体服务的地址和端口，实现与客户端的有效数据传输。

- stream_server_address = "127.0.0.1"  
- stream_server_port = "0"  

​	启动并设置containerd开机自启

```shell
systemctl enable containerd  --now
systemctl status containerd 
```

#### 14.安装Docker-ce(使用docker的拉镜像功能)

​	1）四台主机分别安装docker-ce最新版：

```shell
yum -y install docker-ce
```

​	2）启动并设置docker开机自启：

```shell
systemctl start docker && systemctl enable docker.service
```

​	3）配置docker的镜像加速器地址：

​		注：阿里加速地址登录[阿里云加速器官网](https://developer.aliyun.com/article/1436840#:~:text=%E9%98%BF%E9%87%8C%E4%BA%91%E5%8A%A0%E9%80%9F%E5%99%A8)查看，华为云加速器登录[华为云加速器地址](https://console.huaweicloud.com/swr/?region=cn-east-3#/swr/mirror)查看，每个人的加速地址不同。

```json
tee /etc/docker/daemon.json <<-'EOF'
{
    "registry-mirrors": [
        "https://fb3aq27p.mirror.aliyuncs.com",
		"https://ad7549e6aa7945e5acd79aa85597a630.mirror.swr.myhuaweicloud.com"
    ]
}
EOF
systemctl daemon-reload
systemctl restart docker
systemctl status docker
```

## 二、K8S安装部署

#### 1.安装K8S相关核心组件

​	四台主机分别安装K8S相关核心组件：

```shell
yum -y install  kubelet-1.28.2 kubeadm-1.28.2 kubectl-1.28.2
systemctl enable kubelet
```

- `kubelet` 是 Kubernetes 集群中每个节点上的核心代理，它负责根据控制平面的指示管理和维护节点上的 Pod 及容器的生命周期，确保容器按规范运行并定期与控制平面通信。kubelet 会将节点和 Pod 的状态上报给控制节点的 apiServer，apiServer再将这些信息存储到 etcd 数据库中。
- `kubeadm `是一个用于简化 Kubernetes 集群安装和管理的工具，快速初始化控制平面节点和将工作节点加入集群，减少手动配置的复杂性。
- `kubectl` 是 Kubernetes 的命令行工具，用于管理员与集群进行交互，执行各种任务，如部署应用、查看资源、排查问题、管理集群状态等，通过命令行与 Kubernetes API 直接通信。

#### 2.通过keepalived+nginx实现kubernetes apiServer的节点高可用

​	1）在三台Master节点分别安装keepalived+nginx，实现对apiserver的负载均衡和反向代理。Master01作为keepalived的主节点，Master02和Master03作为keepalived的备用节点。

```shell
yum -y install epel-release nginx keepalived nginx-mod-stream 
```

​	2）修改配置nginx.conf配置文件：

```shell
vim /etc/nginx/nginx.conf
```

​	3）更改配置配置文件完整信息如下：

```yaml
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

include /usr/share/nginx/modules/*.conf;

events {
    worker_connections  1024;
}

# 新增 stream 配置
stream {
    # 日志格式
    log_format  main  '$remote_addr $upstream_addr - [$time_local] $status $upstream_bytes_sent';
    # 日志存放路径
    access_log  /var/log/nginx/k8s-access.log  main;

    # master 调度资源池
    upstream k8s-apiserver {
        server 192.168.116.141:6443 weight=5 max_fails=3 fail_timeout=30s;
        server 192.168.116.142:6443 weight=5 max_fails=3 fail_timeout=30s;
        server 192.168.116.143:6443 weight=5 max_fails=3 fail_timeout=30s;
    }

    server {
        listen 16443;  # 避免与 Kubernetes master 节点冲突
        proxy_pass k8s-apiserver;  # 做反向代理到资源池
    }
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    # 日志格式
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    include /etc/nginx/conf.d/*.conf;
}
```

- `stream`模块用于四层负载均衡，转发到多个`k8s-apiserver`节点。
  - **四层负载均衡**工作在传输层，仅关注 TCP/UDP 连接的信息，例如源IP地址、端口、目标IP地址和端口，基于这些信息进行流量转发，不关心传输的数据内容。
  - **七层负载均衡**工作在应用层，理解更高层的协议（如 HTTP、HTTPS），根据请求的具体内容（例如URL、Cookies、Headers）进行更复杂的流量分配和处理。

- `log_format`和`access_log`用于记录日志。

- `upstream`定义了多个`k8s-apiserver`服务器，负载均衡策略基于权重`weight`，并有故障处理机制。

- `server`块中，`listen`使用端口`16443`，以避免与Kubernetes master节点的默认端口`6443`冲突。

​	4）重启nginx并设置开机自启：

```bash
systemctl restart nginx && systemctl enable nginx
```

​	5）先编写keepalived的检查脚本：

​		注：将这个脚本分别放到三台Master节点上，建议放在/etc/keepalived/目录，方便后续进行优化更新。

```shell
#!/bin/bash

#   Author          ：lyx
#   Description     ：check nginx 
#   Date            ：2024.10.06
#   Version         ：2.3

# 定义日志文件路径
LOG_FILE="/var/log/nginx_keepalived_check.log"
MAX_LINES=1000  # 设置日志保留1000行（因为不做限制日志会无限增大，占用大量磁盘空间）

# 记录日志的函数，带有详细的时间格式，并保留最后1000行日志
log_message() {
    local time_stamp=$(date '+%Y-%m-%d %H:%M:%S')  # 定义时间格式
    echo "$time_stamp - $1" >> $LOG_FILE
    
    # 截取日志文件，只保留最后1000行
    tail -n $MAX_LINES $LOG_FILE > ${LOG_FILE}.tmp && mv ${LOG_FILE}.tmp $LOG_FILE
}

# 检测 Nginx 是否在运行的函数
check_nginx() {
    pgrep -f "nginx: master" > /dev/null 2>&1
    echo $?
}

# 1. 检查 Nginx 是否存活
log_message "正在检查 Nginx 状态..."
if [ $(check_nginx) -ne 0 ]; then
    log_message "Nginx 未运行，尝试启动 Nginx..."

    # 2. 如果 Nginx 不在运行，则尝试启动
    systemctl start nginx
    sleep 2  # 等待 Nginx 启动

    # 3. 再次检查 Nginx 状态
    log_message "启动 Nginx 后再次检查状态..."
    if [ $(check_nginx) -ne 0 ]; then
        log_message "Nginx 启动失败，停止 Keepalived 服务..."
        
        # 4. 如果 Nginx 启动失败，停止 Keepalived
        systemctl stop keepalived
    else
        log_message "Nginx 启动成功。"
    fi
else
    log_message "Nginx 正常运行。"
fi

```

- 若nginx出现问题会打印中文日志到：/var/log/nginx_keepalived_check.log ， 可以手动进行查看输出的日志消息，再根据对应故障的具体时间，结合nginx的日志来进行修复或者优化。

​	分别授予脚本可执行权限：

```shell
chmod +x /etc/keepalived/keepalived_nginx_check.sh
```

​	这个脚本的主要作用是**监控 Nginx 服务的运行状态**，并在检测到 Nginx 停止运行时，尝试重启它。如果重启失败，脚本会停止 Keepalived 服务，避免继续提供不可用的服务。

- **监控 Nginx 状态**：脚本定期检查 Nginx 是否正常运行，使用 `pgrep` 命令检测主进程状态。
- **自动修复机制**：如果 Nginx 未运行，尝试重启服务，并再次检测其状态；若重启成功，记录日志。
- **停止 Keepalived**：如果 Nginx 无法启动，停止 Keepalived 服务，防止服务器继续作为故障节点运行。

​	6）修改keepalived主节点Master01的配置文件：

```shell
global_defs {
   notification_email {
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 127.0.0.1   # 用来发送、接收和中转电子邮件的服务器
   smtp_connect_timeout 30
   router_id LVS_DEVEL
   vrrp_skip_check_adv_addr
   vrrp_strict
   vrrp_garp_interval 0
   vrrp_gna_interval 0
}

vrrp_script keepalived_nginx_check { # 这里是上一步骤添加的脚本，在这里进行调用
    script "/etc/keepalived/keepalived_nginx_check.sh" # 根据自己添加的脚本路径进行修改，建议还是放在这个目录下便于管理
}  

vrrp_instance VI_1 {
    state MASTER		# 主修改state为MASTER，备修改为BACKUP
    interface ens160    # 修改自己的实际网卡名称
    virtual_router_id 51  # 主备的虚拟路由ID要相同
    priority 100        # 优先级，备服务器设置优先级比主服务器的优先级低一些
    advert_int 1        # 广播包发送间隔时间为1秒
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.116.16/24  # 虚拟IP修改为没有占用的IP地址，主备的虚拟IP相同就好
    }
    track_script {
        keepalived_nginx_check # vrrp_script 定义的脚本名，放到这里进行追踪调用，Keepalived 可以根据脚本返回的结果做出相应的动作
    }
}
```

​	7）修改keepalived备份节点Master02的配置文件：

```shell
global_defs {
   notification_email {
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 127.0.0.1   # 用来发送、接收和中转电子邮件的服务器
   smtp_connect_timeout 30
   router_id LVS_DEVEL
   vrrp_skip_check_adv_addr
   vrrp_strict
   vrrp_garp_interval 0
   vrrp_gna_interval 0
}

vrrp_script keepalived_nginx_check { # 这里是上一步骤添加的脚本，在这里进行调用
    script "/etc/keepalived/keepalived_nginx_check.sh" # 根据自己添加的脚本路径进行修改，建议还是放在这个目录下便于管理
}  

vrrp_instance VI_1 {
    state BACKUP		# 主修改state为MASTER，备修改为BACKUP
    interface ens160    # 修改自己的实际网卡名称
    virtual_router_id 51  # 主备的虚拟路由ID要相同
    priority 90         # 优先级，备服务器设置优先级比主服务器的优先级低一些
    advert_int 1        # 广播包发送间隔时间为1秒
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.116.16/24  # 虚拟IP修改为没有占用的IP地址，主备的虚拟IP相同就好
    }
    track_script {
        keepalived_nginx_check # vrrp_script 定义的脚本名，放到这里进行追踪调用，Keepalived 可以根据脚本返回的结果做出相应的动作
    }
}
```

​	7）修改keepalived备份节点Master03的配置文件：

```shell
global_defs {
   notification_email {
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc
   smtp_server 127.0.0.1   # 用来发送、接收和中转电子邮件的服务器
   smtp_connect_timeout 30
   router_id LVS_DEVEL
   vrrp_skip_check_adv_addr
   vrrp_strict
   vrrp_garp_interval 0
   vrrp_gna_interval 0
}

vrrp_script keepalived_nginx_check { # 这里是上一步骤添加的脚本，在这里进行调用
    script "/etc/keepalived/keepalived_nginx_check.sh" # 根据自己添加的脚本路径进行修改，建议还是放在这个目录下便于管理
}  

vrrp_instance VI_1 {
    state BACKUP		# 主修改state为MASTER，备修改为BACKUP
    interface ens160    # 修改自己的实际网卡名称
    virtual_router_id 51  # 主备的虚拟路由ID要相同
    priority 80         # 优先级，备服务器设置优先级比主服务器的优先级低一些
    advert_int 1        # 广播包发送间隔时间为1秒
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.116.16/24  # 虚拟IP修改为没有占用的IP地址，主备的虚拟IP相同就好
    }
    track_script {
        keepalived_nginx_check # vrrp_script 定义的脚本名，放到这里进行追踪调用，Keepalived 可以根据脚本返回的结果做出相应的动作
    }
}
```

- 三台节点主要的修改内容为这三部分：
  - state MASTER / BACKUP
  - interface ens160
  - priority 100 / 90 / 80

​	8）重载配置文件并重启nginx+keepalived服务：

```shell
systemctl daemon-reload && systemctl restart nginx 
systemctl restart keepalived && systemctl enable keepalived
```

​	9）检查虚拟IP是否绑定成功：

```shell
ip address show | grep 192.168.116.16	# 根据你们自己设置的虚拟IP来检查
```

​	如果成功绑定会有如下信息：

> [!IMPORTANT]
>
> inet **192.168.116.16/24** scope global secondary ens160

​	10）检查keepalived漂移是否设置成功（keepalived_nginx_check.sh脚本是否生效）

​		（1）在keepalived的主节点Master01关闭keepalived服务：

```shell
systemctl stop keepalived
```

​		（2）切换到keepalived的备节点Master01查看网卡信息

```shell
ip address show | grep 192.168.116.16	# 根据你们自己设置的虚拟IP来检查
```

​		若出现如下提示，即为漂移成功：

> [!IMPORTANT]
>
> inet 192.168.116.16/24 scope global secondary ens160

​	注：如果主keepalived不异常的情况下，在两个备节点是无法查看到192.168.116.16虚拟IP的。

#### 3.初始化K8S集群

1）**Master01节点**分别使用kubeadm初始化K8S集群：

​	注：<u>kubeadm安装K8S，控制节点和工作节点的组件都是基于**Pod**运行的。</u>

```shell
kubeadm config print init-defaults > kubeadm.yaml
```

- 生成默认的配置文件重定向输出到 kubeadm.yaml 中		

​	2）修改刚刚用kubeadm生成的kubeadm.yaml文件：

```shell
sed -i '/localAPIEndpoint/s/^/#/' kubeadm.yaml
sed -i '/advertiseAddress/s/^/#/' kubeadm.yaml
sed -i '/bindPort/s/^/#/' kubeadm.yaml
sed -i '/name: node/s/^/#/' kubeadm.yaml
sed -i "s|criSocket:.*|criSocket: unix://$(find / -name containerd.sock | head -n 1)|" kubeadm.yaml
sed -i 's|imageRepository: registry.k8s.io|imageRepository: registry.aliyuncs.com/google_containers|' kubeadm.yaml	# 原配置为国外的k8s源，为了加速镜像的下载，需改成国内源
sed -i '/serviceSubnet/a\  podSubnet: 10.244.0.0/12' kubeadm.yaml  # /a\ 表示在serviceSubnet行下方一行内容
cat <<EOF >> kubeadm.yaml
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
EOF

more kubeadm.yaml  # 手动检查一下
```

- `advertiseAddress` 是 Kubernetes 控制节点的广告地址，其他节点通过这个地址与控制平面节点通信。它通常是控制节点所在服务器的 IP 地址，为了确保控制平面节点能在网络中通过正确的**控制节点 IP 地址**（我的MasterIP为：192.168.116.131）进行通信。

- `criSocket` 指定的是 Kubernetes 使用的容器运行时（CRI）套接字地址，K8S 使用这个套接字与容器运行时（如 containerd）进行通信，来管理和启动容器。为了确保 K8S使用正确的容器运行时套接字。通过 find 命令查找 containerd.sock 文件路径并替换进配置文件，可以保证路径的准确性，避免手动查找和配置错误。

- `IPVS` 模式支持更多的负载均衡算法，性能更好，尤其在集群节点和服务较多的情况下，可以显著提升网络转发效率和稳定性（如果没有指定mode为ipvs，则默认选定iptables，iptables性能相对较差）。

- 统一使用 `systemd` 作为容器和系统服务的 `cgroup` 驱动，避免使用 `cgroupfs` 时可能产生的资源管理不一致问题，提升 Kubernetes 和宿主机系统的兼容性和稳定性。

  注：<u>主机 IP、Pod IP 和 Service IP **不能在同一网段**，因会导致 IP 冲突、路由混乱及网络隔离失败，影响 Kubernetes 的正常通信和网络安全。</u>

​	3）基于kubeadm.yaml 文件初始化K8S，**Master01节点**拉取 Kubernetes 1.28.0 所需的镜像（两个方法可以二选一）：

​		（1）使用使用 `kubeadm` 命令，快速拉取 Kubernetes 所有核心组件的镜像，并确保版本一致。

```shell
kubeadm config images pull --image-repository="registry.aliyuncs.com/google_containers" --kubernetes-version=v1.28.0
```

​		（2）使用 `ctr` 命令，需要更细粒度的控制，或在 `kubeadm` 拉取镜像过程中出现问题时，可以**使用 `ctr` 命令手动拉取**镜像。

```sh
ctr -n=k8s.io images pull registry.aliyuncs.com/google_containers/kube-apiserver:v1.28.0
ctr -n=k8s.io images pull registry.aliyuncs.com/google_containers/kube-controller-manager:v1.28.0
ctr -n=k8s.io images pull registry.aliyuncs.com/google_containers/kube-scheduler:v1.28.0
ctr -n=k8s.io images pull registry.aliyuncs.com/google_containers/kube-proxy:v1.28.0
ctr -n=k8s.io images pull registry.aliyuncs.com/google_containers/pause:3.9
ctr -n=k8s.io images pull registry.aliyuncs.com/google_containers/etcd:3.5.9-0
ctr -n=k8s.io images pull registry.aliyuncs.com/google_containers/coredns:v1.10.1
```

​	4）在Master控制节点，初始化 Kubernetes 主节点

```sh
kubeadm init --config=kubeadm.yaml --ignore-preflight-errors=SystemVerification
```

​	个别操作系统可能会出现kubelet启动失败的情况，如下提示，如果提示successfully则忽略以下步骤：

> [!WARNING]
>
> dial tcp [::1]:10248: connect: connection refused

​	执行`systemctl status kubelet`发现出现以下错误提示：	

> [!WARNING]
>
> Process: 2226953 ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS (code=exited, status=1/FAILURE)
> Main PID: 2226953 (code=exited, status=1/FAILURE)

​	解决方法如下，**控制节点**执行：

```shell
sed -i 's|ExecStart=/usr/bin/kubelet|ExecStart=/usr/bin/kubelet --container-runtime-endpoint=unix://$(find / -name containerd.sock | head -n 1) --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml|' /usr/lib/systemd/system/kubelet.service

systemctl daemon-reload
systemctl restart kubelet

kubeadm reset # 删除安装出错的K8S
kubeadm init --config=kubeadm.yaml --ignore-preflight-errors=SystemVerification # 重新安装
```

#### 3.设置 Kubernetes 的配置文件，以便让当前用户能够使用 `kubectl` 命令与 Kubernetes 集群进行交互

​	控制节点Master01执行：

```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### 4.扩容k8s控制节点，将Master01和Master02加入到k8s集群

​	1）Master02和Master03创建证书存放目录：

```shell
mkdir -pv /etc/kubernetes/pki/etcd && mkdir -pv ~/.kube/
```

​	2）远程拷贝证书文件到Master02：

```shell
scp /etc/kubernetes/pki/ca.crt root@192.168.116.142:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/ca.key root@192.168.116.142:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/sa.key root@192.168.116.142:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/sa.pub root@192.168.116.142:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/front-proxy-ca.crt root@192.168.116.142:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/front-proxy-ca.key root@192.168.116.142:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/etcd/ca.crt root@192.168.116.142:/etc/kubernetes/pki/etcd/
scp /etc/kubernetes/pki/etcd/ca.key root@192.168.116.142:/etc/kubernetes/pki/etcd/
```

​	3）远程拷贝证书文件到Master03：

```shell
scp /etc/kubernetes/pki/ca.crt root@192.168.116.143:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/ca.key root@192.168.116.143:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/sa.key root@192.168.116.143:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/sa.pub root@192.168.116.143:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/front-proxy-ca.crt root@192.168.116.143:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/front-proxy-ca.key root@192.168.116.143:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/etcd/ca.crt root@192.168.116.143:/etc/kubernetes/pki/etcd/
scp /etc/kubernetes/pki/etcd/ca.key root@192.168.116.143:/etc/kubernetes/pki/etcd/
```

​	4）控制节点Master01生成集群token：

```shell
kubeadm token create --print-join-command
```

​	生成token如下：

> [!IMPORTANT]
>
> kubeadm join 192.168.116.141:6443 --token pb1pk7.6p6w2jl1gjvlmmdz --discovery-token-ca-cert-hash sha256:b3b9de172cf6c48d97396621858a666e0be2d2d5578e4ce0fba5f1739b735fc1 

- 如果**控制节点**加入集群需要在生成的token后加入 `--control-plane` ，如下：

  ```shell
  kubeadm join 192.168.116.141:6443 --token pb1pk7.6p6w2jl1gjvlmmdz --discovery-token-ca-cert-hash sha256:b3b9de172cf6c48d97396621858a666e0be2d2d5578e4ce0fba5f1739b735fc1 --control-plane
  ```

  - 控制节点加入集群后执行如下命令，即可使用kubectl命令工作：

    ```shell
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ```

-  如果**工作节点**加入集群，直接复制生成的这行命令到node主机执行即可：

  ```shell
  kubeadm join 192.168.116.141:6443 --token pb1pk7.6p6w2jl1gjvlmmdz --discovery-token-ca-cert-hash sha256:b3b9de172cf6c48d97396621858a666e0be2d2d5578e4ce0fba5f1739b735fc1
  ```

  - 工作节点加入集群后，设置一个用户的 `kubectl` 环境，使其能够与 Kubernetes 集群进行交互：：

    ```
    mkdir ~/.kube
    cp /etc/kubernetes/kubelet.conf  ~/.kube/config
    ```

  ​	**注意：**如果扩容节点时，有以下报错：

  > [!CAUTION]
  >
  >
  > [preflight] Reading configuration from the cluster...
  > [preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
  > error execution phase preflight: 
  > One or more conditions for hosting a new control plane instance is not satisfied.
  >
  > unable to add a new control plane instance a cluster that doesn't have a stable controlPlaneEndpoint address
  >
  > Please ensure that:
  > * The cluster has a stable controlPlaneEndpoint address.
  > * The certificates that must be shared among control plane instances are provided.
  >
  >
  > To see the stack trace of this error execute with --v=5 or higher

  - 解决方法为添加 `controlPlaneEndpoint`：

  ```shell
  kubectl -n kube-system edit cm kubeadm-config
  ```

  - 添加如下 `controlPlaneEndpoint: 192.168.116.141:6443` (Master01的IP地址) 即可解决：

  ```shell
  kind: ClusterConfiguration
  kubernetesVersion: v1.18.0
  controlPlaneEndpoint: 192.168.116.141:6443  
  ```

  - 重新复制生成的这行命令到node主机执行（控制节点需要添加 `--control-plane`），提示为如下内容则为成功扩容：

    ```shell
    To start administering your cluster from this node, you need to run the following as a regular user:
    
        mkdir -p $HOME/.kube
        sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
        sudo chown $(id -u):$(id -g) $HOME/.kube/config
    
    Run 'kubectl get nodes' to see this node join the cluster.
    ```

#### 5.安装k8s网络组件Calico

​	Calico 是一个流行的开源网络解决方案，专为 Kubernetes 提供高效、可扩展和安全的网络连接。它采用了基于 IP 的网络模型，使每个 Pod 都能获得一个唯一的 IP 地址，从而简化了网络管理。Calico 支持多种网络策略，可以实现细粒度的流量控制和安全策略，例如基于标签的访问控制，允许用户定义哪些 Pod 可以相互通信。（简单来说就是给Pod和Service分IP的,还能通过网络策略做网络隔离）

​	1）四台主机分别安装calico：

```shell
ctr image pull -n=k8s.io swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/calico/cni:v3.25.0
ctr image pull -n=k8s.io swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/calico/pod2daemon-flexvol:v3.25.0
ctr image pull -n=k8s.io swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/calico/node:v3.25.0
ctr image pull -n=k8s.io swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/calico/kube-controllers:v3.25.0
ctr image pull -n=k8s.io swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/calico/typha:v3.25.0
```

​	2) 控制节点下载calico3.25.0的yaml配置文件(下载失败把URL复制到浏览器，手动复制粘贴内容到Master01节点效果相同)

```shell
curl -O -L https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml
```

​	3）编辑calico.yaml，找到CLUSTER_TYPE行，在下面添加一对键值对，确保使用网卡接口（注意缩进）：

​	原配置：

```yaml
- name: CLUSTER_TYPE
  value: "k8s,bgp"
```

​	新配置：

```yaml
 - name: CLUSTER_TYPE
  value: "k8s,bgp" 
 - name: IP_AUTODELECTION_METHOD
  value: "interface=ens160"
```

​	注：不同操作系统的网卡名称有差异，例：centos7.9的网卡名称为ens33，就要填写value: "interface=ens33"，需灵活变通。

​	注：如果出现calico拉取镜像错误问题，可能是没有修改imagePullPresent规则，可以修改官方源下载为华为源下载，如下：

```shell
sed -i '1,$s|docker.io/calico/cni:v3.25.0|swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/calico/cni:v3.25.0|g' calico.yaml
sed -i '1,$s|docker.io/calico/node:v3.25.0|swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/calico/node:v3.25.0|g' calico.yaml
sed -i '1,$s|docker.io/calico/kube-controllers:v3.25.0|swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/calico/kube-controllers:v3.25.0|g' calico.yaml
```

​	4）部署calico网络服务

```shell
kubectl apply -f calico.yaml 
```

​	查看在 Kubernetes 集群中查看属于 `kube-system` 命名空间的所有 Pod 的详细信息（控制节点和工作节点都查的到）：

```shell
kubectl get pod --namespace kube-system -o wide 
```

​	calico安装成功的信息大概如下：

> [!IMPORTANT]
>
> NAME                                       READY   STATUS    RESTARTS      AGE    IP                NODE       NOMINATED NODE   READINESS GATES
> calico-kube-controllers-665548954f-5dgjw   1/1     Running   0             3m3s   10.255.112.131    master01   <none>           <none>
> calico-node-4gb27                          1/1     Running   0             3m3s   192.168.116.142   master02   <none>           <none>
> calico-node-4kckd                          1/1     Running   0             3m3s   192.168.116.144   node       <none>           <none>
> calico-node-cpnwp                          1/1     Running   0             3m3s   192.168.116.141   master01   <none>           <none>
> calico-node-hldt2                          1/1     Running   0             3m3s   192.168.116.143   master03   <none>           <none>
> coredns-66f779496c-8pzvp                   1/1     Running   0             61m    10.255.112.129    master01   <none>           <none>
> coredns-66f779496c-frsvq                   1/1     Running   0             61m    10.255.112.130    master01   <none>           <none>
> etcd-master01                              1/1     Running   0             61m    192.168.116.141   master01   <none>           <none>
> etcd-master02                              1/1     Running   0             21m    192.168.116.142   master02   <none>           <none>
> etcd-master03                              1/1     Running   0             20m    192.168.116.143   master03   <none>           <none>
> kube-apiserver-master01                    1/1     Running   0             61m    192.168.116.141   master01   <none>           <none>
> kube-apiserver-master02                    1/1     Running   0             22m    192.168.116.142   master02   <none>           <none>
> kube-apiserver-master03                    1/1     Running   1 (21m ago)   19m    192.168.116.143   master03   <none>           <none>
> kube-controller-manager-master01           1/1     Running   1 (21m ago)   61m    192.168.116.141   master01   <none>           <none>
> kube-controller-manager-master02           1/1     Running   0             22m    192.168.116.142   master02   <none>           <none>
> kube-controller-manager-master03           1/1     Running   0             20m    192.168.116.143   master03   <none>           <none>
> kube-proxy-jvt6w                           1/1     Running   0             31m    192.168.116.144   node       <none>           <none>
> kube-proxy-lw8g4                           1/1     Running   0             61m    192.168.116.141   master01   <none>           <none>
> kube-proxy-mjw8h                           1/1     Running   0             22m    192.168.116.142   master02   <none>           <none>
> kube-proxy-rtlpz                           1/1     Running   0             21m    192.168.116.143   master03   <none>           <none>
> kube-scheduler-master01                    1/1     Running   1 (21m ago)   61m    192.168.116.141   master01   <none>           <none>
> kube-scheduler-master02                    1/1     Running   0             22m    192.168.116.142   master02   <none>           <none>
> kube-scheduler-master03                    1/1     Running   0             19m    192.168.116.143   master03   <none>           <none>

#### 6.配置Etcd的高可用

​	Etcd默认的yaml文件--initial-cluster只指定自己，所以需要修改为指定我们的三台Master节点主机：

​	**Master01节点**

```bash
sed -i 's|--initial-cluster=master01=https://192.168.116.141:2380|--initial-cluster=master01=https://192.168.116.141:2380,master02=https://192.168.116.142:2380,master03=https://192.168.116.143:2380|' /etc/kubernetes/manifests/etcd.yaml
```

​	**Master02节点**

```bash
sed -i 's|--initial-cluster=master01=https://192.168.116.141:2380|--initial-cluster=master01=https://192.168.116.142:2380,master02=https://192.168.116.142:2380,master03=https://192.168.116.143:2380|' /etc/kubernetes/manifests/etcd.yaml
```

​	**Master03节点**

```bash
sed -i 's|--initial-cluster=master01=https://192.168.116.141:2380|--initial-cluster=master01=https://192.168.116.143:2380,master02=https://192.168.116.142:2380,master03=https://192.168.116.143:2380|' /etc/kubernetes/manifests/etcd.yaml
```

- 修改的目的是在 **Kubernetes 高可用（HA）集群** 中为 **etcd 集群** 提供节点的初始集群信息。简单来说就是，`--initial-cluster` 的配置作用是告诉 etcd 节点关于集群中的其他成员信息，这样它们可以相互通信、保持一致并提供高可用性保障。

#### 7.k8s的apiserver证书续期

​	查看 Kubernetes API 服务器的证书有效期：

```shell
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text | grep Not
```

> [!CAUTION]
>
>             Not Before: Oct  6 14:09:37 2024 GMT
>             Not After : Oct  6 14:55:00 2025 GMT

​	使用kubeadm部署k8s自动创建的crt证书有效期是1年时间，为了确保证书长期有效，这里选择修改crt证书的过期时间为100年，下载官方提供的 `update-kubeadm-cert.sh` 脚本（在Master控制节点操作）：

```shell
git clone https://github.com/yuyicai/update-kube-cert.git
cd update-kube-cert
chmod 755 update-kubeadm-cert.sh

sed -i '1,$s/CERT_DAYS=3650/CERT_DAYS=36500/g' update-kubeadm-cert.sh
./update-kubeadm-cert.sh all
./update-kubeadm-cert.sh all --cri containerd
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text | grep Not
```

> [!CAUTION]
>
>             Not Before: Oct  6 15:43:28 2024 GMT
>             Not After : Sep 12 15:43:28 2124 GMT

​	证书时效延长成功！