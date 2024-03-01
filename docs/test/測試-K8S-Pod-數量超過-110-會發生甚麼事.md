# 測試 K8S Pod 數量超過 110 會發生甚麼事?



## Preface

<div class="indent-title-1">

本篇文章會介紹，在 Kubernetes 中，如果單一台節點上 Pod 的數量超過 110 個會發生甚麼事?

</div>

## 測試環境

### 1M2W 的架構

<div class="indent-title-1">

```
$ kubectl get nodes -o wide
```

螢幕輸出 : 

```
NAME        STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE         KERNEL-VERSION   CONTAINER-RUNTIME
antony-m1   Ready    control-plane   46m   v1.29.0   172.20.0.31   <none>        Talos (v1.6.1)   6.1.69-talos     containerd://1.7.11
antony-w1   Ready    worker          46m   v1.29.0   172.20.0.32   <none>        Talos (v1.6.1)   6.1.69-talos     containerd://1.7.11
antony-w2   Ready    worker          46m   v1.29.0   172.20.0.33   <none>        Talos (v1.6.1)   6.1.69-talos     containerd://1.7.11
```

</div>

### 單台節點允許執行 Pod 的數量

<div class="indent-title-1">

```
$ kubectl get nodes -o yaml | grep -A 11 "allocatable:$"
```

螢幕輸出 : 

```
    allocatable:
      cpu: 9950m
      ephemeral-storage: "95171693615"
      hugepages-2Mi: "0"
      memory: 32550712Ki
      pods: "110"
    capacity:
      cpu: "10"
      ephemeral-storage: 101132Mi
      hugepages-2Mi: "0"
      memory: 32849720Ki
      pods: "110"
--
    allocatable:
      cpu: 9950m
      ephemeral-storage: "95171693615"
      hugepages-2Mi: "0"
      memory: 32550704Ki
      pods: "110"
    capacity:
      cpu: "10"
      ephemeral-storage: 101132Mi
      hugepages-2Mi: "0"
      memory: 32849712Ki
      pods: "110"
--
    allocatable:
      cpu: 9950m
      ephemeral-storage: "95171693615"
      hugepages-2Mi: "0"
      memory: 32550712Ki
      pods: "110"
    capacity:
      cpu: "10"
      ephemeral-storage: 101132Mi
      hugepages-2Mi: "0"
      memory: 32849720Ki
      pods: "110"
```

> 三台 Node 最多都只能 Run 110 個 Pods

</div>

## 1. 建立測試 Deployment Yaml 檔

<div class="indent-title-1">

```
$ cat <<EOF > deployment-w2.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: alp
  labels:
    node: w2
spec:
  replicas: 3
  selector:
    matchLabels:
      node: w2
  template:
    metadata:
      labels:
        node: w2
    spec:
      containers:
      - name: alpine
        image: quay.io/cloudwalker/alp.base:latest
        command: ["/bin/sleep", "infinity"]
        imagePullPolicy: IfNotPresent
      tolerations:
      - operator: Exists
      nodeName: antony-w2
EOF
```

</div>

## 2. 檢視 w2 節點上有幾個 Pod

<div class="indent-title-1">

```!
$ kubectl get pods -n kube-system -o wide | grep 'antony-w2'
```

螢幕輸出 : 

```
coredns-85b955d87b-jjkwh  1/1  Running  0            57m  10.244.0.7   antony-w2
kube-flannel-q95bd        1/1  Running  2 (62m ago)  79m  172.20.0.33  antony-w2
kube-proxy-h8k74          1/1  Running  2 (62m ago)  79m  172.20.0.33  antony-w2
```

> 目前在 antony-w2 節點上已有 3 台 pods

</div>



## 3. 部屬 Deployment Yaml 檔

<div class="indent-title-1">

```
$ kubectl apply -f deployment-w2.yaml
```

</div>

## 4. 檢視部屬狀態

<div class="indent-title-1">

```!
$ kubectl get pods -l node=w2
```

螢幕輸出 : 

