## ingress
Ingress 就是定义路由规则：从集群外部→集群内部的HTTP和HTTPS的路由规则。
![](https://img2018.cnblogs.com/blog/1156961/201812/1156961-20181225142325638-2072946633.png)

Ingress yaml文件示例：
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  namespace: conn-dev
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: dev.xxx.com
    http:
      paths:
      - path: / # 该配置表示将dev.xxx.com的请求转发到serviceName为nginx，servicePort为80的服务上
        pathType: Prefix
        backend:
          service:
            name: nginx
            port:
              number: 80
```
### Ingress资源类型
1. 单Service资源型Ingress
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: my-ingress
spec:
  backend:
    serviceName: my-svc
    servicePort: 80
```
2. 基于url路径进行流量分发
```

apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-url-demo
  annotations:
     nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: myapp.syztoo.com
    http:
      paths:
      - path: /v1
        backend:
          serviceName: myappv1-svc
          servicePort: 80
      - path: /v2
        backend:
          serviceName: myappv2-svc
          servicePort: 80
 
---
apiVersion: v1
kind: Service
metadata:
  name: myappv1-svc
  namespace: default
spec:
  selector:
    app: myappv1
    release: canary
  type: ClusterIP
  ports: 
  - port: 80
    targetPort: 80
 
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myappv1-depoly
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myappv1
      release: canary
  template:
    metadata:
      labels: 
        app: myappv1
        release: canary
    spec:
      containers:
      - name: myappv1
        image: ikubernetes/myapp:v1
        ports:
        - name: http
          containerPort: 80
 
---
apiVersion: v1
kind: Service
metadata:
  name: myappv2-svc
  namespace: default
spec:
  selector:
    app: myappv2
    release: canary
  type: ClusterIP
  ports: 
  - port: 80
    targetPort: 80
 
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myappv2-deploy
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myappv2
      release: canary
  template:
    metadata:
      labels: 
        app: myappv2
        release: canary
    spec:
      containers:
      - name: myappv2
        image: ikubernetes/myapp:v2
        ports:
        - name: http
          containerPort: 80
```
3. 基于主机名称的虚拟主机
```

apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test
spec:
  rules:
  - host: api.ik8s.io
    http: 
      paths:
      - backend:
          serviceName: api
          servicePort: 80
  - host: wap.ik8s.io
    http: 
      paths:
      - backend:
          serviceName: wap
          servicePort: 80
```
4. TLS类型的Ingress资源
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: no-rules-map
spec: 
  tls:
  - secretName: ikubernetesSecret
  backend:
    serviceName: homesite
    servicePort: 80
```

``
Ingress是k8s的标准资源类型之一，它其实就是一组基于DNS名称或URL路径把请求转发至指定的service资源的规则，用于将集群外部的请求流量转发至集群内部完成服务发布。然而，Ingress资源自身并不能进行流量穿透，它仅是规则的集合，这些规则要想真正发挥作用还需要其他功能的辅助，如监听某套接字，然后根据这些规则的匹配机制路由请求流量。这种能够为Ingress资源监听套接字并转发流量的组件称为Ingress控制器。
``






## ingress controller
``
If Kubernetes Ingress is the API object that provides routing rules to manage external access to services, Ingress Controller is the actual implementation of the Ingress API. The Ingress Controller is usually a load balancer for routing external traffic to your Kubernetes cluster and is responsible for L4-L7 Network Services.
``
![](https://img-blog.csdnimg.cn/20190910113006302.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MjU5NTAxMg==,size_16,color_FFFFFF,t_70)


---


参考：

kubernetes之ingress和Ingress Controller https://blog.csdn.net/weixin_42595012/article/details/100692084

官方文档 https://kubernetes.io/docs/concepts/services-networking/ingress/
