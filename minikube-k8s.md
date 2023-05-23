## 基于minikube安装k8s（适用于本地环境使用）

1. 更新系统
   sudo yum update -y

2. 安装 Docker 设置开机自启
sudo yum install -y docker
sudo systemctl start docker
sudo systemctl enable docker

3. 安装 kubectl

   kubectl 是 Kubernetes 的命令行管理工具

```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```



4. 安装 Minikube

   不推荐在root用户下安装

   先创建一个用户，添加到docker用户组

   ```
   useradd -m minikubeuser
   usermod -aG docker minikubeuser
   ```

   设置密码

   `sudo passwd minikubeuser`

   切换到新用户

   `su minikubeuser `

   ```
   curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
   chmod +x minikube
   sudo mv minikube /usr/local/bin/
   ```

   

   

5. 启动集群环境

   ```
   minikube start --driver=docker
   ```

   或者

   ```
   minikube start --driver=docker --image-mirror-country=CN
   ```



## 安装helm（可选）

备注：最好不要基于二进制版本安装，会有很多问题

方式一: 源码下载编译安装(需要go环境)

```
git clone https://github.com/helm/helm.git
cd helm
make
sudo cp ./bin/helm /usr/local/bin/
helm version
```

如果没有go环境

下载GO（helm依赖go至少V1.2版本）

`curl -O https://golang.org/dl/go1.20.linux-amd64.tar.gz`

下载完成后，解压这个 tar.gz 文件到 /usr/local 目录

`sudo tar -C /usr/local -xzf go1.20.linux-amd64.tar.gz`

将 Go 语言的 bin 目录添加到 PATH 环境变量中，在/etc/profile最后面添加
`export PATH=$PATH:/usr/local/go/bin`

保存退出，然后执行
`source ~/.bash_profile`
换一个国内能访问的代理地址： https://goproxy.cn,
`go env -w GOPROXY=https://goproxy.cn`

方式二: 直接下载压缩包解压
https://github.com/kubernetes/helm/releases
tar -xvzf $HELM.tar.gz
mv linux-amd64/helm /usr/local/bin/helm



## 安装kubernetes-dashboard

### 方式一：基于helm安装

```
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
helm repo update
kubectl create namespace kubernetes-dashboard
helm install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --namespace kubernetes-dashboard
```



### 方式二：基于yaml文件安装

针对需要调整里面的配置场景，比如自定义证书，手动设置nodeport

下载文件deploy.yaml

https://github.com/alan-et/k8s-components/blob/main/kubernetes-dashboard/deploy.yaml 

####  准备证书
##### 创建k8s secret
```
kubectl create secret tls 自定义命名 --key 证书key路径/证书名.key --cert 证书key路径/证书名.pem或crt -n kubernetes-dashboard
```
##### 查看secret
```
kubectl describe  secret xxxxxx-secret
```
输出样例
```
Name:         xxxxxx-secret
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  kubernetes.io/tls

Data
====
tls.crt:  3805 bytes
tls.key:  1675 bytes

```
##### 在deploy.yaml中找到
```
 containers:
        - name: kubernetes-dashboard
          image: kubernetesui/dashboard:v2.3.1
          imagePullPolicy: Always
          ports:
            - containerPort: 8443
              protocol: TCP
          args:
            - --auto-generate-certificates
            - --namespace=kubernetes-dashboard
            - --tls-key-file=tls.key
            - --tls-cert-file=tls.crt
            - --bind-address=0.0.0.0
            - --token-ttl=3600
            # Uncomment the following line to manually specify Kubernetes API server Host
            # If not specified, Dashboard will attempt to auto discover the API server and connect
            # to it. Uncomment only if the default does not work.
            # - --apiserver-host=http://my-address:port
          volumeMounts:
            - name: kubernetes-dashboard-certs
```
把其中证书改成实际的名称(通过--key 参数指定的不需要修改，通过--from参数指定的，可以通过上面查看secret命令输出结果查看)
```
            - --tls-key-file=tls.key
            - --tls-cert-file=tls.crt
```
##### 然后在deploy.yaml中找到
```
     volumes:
        - name: kubernetes-dashboard-certs
          secret:
            secretName: xxxxxx-secret
        - name: tmp-volume
          emptyDir: {}
      serviceAccountName: kubernetes-dashboard
```
修改secretName为自己自定义的的证书名称

##### 指定NodePort,用于访问（不需要可以注释）
```
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  #type: NodePort
  ports:
    - port: 443
      targetPort: 8443
   #   nodePort: 30001
  selector:
    k8s-app: kubernetes-dashboard

```
#### 部署安装
`kubectl apply -f deploy.yaml`



#### 访问

##### 创建管理员账号

1. 执行`vim admin-user.yaml`

