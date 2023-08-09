## 生产环境部署k8s集群
### 环境准备

1、准备三台机器

k8s-master（172.16.10.11）
k8s-node1（172.16.10.12）
k8s-node2 （172.16.10.13）

2、三台机器分别修改 /etc/hosts文件

```shell
172.16.10.11 k8s-master
172.16.10.12 k8s-node1
172.16.10.13 k8s-node2
```

3、三台机器分别修改主机名

```
hostnamectl set-hostname k8s-master(需替换)
```

4、关闭防火墙、selinux、swap分区

```shell
swapoff -a
#永久禁用
#若需要重启后也生效，在禁用swap后还需修改配置文件/etc/fstab，注释swap
sed -i.bak '/swap/s/^/#/' /etc/fstab

systemctl stop firewalld
systemctl disable firewalld
sed -i 's/enforcing/disabled/' /etc/selinux/config
setenforce 0
echo "net.bridge.bridge-nf-call-iptables = 1 ">>/etc/sysctl.d/k8s.conf
echo "net.bridge.bridge-nf-call-ip6tables = 1 ">>/etc/sysctl.d/k8s.conf
sysctl --system
```



### 安装docker（已被k8s弃用，由containerd替代）

```
wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
yum install docker-ce
systemctl enable docker
systemctl start docker
docker version
```
### 安装containerd

```shell
wget https://github.com/containerd/containerd/releases/download/v1.7.1/containerd-1.7.1-linux-amd64.tar.gz
# curl -L "https://github.com/containerd/containerd/releases/download/v1.7.1/containerd-1.7.1-linux-amd64.tar.gz" | sudo tar -xz
if [ $? != 0 ];then
echo "下载安装文件失败，请检查网络或稍后再试！"
exit
fi

tar xvf containerd-1.7.1-linux-amd64.tar.gz
mv -fr bin/* /usr/local/bin/
cat > /lib/systemd/system/containerd.service << \EOF

[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target

[Service]
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd

Type=notify
Delegate=yes
KillMode=process
Restart=always
RestartSec=5
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity
LimitNOFILE=infinity
# Comment TasksMax if your systemd version does not supports it.
# Only systemd 226 and above support this version.
TasksMax=infinity
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target
EOF


mkdir /etc/containerd
# 输出默认配置文件
# containerd config default > /etc/containerd/config.toml
# 这里使用我修改后的,调整了三个地方，sandbox_image、systemd_cgroup=true 、runtime_type = "io.containerd.runtime.v1.linux"
cat > /etc/containerd/config.toml << \EOF
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
    enable_selinux = false
    enable_tls_streaming = false
    enable_unprivileged_icmp = false
    enable_unprivileged_ports = false
    ignore_image_defined_volumes = false
    max_concurrent_downloads = 3
    max_container_log_line_size = 16384
    netns_mounts_under_state_dir = false
    restrict_oom_score_adj = false
    sandbox_image = "docker.io/alan0310/pause:3.9"
    selinux_category_range = 1024
    stats_collect_period = 10
    stream_idle_timeout = "4h0m0s"
    stream_server_address = "127.0.0.1"
    stream_server_port = "0"
    systemd_cgroup = true
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
          runtime_type = "io.containerd.runtime.v1.linux"

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
            SystemdCgroup = true

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
      config_path = ""

      [plugins."io.containerd.grpc.v1.cri".registry.auths]

      [plugins."io.containerd.grpc.v1.cri".registry.configs]

      [plugins."io.containerd.grpc.v1.cri".registry.headers]

      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]

    [plugins."io.containerd.grpc.v1.cri".x509_key_pair_streaming]
      tls_cert_file = ""
      tls_key_file = ""

  [plugins."io.containerd.internal.v1.opt"]
    path = "/opt/containerd"

  [plugins."io.containerd.internal.v1.restart"]
    interval = "10s"

  [plugins."io.containerd.internal.v1.tracing"]
    sampling_ratio = 1.0
    service_name = "containerd"

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

  [plugins."io.containerd.snapshotter.v1.btrfs"]
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
    root_path = ""
    upperdir_label = false

  [plugins."io.containerd.snapshotter.v1.zfs"]
    root_path = ""

  [plugins."io.containerd.tracing.processor.v1.otlp"]
    endpoint = ""
    insecure = false
    protocol = ""

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
EOF
#启动设置开机自启动
systemctl restart containerd && systemctl enable containerd 

```

### 安装runc

```shell
wget https://github.com/opencontainers/runc/releases/download/v1.1.7/runc.amd64
cp runc.amd64  /usr/bin/runc
chmod a+x /usr/bin/runc
```

### 安装cni

```shell
sudo mkdir -p /opt/cni/bin
curl -L "https://github.com/containernetworking/plugins/releases/download/v1.2.0/cni-plugins-linux-amd64-v1.2.0.tgz" | sudo tar -C /opt/cni/bin -xz
```

