## 官方文件安装（自动创建证书）
跳过以下所有步骤，直接执行
`kubectl apply -f deploy-2.3.1.yaml`


## 准备证书
#### 创建k8s secret
```
kubectl create secret tls 自定义命名 --key 证书key路径/证书名.key --cert 证书key路径/证书名.pem或crt -n kubernetes-dashboard
```
#### 查看secret
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
#### 在deploy.yaml中找到
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
#### 然后在deploy.yaml中找到
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

### 指定NodePort,用于访问（不需要可以注释）
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
## 部署安装
`kubectl apply -f deploy.yaml`