2. 输入以下代码

   ```
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

   保存并退出

3. `kubectl apply -f apply admin-user.yaml`

##### 获取token

1. 查看kubectl安装的版本

   `kubectl version --short`

   **如果是v1.23及以下版本执行此命令获取**

   `kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')`

   输出样例

   ```
   Name:         admin-user-token-xpm5v
   Namespace:    kubernetes-dashboard
   Labels:       <none>
   Annotations:  kubernetes.io/service-account.name= admin-user
                 kubernetes.io/service-account.uid=0610610c-84e7-11e8-98de-00163e02d9ff
   
   Type:  kubernetes.io/service-account-token
   
   Data
   ====
   ca.crt:     1090 bytes
   namespace:  11 bytes
   token:      eyGciOiJSUzI1NiIsImtpZCI6Im5HeEp6UEphRmc3czV3dVBvc01PNllHLXNhckEzb094bmZGd1d6cTBUNncifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNjg0ODEzNzk4LCJpYXQiOjE2ODQ4MTAxOTgsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlLTg5ZWItNjBmMjhjMTgxOWI4In19LCJuYmYiOjE2ODQ4MTAxOTgsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlcm5ldGVzLWRhc2hib2FyZDphZG1pbi11c2VyIn0.O7PURPuf53UY69T73DTijk60QbGG58uHec4htxIVYMdazQveN9rEUj8HXGAsZGD3ZzmdD2FUU8kG1JAKB_9prD2SFlKrF9nQ_-xBDxaYrJKn3CZaeisX3Ly9NKayOvINjoxxzoLdOyX47uTzVg8eaQt5mDTbUyZaTjuRoQu_yXAh65RU45fA55yuN5BerfVJ1qADM-oAWcJ1afU4W1aRZxaT26O5dMzILKI2xh3zSNmmsndkfRfu6FyybpVF56x3aXjlRDUBao8HlYhUJKIEvC-HAusC9_QKyHOh1kdTb5RgA3aixyB7SXJopFfYKUcOy5xp6Nwh5IEUHHjb1MrCOg
   ```

   **如果是v1.24及以上版本**以上方式无法获取，创建账号也不会生成secret；需要通过kubectl api获取

   开启kubectl端口代理

   `kubectl proxy`

   执行命令访问api服务

   `curl -X POST -H "Content-Type:application/json" 'http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/serviceaccounts/admin-user/token' -d '{}'`

   输出样例

   ```
   {
     "kind": "TokenRequest",
     "apiVersion": "authentication.k8s.io/v1",
     "metadata": {
       "name": "admin-user",
       "namespace": "kubernetes-dashboard",
       "creationTimestamp": "2023-05-23T02:49:58Z",
       "managedFields": [
         {
           "manager": "curl",
           "operation": "Update",
           "apiVersion": "authentication.k8s.io/v1",
           "time": "2023-05-23T02:49:58Z",
           "fieldsType": "FieldsV1",
           "fieldsV1": {
             "f:spec": {
               "f:expirationSeconds": {}
             }
           },
           "subresource": "token"
         }
       ]
     },
     "spec": {
       "audiences": [
         "https://kubernetes.default.svc.cluster.local"
       ],
       "expirationSeconds": 3600,
       "boundObjectRef": null
     },
     "status": {
       "token": "eyJGciOiJSUzI1NiIsImtpZCI6Im5HeEp6UEphRmc3czV3dVBvc01PNllHLXNhckEzb094bmZGd1d6cTBUNncifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNjg0ODEzNzk4LCJpYXQiOjE2ODQ4MTAxOTgsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRljBmMjhjMTgxOWI4In19LCJuYmYiOjE2ODQ4MTAxOTgsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlcm5ldGVzLWRhc2hib2FyZDphZG1pbi11c2VyIn0.O7PURPuf53UY69T73DTijk60QbGG58uHec4htxIVYMdazQveN9rEUj8HXGAsZGD3ZzmdD2FUU8kG1JAKB_9prD2SFlKrF9nQ_-xBDxaYrJKn3CZaeisX3Ly9NKayOvINjoxxzoLdOyX47uTzVg8eaQt5mDTbUyZaTjuRoQu_yXAh65RU45fA55yuN5BerfVJ1qADM-oAWcJ1afU4W1aRZxaT26O5dMzILKI2xh3zSNmmsndkfRfu6FyybpVF56x3aXjlRDUBao8HhUJKIEvC-HAusC9_QKyHOh1kdTb5RgA3aixyB7SXJopFfYKUcOy5xp6Nwh5IEUHHjb1MrCOg",
       "expirationTimestamp": "2023-05-23T03:49:58Z"
     }
   }
   ```

   

##### kubectl proxy访问方式

只能是http访问

```
nohup kubectl proxy
```

如果想外部ip可以访问执行以下命令

```
nohup kubectl proxy --address='0.0.0.0' --accept-hosts='^*$' &
```

然后浏览器打开

`http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/`控制台,选择token登入



##### port-forward访问方式

`kubectl -n kubernetes-dashboard port-forward --address 0.0.0.0 service/kubernetes-dashboard 8002:443`

然后浏览器打开(可以用前面配置的域名访问)

`https://localhost:8002/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/`控制台,选择token登入

##### nodeport模式访问

*前提: 发布服务的时候配置的nodeport模式，并且设置了端口；上面步骤有提到*

然后浏览器打开

`https://localhost:31001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/`控制台,选择token登入

##### 通过ingress访问

*前提：已经安装了Ingress Controller*

