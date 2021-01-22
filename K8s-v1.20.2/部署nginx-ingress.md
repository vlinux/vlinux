# 04.k8s集群部署nginx-ingress

## 组件版本

| 组件          | 版本   |
| ------------- | ------ |
| nginx ingress | 0.43.0 |



## 安装

nginx ingress官网

```
https://kubernetes.github.io/ingress-nginx/
```

### 一键部署

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.43.0/deploy/static/provider/baremetal/deploy.yaml
```

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611026223844-62a0b20c-cc5b-42ea-a01a-c8b1ad79f063.png)



## 查看部署情况

```
kubectl get pod -n ingress-nginx
kubectl get svc -n ingress-nginx
```

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611026283067-471aad1b-9ec1-4ac6-9dae-d94d086dec2b.png)