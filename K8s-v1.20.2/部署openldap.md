# 06.k8s集群部署openldap

为了方便管理和集成jenkins，k8s、harbor、jenkins均使用openLDAP统一认证。

## 组件版本

| 组件     | 版本                  |
| -------- | --------------------- |
| openldap | osixia/openldap:1.2.2 |
| lam      | 7.4                   |



## 安装ldap

### 创建pvc

```
cat > ldap-pvc.yaml << EOF
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: ldap-data-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi 
  storageClassName: openebs-hostpath
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: ldap-config-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi 
  storageClassName: openebs-hostpath
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: ldap-certs-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi 
  storageClassName: openebs-hostpath
EOF
 kubectl apply -f ldap-pvc.yaml
```

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611209379340-d52a8aec-86be-4965-a7ed-2e151575f297.png)

### 创建deployment

```
cat > ladp-deployment.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ldap
  labels:
    app: ldap
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ldap
  template:
    metadata:
      labels:
        app: ldap
    spec:
      containers:
        - name: ldap
          image: osixia/openldap:1.2.2
          volumeMounts:
            - name: ldap-data
              mountPath: /var/lib/ldap
            - name: ldap-config
              mountPath: /etc/ldap/slapd.d
            - name: ldap-certs
              mountPath: /container/service/slapd/assets/certs
          ports:
            - containerPort: 389
              name: openldap
          env:
            - name: LDAP_LOG_LEVEL
              value: "256"
            - name: LDAP_ORGANISATION
              value: "tk8s"
            - name: LDAP_DOMAIN
              value: "tk8s.com"
            - name: LDAP_ADMIN_PASSWORD
              value: "admin"
            - name: LDAP_CONFIG_PASSWORD
              value: "config"
            - name: LDAP_READONLY_USER
              value: "false"
            - name: LDAP_READONLY_USER_USERNAME
              value: "readonly"
            - name: LDAP_READONLY_USER_PASSWORD
              value: "readonly"
            - name: LDAP_RFC2307BIS_SCHEMA
              value: "false"
            - name: LDAP_BACKEND
              value: "mdb"
            - name: LDAP_TLS
              value: "true"
            - name: LDAP_TLS_CRT_FILENAME
              value: "ldap.crt"
            - name: LDAP_TLS_KEY_FILENAME
              value: "ldap.key"
            - name: LDAP_TLS_CA_CRT_FILENAME
              value: "ca.crt"
            - name: LDAP_TLS_ENFORCE
              value: "false"
            - name: LDAP_TLS_CIPHER_SUITE
              value: "SECURE256:+SECURE128:-VERS-TLS-ALL:+VERS-TLS1.2:-RSA:-DHE-DSS:-CAMELLIA-128-CBC:-CAMELLIA-256-CBC"
            - name: LDAP_TLS_VERIFY_CLIENT
              value: "demand"
            - name: LDAP_REPLICATION
              value: "false"
            - name: LDAP_REPLICATION_CONFIG_SYNCPROV
              value: "binddn=\"cn=admin,cn=config\" bindmethod=simple credentials=$LDAP_CONFIG_PASSWORD searchbase=\"cn=config\" type=refreshAndPersist retry=\"60 +\" timeout=1 starttls=critical"
            - name: LDAP_REPLICATION_DB_SYNCPROV
              value: "binddn=\"cn=admin,$LDAP_BASE_DN\" bindmethod=simple credentials=$LDAP_ADMIN_PASSWORD searchbase=\"$LDAP_BASE_DN\" type=refreshAndPersist interval=00:00:00:10 retry=\"60 +\" timeout=1 starttls=critical"
            - name: LDAP_REPLICATION_HOSTS
              value: "#PYTHON2BASH:['ldap://ldap-one-service', 'ldap://ldap-two-service']"
            - name: KEEP_EXISTING_CONFIG
              value: "false"
            - name: LDAP_REMOVE_CONFIG_AFTER_SETUP
              value: "true"
            - name: LDAP_SSL_HELPER_PREFIX
              value: "ldap"
      volumes:
        - name: ldap-data
          persistentVolumeClaim:
            claimName: ldap-data-pvc
        - name: ldap-config
          persistentVolumeClaim:
            claimName: ldap-config-pvc
        - name: ldap-certs
          persistentVolumeClaim:
            claimName: ldap-certs-pvc
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: ldap
  name: ldap-service
spec:
  ports:
    - port: 389
  selector:
    app: ldap
EOF
```

