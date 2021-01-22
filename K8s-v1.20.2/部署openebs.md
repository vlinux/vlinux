# 02. k8s集群部署openebs

这里需要用到pv存储，这里用的是本地存储local pv。

## 组件版本

| 组件    | 版本  |
| ------- | ----- |
| openebs | 2.4.0 |



## 安装openebs



每个节点创建local pv的存储目录

```
mkdir /data/local -p
```

下载yaml文件

[openebs-operator.yaml](https://www.yuque.com/attachments/yuque/0/2021/yaml/1176682/1610700919913-dc805ea8-a16f-496e-9549-bc9fbb7cdaa0.yaml)

```
wget https://openebs.github.io/charts/openebs-operator.yaml
 vim openebs-operator.yaml
```

修改localpv存储的目录

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1610700740797-3f312ff3-9f61-4acc-80a2-8bbd8d86609e.png)

```
kubectl apply -f openebs-operator.yaml
```



## 查看部署情况

```
kubectl get pods -n openebs -w
```

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1610701122208-cb37a4ab-6910-4d28-8454-ceb9607067a3.png)



