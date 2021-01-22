# 07.k8s集群部署Kubeord

## 组件版本

| 组件    | 版本 |
| ------- | ---- |
| Kubeord | v3   |



## 安装

官网地址

```
https://www.kuboard.cn/install/install-dashboard.html
```

Kuboard 是 Kubernetes 的一款图形化管理界面

### 下载yaml

```
curl -o kuboard-v3.yaml https://addons.kuboard.cn/kuboard/kuboard-v3.yaml
```

### 修改yaml

```
 vim kuboard-v3.yaml 
```

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611231509683-175bca74-2568-4258-b87f-96a72a85fc87.png)

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611231547908-6a5d9843-1663-422d-a2b0-aca4fbbc3a17.png)



```
kubectl apply -f kuboard-v3.yaml
```

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611230234490-e0455207-970b-4466-a357-3c9bb6352827.png)



## 访问

### 登陆

node节点:30080

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611230474723-5646d604-0b21-4ff1-93c6-833b563b7d7b.png)

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611230524574-2c694ecb-6983-4411-9bbe-c7f8a3e04b45.png)

### 添加集群

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611230670050-110e4bd4-7f06-49d7-a29a-0e2199347887.png)

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611231452422-c1420296-a3d8-49e9-9d93-09d2cbcb6f5e.png)

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611231608229-991ffa65-fca9-4d15-b8c9-9881335eb761.png)

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611231663802-d8fc35da-91ce-4f2b-975a-a62b91884b25.png)

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611231686770-4c2efe9b-0d85-4acf-af54-930db39ee89d.png)