```
NAME                 READY   STATUS    RESTARTS   AGE
alp-8d96b675-jmkdb   1/1     Running   0          7s
alp-8d96b675-t4x2v   1/1     Running   0          7s
alp-8d96b675-xmp2b   1/1     Running   0          7s
```

</div>

## 5. 把 Pod 數量 Scale 到 20 個

<div class="indent-title-1">

```!
$ kubectl scale deploy alp --replicas=20
```

</div>

## 6. 檢視部屬狀態

<div class="indent-title-1">

```!
$ kubectl get pods -l node=w2
```

螢幕輸出 : 

```
NAME                 READY   STATUS    RESTARTS   AGE
alp-8d96b675-5b7gq   1/1     Running   0          34s
alp-8d96b675-824tt   1/1     Running   0          34s
alp-8d96b675-9q8jn   1/1     Running   0          34s
alp-8d96b675-cgbxq   1/1     Running   0          34s
alp-8d96b675-cp59z   1/1     Running   0          34s
alp-8d96b675-d8775   1/1     Running   0          34s
alp-8d96b675-dbh5r   1/1     Running   0          34s
alp-8d96b675-g58ks   1/1     Running   0          34s
alp-8d96b675-gjlqp   1/1     Running   0          34s
alp-8d96b675-jmkdb   1/1     Running   0          8m6s
alp-8d96b675-jtfx7   1/1     Running   0          34s
alp-8d96b675-khrsq   1/1     Running   0          34s
alp-8d96b675-l7sgf   1/1     Running   0          34s
alp-8d96b675-n7ghc   1/1     Running   0          34s
alp-8d96b675-p8cvs   1/1     Running   0          34s
alp-8d96b675-s9xd6   1/1     Running   0          34s
alp-8d96b675-t4x2v   1/1     Running   0          8m6s
alp-8d96b675-xmp2b   1/1     Running   0          8m6s
alp-8d96b675-z5dz7   1/1     Running   0          34s
alp-8d96b675-zv6p7   1/1     Running   0          34s
```

</div>


## 7. 把 Pod 數量 Scale 到 107 個

<div class="indent-title-1">

```!
$ kubectl scale deploy alp --replicas=107
```

</div>

## 8. 檢視部屬狀態

<div class="indent-title-1">

```!
$ kubectl get pods -l node=w2 --no-headers=true | grep 'Running' | wc -l
```

螢幕輸出 : 

```
107
```

</div>

## 9. 把 Pod 數量 Scale 到 108 個 (此時總量會超過 110)

<div class="indent-title-1">

```!
$ kubectl scale deploy alp --replicas=108
```

</div>

## 10. 檢視部屬狀態


<div class="indent-title-1">

```!
$ kubectl get pods -l node=w2
```

螢幕輸出 : 

```
NAME                   READY   STATUS      RESTARTS   AGE
alp-779f49464f-24r7q   1/1     Running     0          2m57s
alp-779f49464f-26245   1/1     Running     0          2m58s
alp-779f49464f-2h85w   0/1     OutOfpods   0          2s
alp-779f49464f-2h896   1/1     Running     0          2m58s
alp-779f49464f-2hcp8   1/1     Running     0          2m58s
alp-779f49464f-2lcc4   1/1     Running     0          2m57s
alp-779f49464f-2ndqf   1/1     Running     0          2m58s
alp-779f49464f-2w2ct   1/1     Running     0          4m27s
alp-779f49464f-2x9fn   1/1     Running     0          4m27s
alp-779f49464f-2xb5q   0/1     OutOfpods   0          4s
alp-779f49464f-42gq5   1/1     Running     0          2m56s
alp-779f49464f-48zhp   0/1     OutOfpods   0          5s
alp-779f49464f-4hbtq   0/1     OutOfpods   0          2s
alp-779f49464f-4jvzs   1/1     Running     0          2m59s
alp-779f49464f-4lvgr   1/1     Running     0          2m59s
alp-779f49464f-4qmgk   0/1     OutOfpods   0          0s
...以下太多省略
```

> 此時會有一堆 Pod 的 STATUS 呈現 `OutOfpods`，並且 STATUS 是 `OutOfpods` 的 Pod 總數還會不斷增長

</div>

