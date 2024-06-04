### Linux CentOS 安装docker



#### 设置阿里云仓库

```shell
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```



#### docker-ce安装

```shel
yum install -y docker-ce
```



#### 启动docker并设置开机自启

```she
#启动docker命令
systemctl start docker
#设置开机自启命令
systemctl enable docker
#查看docker版本命令
docker version
```



#### 转存镜像

```she
# 登陆
docker login --username alan0310

# 拉取需要转存的镜像
docker pull quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.30.0

# 查看镜像ID
docker images

# 给镜像打上标签，标签名称就是自己的存储库
docker tag 89ccad40ce8e alan0310/nginx-ingress-controller:0.30.0

# 推送到自己的仓库
docker push alan0310/nginx-ingress-controller
```



#### 运行镜像

```shell
 docker run --rm p 81:5000 dc75e2783557
```

