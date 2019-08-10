# 基于Nginx-ingress Controller实现网站静态页面

## 一、 准备静态文件
将静态文件在集群中持久化保存，本文将以CFS云文件存储为例，说明静态文件在集群中持久化保存的步骤。

1. 使用CFS云文件存储创建NFS类型的PVC、PV

CFS云文件存储支持标准的NFS协议，支持标准的NFSv4.0和NFSv4.1协议，提供全托管的服务，您无需修改应用，通过标准的文件系统挂载步骤即可实现与Kubernetes集群的无缝集成。详情参考[使用nfs-client-provisioner动态创建PV](https://docs.jdcloud.com/cn/jcs-for-kubernetes/Create-PV-Dynamically)。

2. 将静态文件保存到云文件存储

```
kind: Pod
apiVersion: v1
metadata:
  name: verify-pv-cfs
spec:
  containers:
  - name: c1
    image: busybox
    imagePullPolicy: IfNotPresent
    command:
    - /bin/sh
    args:
    - -c
    - 'wget http://dlib.net/files/dlib-19.17.tar.bz2; tar jxvf dlib-19.17.tar.bz2'		
    volumeMounts:
    - mountPath: "/mnt/cfs"
      name: cfs-pv001
  volumes:
  - name: cfs-pv001
    persistentVolumeClaim:
      claimName: auto-pv-with-nfs-client-provisioner          #与云文件存储建立绑定关系的PVC 名称
```

2. 使用configmap处理静态部分的路由

* Yaml 文件说明如下
```
# cat test2-static-data-pv-pvc.yaml
user nginx;
worker_processes auto;
error_log /usr/share/nginx/html/nginx-error.log;
pid /run/nginx.pid;

# Load dynamic modules. See /usr/share/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 102400;
    use epoll;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    server_tokens     off;
    access_log        /usr/share/nginx/html/nginx-default-access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/conf/extra/*.conf;

server {
        listen       80 default_server;
        index  index.html index.htm;
        access_log /usr/share/nginx/html/logs/test2-static-access.log main;
        location / {
        root /usr/share/nginx/html/;
        index  index.html index.htm;
        }
        
   }

```
* 创建configmap

```
kubectl create configmap test2-static-etc --from-file nginx.conf 
```

3. 使用nginx镜像创建Deployment，提供静态页面服务

* Yaml文件说明如下

```
# cat test2-static-deployment.yaml 
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: test2-static
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: test2-static
  labels:
    name: test2-static
spec:
  replicas: 2
  template:
    metadata:
      labels:
       name: test2-static
    spec:
      containers:
       - name: test2-static
         image: nginx:latest
         volumeMounts:
          - mountPath: /usr/share/nginx/html
            name: test2-static-data
          - mountPath: /etc/nginx/nginx.conf
            subPath: nginx.conf
            name: test2-static-etc
         ports:
          - containerPort: 80
      volumes:
         - name: test2-static-data
           persistentVolumeClaim:
            claimName: test2-static                 #claimName请使用保存静态文件的PVC名称替换
         - name: test2-static-etc
           configMap:
             name: test2-static-etc                 #configmap Name请使用上一步创建的保存nginx.conf的配置文件的configmap名称替换
             items:
             - key: nginx.conf
               path: nginx.conf
```

* 等待一段时间后，deployment处于运行状态后，创建ClusterIP类型的Service将deployment暴露出去，执行如下命令：

```
# 创建创建ClusterIP类型的Service将deployment暴露出去
kubectl expose deployment/test2-static --name=test2-static --port=80 --target-port=80 --type=ClusterIP -n ingress-nginx

```
三、创建或编辑Ingres Resource，将静态服务的路径路由到提供静态页面服务的Service

1. 创建Ingress Resource的步骤参考[使用开源nginx-ingress controller定义ingress resource])(https://docs.jdcloud.com/cn/jcs-for-kubernetes/Deploy-Ingress-Resource);

* 以HTTP类型的Ingress为例，添加静态页面服务的Yaml文件说明如下：

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: k8s-app-monitor-agent-ingress
  annotations:
    metadata.annotations.kubernetes.io/ingress.class: "nginx"     #指定Ingress Resource创建时使用的Ingress Controller，本例使用上述创建的Nginx Controller
spec:
  rules:
  - host: nginx-ingress-test.jdcloud
    http:
      paths:
      - backend:
          serviceName: nginx-demo-svc
          servicePort: 80
        path: /nginx-demo
      - backend:
          serviceName: test2-static
          servicePort: 80
        path: /static-server
```

2. 创建完成后即可在浏览器中验证动静态页面分离效果，详情参考[使用开源nginx-ingress controller定义ingress resource])(https://docs.jdcloud.com/cn/jcs-for-kubernetes/Deploy-Ingress-Resource)。
