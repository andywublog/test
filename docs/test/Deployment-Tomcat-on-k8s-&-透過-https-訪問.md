# Deployment Tomcat on k8s & 透過 https 訪問

## 建立 Tomcat
* Tomcat yaml example 
```
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: tomcat
  name: tomcat
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tomcat
  template:
    metadata:
      labels:
        app: tomcat
    spec:
      containers:
      - image: tomcat
        name: tomcat
        ports:
        - containerPort: 8080
        lifecycle:
            postStart:
              exec:
                command:
                  - /bin/sh
                  - -c
                  - |
                   cp -R  /usr/local/tomcat/webapps.dist/* /usr/local/tomcat/webapps/
```
* 建立 service
```
$ kubectl expose deploy tomcat --target-port=8080 --port=8080
```

> 登入位置
> http://localhost:8080

![](https://hackmd.io/_uploads/rJdy4eNb6.png)


## 透過 https 連到 tomcat
* 建立自簽憑證
```
$ git clone https://github.com/cooloo9871/openssl.git;cd openssl

# ip 為 cluster node ip
$ ./mk create tomcat.example.com 192.168.11.101

# 需要有 dns server 做名稱解析
$ host tomcat.example.com
tomcat.example.com has address 192.168.11.101
```
* 建立 secret
```
$ kubectl create secret tls tls-tomcat --cert=cert.pem --key=cert-key.pem
```

```
$ kubectl get secret tls-tomcat
NAME         TYPE                DATA   AGE
tls-tomcat   kubernetes.io/tls   2      55s
```

* 建立 ingress

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-tomcat-ingress
spec:
  ingressClassName: nginx
  tls:
  - hosts:
      - tomcat.example.com
    secretName: tls-tomcat
  rules:
  - host: tomcat.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: tomcat 
            port:
              number: 8080
```

```
$ kubectl get ing
NAME                 CLASS   HOSTS                ADDRESS          PORTS     AGE
tls-tomcat-ingress   nginx   tomcat.example.com   192.168.11.101   80, 443   74s
```

* 把 ca.pem import 到瀏覽器
![](https://i.imgur.com/JMoScUp.png)

* 透過火狐瀏覽器就可以走 https
![image](https://hackmd.io/_uploads/r1fKL76O6.png)


### on k3s
* 在 k3s ingress 是使用 traefik 
```
$ kubectl get ingressclass
NAME      CONTROLLER                      PARAMETERS   AGE
traefik   traefik.io/ingress-controller   <none>       7m25s
```
    
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-tomcat-ingress
spec:
  ingressClassName: traefik         # 更改此行
  tls:
  - hosts:
      - tomcat2.example.com
    secretName: tls-tomcat
  rules:
  - host: tomcat2.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: tomcat 
            port:
              number: 8080
```

```
$ kubectl get ing
NAME                 CLASS     HOSTS                 ADDRESS          PORTS     AGE
tls-tomcat-ingress   traefik   tomcat2.example.com   192.168.11.114   80, 443   7s
```
* 把 ca.pem 匯入瀏覽器後再次驗證
![image](https://hackmd.io/_uploads/r1kcNrR_6.png)