## 11. 查看狀態為 OutOfpods 的 Deployment Pod 總數

<div class="indent-title-1">

```!
$ kubectl get pods -l node=w2 | grep OutOfpods | wc -l
```

螢幕輸出 : 

```
5824
```

</div>

## 12. 查看 Deployment 狀態

<div class="indent-title-1">

```!
$ kubectl get deploy alp
```

螢幕輸出 : 

```
NAME   READY     UP-TO-DATE   AVAILABLE   AGE
alp    107/108   108          107         11m
```

> 狀態是 `OutOfpods` 的 pod 看起來並沒有被 Deployment Controller 掌管

</div>

## 13. 查看 replicaset 狀態

<div class="indent-title-1">

```!
$ kubectl get rs
```

螢幕輸出 : 

```
NAME             DESIRED   CURRENT   READY   AGE
alp-779f49464f   108       108       107     16m
```

> 狀態是 `OutOfpods` 的 pod 看起來也沒有被 Replicaset Controller 記錄下來

</div>

## 14. 查看 node 資源使用量

<div class="indent-title-1">

```!
$ kubectl top nodes
```

螢幕輸出 : 

```
NAME        CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
antony-m1   243m         2%     1595Mi          5%
antony-w1   9m           0%     609Mi           1%
antony-w2   1227m        12%    1334Mi          4%
```

</div>


## 15. 查看狀態為 Running 的 Pod 是否有被影響

<div class="indent-title-1">

```!
$ kubectl get pods -l node=w2 | grep -i runn | wc -l
```

螢幕輸出 : 

```
107
```

</div>

## 16. 查看 狀態為 OutOfpods 的 Pod 詳細資訊

<div class="indent-title-1">

```!
$ kubectl describe pod alp-779f49464f-zspj6
```

螢幕輸出 : 

```!
Name:             alp-779f49464f-zspj6
Namespace:        default
Priority:         0
Service Account:  default
Node:             antony-w2/
Start Time:       Mon, 25 Dec 2023 23:21:25 +0800
Labels:           node=w2
                  pod-template-hash=779f49464f
Annotations:      <none>
Status:           Failed
Reason:           OutOfpods
Message:          Pod was rejected: Node didn't have enough resource: pods, requested: 1, used: 110, capacity: 110
IP:
IPs:              <none>
Controlled By:    ReplicaSet/alp-779f49464f
Containers:
  alpine:
    Image:      quay.io/cloudwalker/alp.base:latest
    Port:       <none>
    Host Port:  <none>
    Command:
      /bin/sleep
      infinity
    Environment:  <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-qgkvc (ro)
Volumes:
  kube-api-access-qgkvc:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 op=Exists
Events:
  Type     Reason     Age   From     Message
  ----     ------     ----  ----     -------
  Warning  OutOfpods  12m   kubelet  Node didn't have enough resource: pods, requested: 1, used: 110, capacity: 110
```

> 看到 Event: `Node didn't have enough resource: pods, requested: 1, used: 110, capacity: 110`
> Pod 也並未得到 IP Address。

</div>

## 17. 再次查看狀態為 OutOfpods 的 Deployment Pod 總數

<div class="indent-title-1">

```!
$ kubectl get pods -l node=w2 | grep OutOfpods | wc -l
```

螢幕輸出 : 

```
11141
```

> 狀態為 OutOfpods 的 Deployment Pod 總數會一直增加，最後測到 `12529` 個不想等了。

</div>

## 結論

1. **即使節點的 CPU、記憶體、Disk ...等硬體資源很夠力，但只要在單一節點上 kubelet 檢查到超過 110 個 Pod 總數，新建立的 Pod 是無法 Running 的**
2. 狀態為 OutOfpods 的 Deployment Pod 總數會一直增加似乎會一直增加到 Node 的資源被使用完畢
3. 狀態為 OutOfpods 的 Deployment Pod 總數會一直增加的原因，推測是因為 Deployment 需要把 Pod 數量擴充到 108 個並且狀態都是 `Running`，但是新建立出來的 Pod 因為超過 `max-pods=110` 的限制，所以狀態一直是 `OutOfpods`，一直無限循環。