需要修改的地方有3点，其他不需要改动

- `LDAP_ORGANISATION` 组织名称。默认为Example Inc.
- `LDAP_DOMAIN` Ldap域。默认为example.org
- `LDAP_ADMIN_PASSWORD` Ldap管理员密码。默认为admin

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611199749535-cc73a6f3-8245-454e-9da8-7d39f08d4a0f.png)

```
kubectl apply -f ladp-deployment.yaml
```

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611209773637-1fba4af7-44ff-4f03-bfa3-2c86248d6064.png)

### 测试

```
kubectl exec -it  ldap-7fc5bc794d-5pbgx -- bash
ldapsearch -x -H ldap://localhost -b dc=tk8s,dc=com -D "cn=admin,dc=tk8s,dc=com" -w admin
```

若成功得到类似以下文本的返回值，代表服务启动成功。

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611210058880-e5cd7e8f-3896-408a-915e-4758409c9b01.png)



## 安装ldap-lam

我这边dn写死在文件中，用的是`dc=tk8s,dc=com`

需要更改dn，先下载镜像，更改`/var/www/html/config/lam.conf` ,然后重新`docker commit`

### 创建deployment

```
cat > ladp-lam-deployment.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ldap-lam
  labels:
    app: ldap-lam
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ldap-lam
  template:
    metadata:
      labels:
        app: ldap-lam
    spec:
      containers:
        - name: ldap
          image: tanmgweiwow/lam:v1
          ports:
            - containerPort: 80
              name: http
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: ldap-lam
  name: ldap-lam-service
spec:
  ports:
    - port: 80
      targetPort: 80
  selector:
    app: ldap-lam
EOF
kubectl apply -f ladp-lam-deployment.yaml
```

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611218828683-4a68e5d9-58b1-42fd-8916-e8576eb74651.png)

### 创建ingress

```
cat > lam-ingress.yaml << EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: lam
spec:
  rules:
  - host: lam.tk8s.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ldap-lam-service
            port: 
              number: 80
EOF
```

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611219059195-200609d6-bbf1-41b0-ae65-cb2ad40190cb.png)

## 访问

### 登陆

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611220451537-a1c63b6c-dbbe-44ad-ac05-947bd3eb581d.png)

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611220535797-da798416-588e-4314-aad2-8dbe43f4f2a3.png)

### 添加组

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611220642881-47e8c951-d27f-475d-89d9-b50c4ad6615e.png)

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611220735340-041c9b59-3f63-493c-a54e-8034a5c23805.png)

### 添加用户

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611220787076-608452c1-506c-4f42-a961-1d04ea0e52cf.png)

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611220890888-70463be7-524e-4389-a672-60c3ed9219ae.png)

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611230410693-bcd2ac6b-bf5e-4530-936b-464cd485445f.png)

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611220949256-61ceb369-0c42-4add-9ab5-50d8e56370a0.png)

### 查看用户是否存在

```
kubectl exec -it  ldap-7fc5bc794d-5pbgx -- bash
ldapsearch -x -H ldap://localhost -b dc=tk8s,dc=com -D "cn=admin,dc=tk8s,dc=com" -w admin
```

![image.png](https://gitee.com/xoxoyun/img/raw/master/image/1611223289193-5b3e4395-0420-4462-bc3c-197e0a1efdaa.png)

