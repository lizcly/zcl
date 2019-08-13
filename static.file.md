# 基于Nginx-ingress Controller实现网站静态页面

## 一、 准备静态文件
将静态文件在集群中持久化保存，本文将以CFS云文件存储为例，说明静态文件在集群中持久化保存的步骤。

1. 使用CFS云文件存储创建NFS类型的PVC、PV

CFS云文件存储支持标准的NFS协议，支持标准的NFSv4.0和NFSv4.1协议，提供全托管的服务，您无需修改应用，通过标准的文件系统挂载步骤即可实现与Kubernetes集群的无缝集成。详情参考[使用nfs-client-provisioner动态创建PV](https://docs.jdcloud.com/cn/jcs-for-kubernetes/Create-PV-Dynamically)。

您可以[挂载CFS云文件服务](https://docs.jdcloud.com/cn/cloud-file-service/mount-file-system)到保存静态文件的服务器，将静态文件保存到PV在云文件服务中新建的子目录中；

**备注**：如果静态文件被保存到S3，您也可以将[对象存储Bucket作为共享存储](https://docs.jdcloud.com/cn/jcs-for-kubernetes/product-overview)挂载到对应的Pod；

## 二、将保存到CFS云文件存储的静态文件添加nginx server配置

1. 使用configmap处理静态部分的配置

* Yaml 文件说明如下
```
# cat static-server.conf

server {
    listen       80;
    server_name  nginx-ingress-test.jdcloud;
    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}

```

* 创建configmap

```
kubectl create configmap static-server --from-file static-server.conf 
```

2. 使用nginx镜像创建Deployment，提供静态页面服务

* Yaml文件说明如下

```
# cat test2-static-deployment.yaml 

apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: test-static
  labels:
    name: test-static
spec:
  replicas: 2
  template:
    metadata:
      labels:
       name: test-static
    spec:
      containers:
       - name: test-static
         image: nginx:latest
         volumeMounts:
          - mountPath: /usr/share/nginx/html/static-server
            name: static-data
          - mountPath: /etc/nginx/conf.d
            name: static-config
         ports:
          - containerPort: 80
      volumes:
         - name: static-data
           persistentVolumeClaim:
            claimName: auto-pv-with-nfs-client-provisioner                 #claimName请使用保存静态文件的PVC名称替换
         - name: static-config
           configMap:
             name: static-server                 #configmap Name请使用上一步创建的保存nginx.conf的配置文件的configmap名称替换
```

* 等待一段时间后，deployment处于运行状态后，创建ClusterIP类型的Service将deployment暴露出去，执行如下命令：

3. 创建创建ClusterIP类型的Service将deployment暴露出去

```
kubectl expose deployment/test-static --name=test-static --port=80 --target-port=80 --type=ClusterIP 
```

## 三、创建或编辑Ingres Resource，将静态服务的路径路由到提供静态页面服务的Service

1. 创建Ingress Resource的步骤参考[使用开源nginx-ingress controller定义ingress resource](https://docs.jdcloud.com/cn/jcs-for-kubernetes/Deploy-Ingress-Resource);

* 以HTTP类型的Ingress为例，添加静态页面服务的Yaml文件说明如下：

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: static-server-ingress
  annotations:
    metadata.annotations.kubernetes.io/ingress.class: "nginx"     #指定Ingress Resource创建时使用的Ingress Controller，本例使用上述创建的Nginx Controller
spec:
  rules:
  - host: nginx-ingress-test.jdcloud.com
    http:
      paths:
      - backend:
          serviceName: test-static
          servicePort: 80
        path: /static-server
```

2. 创建完成后即可在浏览器中验证动静态页面分离效果

* 获取Nginx-ingress Controller的外网IP，即Nginx-ingress Controller 关联的LoadBalancer类型Service的External IP，详情参考[Nginx-ingress controller部署](https://docs.jdcloud.com/cn/jcs-for-kubernetes/deploy-ingress-nginx-controller)；

* 在本地服务器的/etc/hosts中增加DNS配置：IP为上一步操作中查询到的LoadBalance类型service的external IP，域名为ingress resource rule中配置的虚拟主机名：nginx-ingress-test.jdcloud.com；

* 本例在CFS云文件服务中保存的静态文件为dlib库的说明文档；在浏览器中输入http://nginx-ingress-test.jdcloud.com/static-server/dlib-19.17/docs/index.html即可验证输出结果；

* 更多详情参考[使用开源nginx-ingress controller定义ingress resource](https://docs.jdcloud.com/cn/jcs-for-kubernetes/Deploy-Ingress-Resource)。