### 安装crictl

```shell
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.22.0/crictl-v1.22.0-linux-amd64.tar.gz
sudo tar zxvf crictl-v1.22.0-linux-amd64.tar.gz -C /usr/local/bin
rm -f crictl-v1.22.0-linux-amd64.tar.gz
cat >  /etc/crictl.yaml << EOF
runtime-endpoint: unix:///var/run/containerd/containerd.sock
image-endpoint: unix:///var/run/containerd/containerd.sock
timeout: 30
debug: false
pull-image-on-create: false
EOF

```

查看版本信息

`crictl version`

正常输出

![image](https://github.com/alan-et/k8s-components/assets/46310121/6a0522f6-c021-4123-a2e9-d6657f2e78a3)


**如果输出错误`getting the runtime version: rpc error: code = Unimplemented desc = unknown service runtime.v1alpha2.RuntimeService`**

请查看`/etc/containerd/config.toml`文件是否没有调整runtime_type值

### 安装kubelet kubeadm kubect

1、修改存储库

**注意: 所有节点需执行**

```shell
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

2、安装并启动

**注意: 所有节点需执行**

```shell
yum install -y kubelet-1.27.2-0 kubectl-1.27.2-0 kubeadm-1.27.2-0
systemctl enable kubelet 
systemctl start kubelet
```

3、主节点初始化

```shell
kubeadm init  --image-repository docker.io/alan0310 --kubernetes-version v1.27.2 --apiserver-advertise-address 172.16.10.11 --pod-network-cidr=172.244.0.0/16 --service-cidr=172.1.0.0/16
```

***其中apiserver-advertise-address参数必须和你自己的ip对应；仓库地址（--image-repository）也可以使用阿里云的registry.aliyuncs.com/google_containers***

![image](https://github.com/alan-et/k8s-components/assets/46310121/4a6a29be-afbe-4e55-82d4-1031d4375b54)


4、初始化后注意提示，执行以上脚本

**其中kubeadm join先不急**

```shell
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

查看`kubectl get nodes`发现master还是处于NotReady状态

![image](https://github.com/alan-et/k8s-components/assets/46310121/a474bbd4-f2fa-440e-8cce-4d6b6292f87f)


5、安装网络插件flannel

```shell
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
kubectl apply -f kube-flannel.yml 
kubectl get pods -n kube-system
kubectl get nodes
```

注意：--pod-network-cidr=172.244.0.0/16的ip段需要和kube-flannel.yml文件里面的一致

```shell
  net-conf.json: |
    {
      "Network": "172.244.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }

```



稍等一会再次查看`kubectl get nodes`已经是Ready状态了

![image](https://github.com/alan-et/k8s-components/assets/46310121/1aeededc-4ae3-4c9c-8917-4f572b7cd369)


6、其他节点加入集群

```shell
kubeadm join 172.16.10.11:6443 --token k1aitf.zf78saxhn7p4vj9g \
	--discovery-token-ca-cert-hash sha256:5426847822104c2fd6caa92e9a7d1e5f496f318d0a2e06c6adb0e0ee3614ca30
```
### 安装管理工具

1、cockpit管理kubernetes集群

```shell
yum install cockpit
yum install cockpit-docker.x86_64
yum install cockpit-kubernetes.x86_64
systemctl enable cockpit.socket 
systemctl start cockpit.socket
```

登陆控制台 https://IP:9090 账号密码为服务器的登陆密码

![image](https://github.com/alan-et/k8s-components/assets/46310121/2bc1fe1f-f63d-4bc0-8e9d-1fedda78ca73)


2、kubernetes-dashboard管理集群

文件地址:[https://github.com/alan-et/k8s-components/blob/main/kubernetes-dashboard/deploy-2.3.1.yaml](https://github.com/alan-et/k8s-components/blob/main/kubernetes-dashboard/deploy-2.3.1.yaml)

```shell
kubectl apply -f deploy-2.3.1.yaml
```

有自己证书的需要手动创建然后替换其中的证书名称，可以参考如下文章

[https://blog.csdn.net/weixin_43225813/article/details/130825342](https://blog.csdn.net/weixin_43225813/article/details/130825342)

创建ServiceAccount

`vim admin-user.yaml`

```shell

apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: admin-user
    namespace: kubernetes-dashboard
```

```shell
kubectl apply -f admin-user.yaml
```

获取token登陆

```shell
kubectl create token admin-user -n kubernetes-dashboard
```

浏览器打开https://IP:30001

![image](https://github.com/alan-et/k8s-components/assets/46310121/e449d044-71ca-4073-96b8-37a148907335)


![image](https://github.com/alan-et/k8s-components/assets/46310121/1b1ce8b8-91a1-4b0f-89f3-960c2944cf79)


