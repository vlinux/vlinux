# 03.k8s集群部署Metrics-Server

这里需要用到metrics，需要部署metrics-server

## 组件版本

| 组件           | 版本  |
| -------------- | ----- |
| metrics-server | 0.4.1 |

官方GitHub

```
https://github.com/kubernetes-sigs/metrics-server
```



## 部署Metrics-Server

[components.yaml](https://www.yuque.com/attachments/yuque/0/2021/yaml/1176682/1610873566066-62cd281c-b319-4840-a8ee-aaea739cddaa.yaml)

```
kubectl apply -f components.yaml 
```



## 查看部署情况

```
kubectl top pods -A
```

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1610873644797-f41b1383-460a-4a2a-a5e3-a61ccc0a35f4.png)



```
kubectl top nodes
```

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1610873683850-6976fdd8-9630-42fd-be16-a9d2d16996f2.png)

