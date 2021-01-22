# 05.k8s集群部署prometheus-operator

## 组件版本

| 组件                | 版本        |
| ------------------- | ----------- |
| prometheus-operator | release-0.7 |

## 准备

```
cat /etc/hosts
10.140.0.4 master
10.140.0.5 node01
10.140.0.6 node02
10.140.0.7 node03
```

### 下载部署文件

github地址

```
https://github.com/coreos/kube-prometheus.git
```



![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611021157433-658589-00428daa0c1f.png)





[kube-prometheus-release-0.7.zip](https://www.yuque.com/attachments/yuque/0/2021/zip/1176682/1611021285134-fafc1db6-03e4-45e3-b90b-43f0131644ce.zip)



```
unzip kube-prometheus-release-0.7.zip
cd kube-prometheus-release-0.7/manifests/
```

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611021398734-335ffc63-f32b-4d3d-96fa-d62f639ce3c6.png)

### 分类

```
mkdir /root/prometheus
cp -r /root/kube-prometheus-release-0.7/manifests/* /root/prometheus/
cd /root/prometheus/
```

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611021652700-9b9a03d7-7c5b-4bb9-b7a1-81cb8ed3fc6c.png)

```
mkdir -p operator node-exporter alertmanager grafana kube-state-metrics prometheus serviceMonitor adapter
 
mv *-serviceMonitor* serviceMonitor/

mv grafana-* grafana/

mv kube-state-metrics-* kube-state-metrics/

mv alertmanager-* alertmanager/

mv node-exporter-* node-exporter/

mv prometheus-adapter* adapter/

mv prometheus-* prometheus/
```

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611021719100-03a82033-34ed-4078-9c31-0d459dc1dbf0.png)

## 修改prometheus配置文件

```
cd prometheus/prometheus/
```

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611022205974-6a134d31-4932-44f7-8bc6-b67965337849.png)



### 添加pv存储

这边用的是openebs自动提供的local-pv

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611022455344-71bc4ba3-7e41-4054-830c-49c66a559e8e.png)

```
vim prometheus-prometheus.yaml
  storage:
    volumeClaimTemplate:
      spec:
        storageClassName: openebs-hostpath
        resources:
          requests:
            storage: 20Gi
```

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611022573461-99934ec1-f565-468c-816c-a3e8e2b501f8.png)

### 添加etcd密钥

为了以后监控etcd，提前加入etcd的密钥

```
ls /etc/kubernetes/pki/etcd
```

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611022895496-ddbc4ff0-fd87-4a73-917a-a15ec1708a29.png)

先将需要使用的证书通过secret对象保存到集群中

```
kubectl create  ns monitoring
kubectl -n monitoring create secret generic etcd-certs \
--from-file=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
--from-file=/etc/kubernetes/pki/etcd/healthcheck-client.key \
--from-file=/etc/kubernetes/pki/etcd/ca.crt 
```

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611022972470-2817f7a4-de00-4ec6-871e-baf5fb1dd21d.png)

```
  secrets:
  - etcd-certs
```

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611023214815-3c9a1cfe-9690-490f-ac7b-73692523c097.png)

### 添加过期时长

通过`retention`参数进行修改，在`prometheus.spec`下填写

```
retention: 14d
```

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611022700801-c078de2e-18bb-47ba-9f22-9c8479e97cfb.png)

### 去除watchdog报警规则（可选）

watchdog只是告警的测试，用来查看告警是否成功，一般测试完成后需要删除

```
cd /root/prometheus/prometheus/
vim prometheus-rules.yaml
```

删掉watchdog告警

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611024216759-9952f703-d2bd-45b4-b32c-fb195454c866.png)

### 创建kube-controller-manager和kube-scheduler的svc

默认没有，需要自己建立

先查看master节点上的端口

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611195885130-faa1c156-abdf-42e1-a940-1c8af30e01e3.png)



```
cat > prom-svc.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  namespace: kube-system
  name: kube-controller-manager
  labels:
    k8s-app: kube-controller-manager
spec:
  type: ClusterIP
  clusterIP: None
  ports:
  - name: https-metrics
    port: 10257
    targetPort: 10257
    protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  namespace: kube-system
  name: kube-scheduler
  labels:
    k8s-app: kube-scheduler
spec:
  type: ClusterIP
  clusterIP: None
  ports:
  - name: https-metrics
    port: 10259
    targetPort: 10259
    protocol: TCP

---
apiVersion: v1
kind: Endpoints
metadata:
  labels:
    k8s-app: kube-controller-manager
  name: kube-controller-manager
  namespace: kube-system
subsets:
- addresses:
  - ip: 10.140.0.4
  ports:
  - name: https-metrics
    port: 10257
    protocol: TCP

---
apiVersion: v1
kind: Endpoints
metadata:
  labels:
    k8s-app: kube-scheduler
  name: kube-scheduler
  namespace: kube-system
subsets:
- addresses:
  - ip: 10.140.0.4
  ports:
  - name: https-metrics
    port: 10259
    protocol: TCP
EOF
```

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611024008505-b219f4c7-cf13-4e93-94ad-c24d0ecc5135.png)

### grafana添加pv存储

为了配置落盘，添加pv

```
cat > grafana-pvc.yaml << EOF
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: grafana-pvc
  namespace: monitoring
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi 
  storageClassName: openebs-hostpath
EOF
kubectl apply -f grafana-pvc.yaml
```

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611024439547-a81b900e-7cd8-456c-88a4-b6c18cdaa54f.png)

修改grafana配置

```
cd /root/prometheus/grafana/
vim grafana-deployment.yaml
```

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611024629658-3e935cc1-52ef-4a99-99be-f8ee9ee10405.png)

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611025100595-60c95bc3-625f-4586-a8b0-cced0bf431f1.png)



## 安装

```
cd /root/prometheus/
kubectl apply -f setup/
kubectl apply -f adapter/
kubectl apply -f alertmanager/
kubectl apply -f node-exporter/
kubectl apply -f kube-state-metrics/
kubectl apply -f grafana/
kubectl apply -f prometheus/
kubectl apply -f serviceMonitor/
```



## 查看部署情况

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611025394458-416c4f62-fb3e-4bf6-88a6-3504976b6800.png)



## 创建ingress

```
cat > prometheus-ingress.yaml << EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: prometheus.com
  namespace: monitoring
spec:
  rules:
  - host: grafana.tk8s.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: grafana
            port: 
              number: 3000
  - host: prometheus.tk8s.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: prometheus-k8s
            port: 
              number: 9090
  - host: alertmanager.tk8s.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: alertmanager-main
            port:
              number: 9093
EOF
 kubectl apply -f prometheus-ingress.yaml
kubectl get svc -n ingress-nginx 
```

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611026689450-fc83ea2c-b958-412d-870b-1edc72efba06.png)

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611028043559-fe1fa15b-1b2d-4024-809a-1290035344d2.png)