1. `vim dashboard-ingress.yaml`

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /
    # 默认为 true，启用 TLS 时，http请求会 308 重定向到https
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
  name: kubernetes-dashboard-ingress
  namespace: kubernetes-dashboard
spec:
  tls:
    - secretName: xxxxxx-secret #设置成前面步骤创建的证书名称
      hosts:
        - xxxxxx.com #证书绑定的域名
  rules:
    - host: xxxxxx.com #证书绑定的域名
      http:
        paths:
          - pathType: Prefix
            path: /
            backend:
              service:
                name: kubernetes-dashboard
                port:
                  number: 443
```

2. `kubectl apply -f dashboard-ingress.yaml `

3. 执行`kubectl get ingress -n kubernetes-dashboard` 查看Host是否为设置的域名,address是否有分配外部ip

   正确输出样例

   ```
   NAME                           CLASS    HOSTS             ADDRESS         PORTS     AGE
   kubernetes-dashboard-ingress   <none>   k8s.alanpoi.com   10.108.249.35   80, 443   15h
   ```

   如果是如下

   ```
   NAME                           CLASS    HOSTS             ADDRESS   PORTS     AGE
   kubernetes-dashboard-ingress   <none>   k8s.alanpoi.com             80, 443   15h
   ```

   则是minikube创建的本地集群，默认无法分配外部ip

   可以开启隧道解决

   ```
   minikube tunnel
   ```

   然后重新执行ingress则可解决

   ```
   `kubectl apply -f dashboard-ingress.yaml `
   ```

   

3. （minikube创建的集群，只能本地访问，hosts文件设置域名解析）直接域名访问即可

   

#### 部署Ingress Controller

**安装请不要相信网上其他人的步骤操作，基本上你安装不成功**

1. (我已经整理上传到了我的git)下载部署文件 https://github.com/alan-et/k8s-components/blob/main/Ingress-controller/deploy.yaml

2. 部署

   ```
   kubectl apply -f deploy.yaml
   ```

3. 查看状态（两个webhook的job是一次性任务，所以为completed,ingress-nginx-controller状态必须为Running）

   执行`kubectl get pods -n ingress-nginx`

   输出样例

   ```
   NAME                                        READY   STATUS      RESTARTS   AGE
   ingress-nginx-admission-create-mlqng        0/1     Completed   0          18h
   ingress-nginx-admission-patch-fvj6h         0/1     Completed   0          18h
   ingress-nginx-controller-57948bb7bd-x8d9v   1/1     Running     0          18h
   ```

   执行`kubectl get services -n ingress-nginx`

   输出样例

   ```
   NAME                                 TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)                      AGE
   ingress-nginx-controller             LoadBalancer   10.108.249.35   10.108.249.35   80:31741/TCP,443:31871/TCP   18h
   ingress-nginx-controller-admission   ClusterIP      10.107.34.40    <none>          443/TCP                      18h
   ```

   如果EXTERNAL-IP一直为**pending**,则是minikube本地创建的集群，无法分配外部IP，通过`minikube tunnel`创建隧道解决

## 常见错误

#### 出现Exiting due to PROVIDER_DOCKER_VERSION_EXIT_1: "docker version --format -:" exit status 1: template: :1:44: executing "" at <.Server.Platform.Nam...>: can't evaluate field Platform in type *types.Version

1. 删除旧docker

```
sudo yum remove docker \
                docker-client \
                docker-client-latest \
                docker-common \
                docker-latest \
                docker-latest-logrotate \
                docker-logrotate \
                docker-selinux \
                docker-engine-selinux \
                docker-engine
```

2. 安装依赖包

   ```
   sudo yum install -y yum-utils \
     device-mapper-persistent-data \
     lvm2
   ```

3. 设置 Docker repository：

   ```
   sudo yum-config-manager \
       --add-repo \
       https://download.docker.com/linux/centos/docker-ce.repo
   ```

4. 安装最新版本的 Docker CE：

   ```
   sudo yum install docker-ce docker-ce-cli containerd.io
   ```

5. 启动docker

   ```
   sudo systemctl start docker
   ```

   

#### 出现execution phase certs/apiserver-kubelet-client: [certs] certificate apiserver-kubelet-client not signed by CA certificate ca: x509: certificate has expired or is not yet valid: current time 2023-05-18T03:26:25Z is after 2023-01-19T18:41:11Z

原因:  Kubernetes 集群的某个证书（具体来说，是 apiserver-kubelet-client）已经过期，这会阻止 Kubernetes 正常运行。证书是 Kubernetes 用来确保集群中的通信安全的重要组成部分。

对于 Minikube，删除现有的 Minikube 实例并重新创建一个新的，这将自动生成新的证书

`minikube delete`

`minikube start`

即可解决



#### 执行kubectl相关命令出现The connection to the server localhost:8080 was refused - did you specify the right host or port?

这个问题一般出现在minikube集群模式中，A用户执行的minikube start，在root或者其他用户执行kubectl命令则提示此错误

解决方式，把A用户的kube配置cp到此用户下

```
cp ~/.kube/config /root/.kube/config
```




