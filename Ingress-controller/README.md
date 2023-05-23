### 部署

 ```
 kubectl apply -f deploy.yaml
 ```
### 查看状态（两个webhook的job是一次性任务，所以为completed; ingress-nginx-controller状态必须为Running）

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
