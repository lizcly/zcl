# 使用Nginx-ingress实现蓝绿发布

本文基于Kubernetes社区的开源ingress-controller，支持基于服务权重的流量切分。

**应用场景**：假设当前线上环境我们已经有一套服务 app-old 对外提供 7 层服务，此时我们修复了一些问题，需要灰度发布上线一个新的版本 app-new，但是我们又不希望简单直接地将所有客户端流量切换到新版本 app-new 中，而是希望仅仅切换 20% 的流量到新版本 app-new 中，待运行一段时间稳定，将所有的流量切换到 app-new 服务中后，再平滑地下线掉 app-old 服务。

针对以上多种不同的应用发布需求，K8S Ingress Controller 支持了多种流量切分方式：

* 基于 Request Header 的流量切分，适用于灰度发布以及 AB 测试场景

* 基于 Cookie 的流量切分，适用于灰度发布以及 AB 测试场景

* 基于 Query Param 的流量切分，适用于灰度发布以及 AB 测试场景

* 基于服务权重的流量切分，适用于蓝绿发布场景

以下测试基于服务权重的流量切分，也可以将nginx.ingress.kubernetes.io/canary-weight: "30"改为基于 header 的流量切分。

## 一、部署Kubernetes ingress-nginx Controller

1. ingress-nginx Controller相关的Yaml部署文件下载及说明如下：

```
#下载yaml文件
wget https://kubernetes.s3.cn-north-1.jdcloud-oss.com/kubernetes-ingress/ingress-nginx.yml

#Yaml文件说明
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---

kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
kind: ConfigMap
apiVersion: v1
metadata:
  name: tcp-services
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
kind: ConfigMap
apiVersion: v1
metadata:
  name: udp-services
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress-serviceaccount
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: nginx-ingress-clusterrole
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - endpoints
      - nodes
      - pods
      - secrets
    verbs:
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - services
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - events
    verbs:
      - create
      - patch
  - apiGroups:
      - "extensions"
      - "networking.k8s.io"
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - "extensions"
      - "networking.k8s.io"
    resources:
      - ingresses/status
    verbs:
      - update

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: Role
metadata:
  name: nginx-ingress-role
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - pods
      - secrets
      - namespaces
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - configmaps
    resourceNames:
      # Defaults to "<election-id>-<ingress-class>"
      # Here: "<ingress-controller-leader>-<nginx>"
      # This has to be adapted if you change either parameter
      # when launching the nginx-ingress-controller.
      - "ingress-controller-leader-nginx"
    verbs:
      - get
      - update
  - apiGroups:
      - ""
    resources:
      - configmaps
    verbs:
      - create
  - apiGroups:
      - ""
    resources:
      - endpoints
    verbs:
      - get

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: nginx-ingress-role-nisa-binding
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: nginx-ingress-role
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: nginx-ingress-clusterrole-nisa-binding
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: nginx-ingress-clusterrole
subjects:
  - kind: ServiceAccount
    name: nginx-ingress-serviceaccount
    namespace: ingress-nginx

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/part-of: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
      annotations:
        prometheus.io/port: "10254"
        prometheus.io/scrape: "true"
    spec:
      serviceAccountName: nginx-ingress-serviceaccount
      containers:
        - name: nginx-ingress-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.25.0
          args:
            - /nginx-ingress-controller
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
            - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
            - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
            - --publish-service=$(POD_NAMESPACE)/ingress-nginx
            - --annotations-prefix=nginx.ingress.kubernetes.io
          securityContext:
            allowPrivilegeEscalation: true
            capabilities:
              drop:
                - ALL
              add:
                - NET_BIND_SERVICE
            # www-data -> 33
            runAsUser: 33
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - name: http
              containerPort: 80
            - name: https
              containerPort: 443
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 10
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            periodSeconds: 10
            successThreshold: 1
            timeoutSeconds: 10

---

```
2. 部署ingress-nginx Controller，执行如下命令：`kubectl create -f ingress-nginx.yml`

3. 等待一段实际后，ingress-nginx deployment处于运行状态后，创建Loadbalancer类型的Service将ingress-nginx deployment暴露到公网，执行如下命令：

```
# 创建创建Loadbalancer类型的Service将ingress-nginx deployment暴露到公网
kubectl expose deployment/nginx-ingress-controller --name=nginx-ingress-service --port=80 --target-port=80 --type=LoadBalancer -n ingress-nginx

# 查看Loadbalancer类型的Service运行状态：
kubectl get service -n ingress-nginx
NAME                    TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)        AGE
nginx-ingress-service   LoadBalancer   10.0.61.209   116.ab.cd.104   80:31691/TCP   6m1s

# 查看nginx-ingress-service关联的endpoints：
kubectl get endpoints -n ingress-nginx
NAME                    ENDPOINTS      AGE
nginx-ingress-service   10.0.0.12:80   6m11s
```

## 二、准备老版本程序

1. Yaml文件下载及说明如下：

