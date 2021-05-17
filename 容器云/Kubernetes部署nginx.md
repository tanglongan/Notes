在Kubernetes集群中部署一个Nginx

```shell
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get pods,svc
```

在Kubernetes集群中部署一个Tomcat

```shell
kubectl create deployment nginx --image=tomcat
kubectl expose deployment nginx --port=8080 --type=NodePort
kubectl get pods,svc

```

Kubernetes部署容器化应用的步骤：

1. 制作镜像
2. 通过控制器管理Pod（实际上就是通过镜像启动而创建一个容器，容器在Pod里面）
3. 暴露应用，以便于外接可以访问