```
#下载app-old.yaml
wget https://kubernetes.s3.cn-north-1.jdcloud-oss.com/kubernetes-ingress/app-old.yml

#Yaml文件说明

apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: app-old
spec:
  replicas: 2
  selector:
    matchLabels:
      run: app-old
  template:
    metadata:
      labels:
        run: app-old
    spec:
      containers:
      - image: zouhl/app:v2.1
        imagePullPolicy: Always
        name: app-old
        ports:
        - containerPort: 80
          protocol: TCP
      restartPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: app-old
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    run: app-old
  sessionAffinity: None
  type: ClusterIP

# 下载app-v1.yaml
wget https://kubernetes.s3.cn-north-1.jdcloud-oss.com/kubernetes-ingress/app-v1.yml

#Yaml文件说明
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: my-app
  labels:
    app: my-app
  annotations:
    kubernetes.io/ingress.class: nginx
  namespace: default
spec:
  rules:
    - host: test.192.168.2.20.xip.io
      http:
        paths:
          - backend:
              serviceName: app-old
              servicePort: 80
            path: /
```

2. 在 Kubernetes集群中创建老版本的应用和对应的ingress resource：

```
kubectl create -f app-old.yaml
kubectl create -f app-v1.yaml
```

## 三、装备新版本程序

1. yaml文件下载及说明如下；

```
#下载新版本app-new.yaml
wget https://kubernetes.s3.cn-north-1.jdcloud-oss.com/kubernetes-ingress/app-new.yml

#Yaml文件说明：
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: app-new
spec:
  replicas: 2
  selector:
    matchLabels:
      run: app-new
  template:
    metadata:
      labels:
        run: app-new
    spec:
      containers:
      - image: zouhl/app:v2.2
        imagePullPolicy: Always
        name: app-new
        ports:
        - containerPort: 80
          protocol: TCP
      restartPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: app-new
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    run: app-new
  sessionAffinity: None
  type: ClusterIP

#下载新版本的app-v2-canary.yaml
wget https://kubernetes.s3.cn-north-1.jdcloud-oss.com/kubernetes-ingress/app-v2-canary.yaml

#Yaml文件说明如下：
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: my-app-canary
  labels:
    app: my-app
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "30"
  namespace: default
spec:
  rules:
    - host: test.192.168.2.20.xip.io
      http:
        paths:
          - backend:
              serviceName: app-new
              servicePort: 80
            path: /
```
2. 在 Kubernetes集群中创建老版本的应用和对应的ingress resource：

```
kubectl create -f app-new.yaml
kubectl create -f app-v2-canary.yaml
```

## 四、验证基于服务权重的流量切分

1. 验证ingress的状态

```
kubectl get ingresses.extensions -n ingress-nginx
NAME            HOSTS                      ADDRESS   PORTS   AGE
my-app          test.192.168.2.20.xip.io             80      12m
my-app-canary   test.192.168.2.20.xip.io             80      12m
```
2. 在本地/etc/hosts文件中添加dns配置，并测试发布效果：

```
#/etc/hosts文件中增加如下内容
116.ab.cd.104  test.192.168.2.20.xip.io          #其中116.ab.cd.104是ingress-nginx controller关联的LoadBalancer 类型Service的external IP

#验证发布效果：
# while sleep 0.5; do curl "test.192.168.2.20.xip.io";echo; done
{"v1 hostname":"app-old-945c649fb-p6m88"}
{"v1 hostname":"app-old-945c649fb-mrwq6"}
{"v1 hostname":"app-old-945c649fb-p6m88"}
{"v2.2 hostname":"app-new-5fb7b7b7d5-tcj9p"}
{"v1 hostname":"app-old-945c649fb-mrwq6"}
{"v1 hostname":"app-old-945c649fb-p6m88"}
{"v1 hostname":"app-old-945c649fb-mrwq6"}
{"v1 hostname":"app-old-945c649fb-p6m88"}
{"v1 hostname":"app-old-945c649fb-mrwq6"}
{"v1 hostname":"app-old-945c649fb-p6m88"}
{"v2.2 hostname":"app-new-5fb7b7b7d5-55w6t"}
{"v2.2 hostname":"app-new-5fb7b7b7d5-tcj9p"}
{"v2.2 hostname":"app-new-5fb7b7b7d5-55w6t"}
{"v2.2 hostname":"app-new-5fb7b7b7d5-tcj9p"}
{"v1 hostname":"app-old-945c649fb-mrwq6"}
{"v2.2 hostname":"app-new-5fb7b7b7d5-55w6t"}
{"v2.2 hostname":"app-new-5fb7b7b7d5-tcj9p"}
```
## 五、删除服务权重相关的配置

1. 删除配置服务权重ingress resource，重新部署未配置服务权重的ingress resource，yaml文件下载及说明如下：

```
# 删除配置服务权重的新版本ingress resource
kubectl delete -f app-v2-canary.yaml
kubectl delete -f app-v1.yaml

#下载未配置服务权重的ingress resource Yaml
wget https://kubernetes.s3.cn-north-1.jdcloud-oss.com/kubernetes-ingress/app-v2.yaml

#部署未配置服务权重的ingress resource
kubectl apply -f app-v2.yaml 

#检查 ingress resouce状态
kubectl get ingresses.extensions -n ingress-nginx
NAME        HOSTS                      ADDRESS   PORTS   AGE
my-app      test.192.168.2.20.xip.io             80      20m
my-app-v2   test.192.168.2.20.xip.io             80      23s
```
2. 重新测试发布效果

```
while sleep 0.5; do curl "test.192.168.2.20.xip.io";echo; done
{"v2.2 hostname":"app-new-5fb7b7b7d5-tcj9p"}
{"v2.2 hostname":"app-new-5fb7b7b7d5-55w6t"}
```