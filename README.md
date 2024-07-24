# Домашнее задание к занятию «Запуск приложений в K8S»

### Цель задания

В тестовой среде для работы с Kubernetes, установленной в предыдущем ДЗ, необходимо развернуть Deployment с приложением, состоящим из нескольких контейнеров, и масштабировать его.

------

### Чеклист готовности к домашнему заданию

1. Установленное k8s-решение (например, MicroK8S).
2. Установленный локальный kubectl.
3. Редактор YAML-файлов с подключённым git-репозиторием.

------

### Инструменты и дополнительные материалы, которые пригодятся для выполнения задания

1. [Описание](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) Deployment и примеры манифестов.
2. [Описание](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/) Init-контейнеров.
3. [Описание](https://github.com/wbitt/Network-MultiTool) Multitool.

------

### Задание 1. Создать Deployment и обеспечить доступ к репликам приложения из другого Pod

1. Создать Deployment приложения, состоящего из двух контейнеров — nginx и multitool. Решить возникшую ошибку.  
[deployment.yml](https://github.com/PatKolzin/kuber-1.3/blob/main/deployment.yml)

```
pat@yubunta:~/kuber/1-3$ kubectl apply -n netology -f deployment.yml
deployment.apps/nginx-multitool-deploy created

pat@yubunta:~/kuber/1-3$ k get deployments -n netology
NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
pat@yubunta:~/kuber/1-3$ k get po -o wide
NAME                          READY   STATUS     RESTARTS   AGE    IP            NODE      NOMINATED NODE   READINESS GATES
nginx-init-64879c8598-cpp2l   0/1     Init:0/1   0          6m5s   10.1.198.67   yubunta   <none>           <none>

```

2. После запуска увеличить количество реплик работающего приложения до 2.

```
pat@yubunta:~/kuber/1-3$ kubectl apply -n netology -f deployment.yml
deployment.apps/nginx-deployment configured

pat@yubunta:~/kuber/1-3$ k get pods -o wide
NAME                                      READY   STATUS    RESTARTS   AGE   IP            NODE      NOMINATED NODE   READINESS GATES
nginx-multitool-deploy-856c85c765-92bdv   2/2     Running   0          78s   10.1.198.69   yubunta   <none>           <none>
nginx-multitool-deploy-856c85c765-vqlhx   2/2     Running   0          78s   10.1.198.68   yubunta   <none>           <none>

```

3. Продемонстрировать количество подов до и после масштабирования.
4. Создать Service, который обеспечит доступ до реплик приложений из п.1.  
[service.yml](https://github.com/PatKolzin/kuber-1.3/blob/main/service.yml)
```
pat@yubunta:~/kuber/1-3$ k apply -f service.yml 
service/nginx-multitool-svc created
pat@yubunta:~/kuber/1-3$ k get svc -o wide
NAME                  TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                   AGE   SELECTOR
kubernetes            ClusterIP   10.152.183.1     <none>        443/TCP                   12d   <none>
nginx-multitool-svc   ClusterIP   10.152.183.119   <none>        80/TCP,1180/TCP,443/TCP   18s   app=ng-multi
pat@yubunta:~/kuber/1-3$ 

```
5. Создать отдельный Pod с приложением multitool и убедиться с помощью `curl`, что из пода есть доступ до приложений из п.1.
[test_multitool.yml](https://github.com/PatKolzin/kuber-1.3/blob/main/test_multitool.yml)
```
pat@yubunta:~/kuber/1-3$ k apply -f test_multitool.yml 
pod/test-multitool created
pat@yubunta:~/kuber/1-3$ k get po -o wide
NAME                                      READY   STATUS    RESTARTS   AGE   IP            NODE      NOMINATED NODE   READINESS GATES
nginx-multitool-deploy-856c85c765-92bdv   2/2     Running   0          6m    10.1.198.69   yubunta   <none>           <none>
nginx-multitool-deploy-856c85c765-vqlhx   2/2     Running   0          6m    10.1.198.68   yubunta   <none>           <none>
test-multitool                            1/1     Running   0          9s    10.1.198.70   yubunta   <none>           <none>
pat@yubunta:~/kuber/1-3$ 



pat@yubunta:~/kuber/1-3$ microk8s kubectl exec -it test-multitool -- curl 10.152.183.119:80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>


pat@yubunta:~/kuber/1-3$ microk8s kubectl exec -it test-multitool -- curl 10.152.183.119:443
<html>
<head><title>400 The plain HTTP request was sent to HTTPS port</title></head>
<body>
<center><h1>400 Bad Request</h1></center>
<center>The plain HTTP request was sent to HTTPS port</center>
<hr><center>nginx/1.24.0</center>
</body>
</html>


pat@yubunta:~/kuber/1-3$ microk8s kubectl exec -it test-multitool -- curl 10.152.183.119:1180
WBITT Network MultiTool (with NGINX) - nginx-multitool-deploy-856c85c765-nnf8z - 10.1.198.117 - HTTP: 1180 , HTTPS: 443 . (Formerly praqma/network-multitool)


```

------

### Задание 2. Создать Deployment и обеспечить старт основного контейнера при выполнении условий

1. Создать Deployment приложения nginx и обеспечить старт контейнера только после того, как будет запущен сервис этого приложения.

[nginx-deployment.yaml](https://github.com/PatKolzin/kuber-1.3/blob/main/nginx-init.yml)

```
pat@yubunta:~/kuber/1-3$ k apply -f nginx-init.yml 
deployment.apps/nginx-init created

```

2. Убедиться, что nginx не стартует. В качестве Init-контейнера взять busybox.
```
pat@yubunta:~/kuber/1-3$ k get po -o wide
NAME                          READY   STATUS     RESTARTS   AGE   IP            NODE      NOMINATED NODE   READINESS GATES
nginx-init-64879c8598-cpp2l   0/1     Init:0/1   0          11s   10.1.198.67   yubunta   <none>           <none>
pat@yubunta:~/kuber/1-3$ k logs nginx-init-64879c8598-cpp2l
Defaulted container "nginx" out of: nginx, init-busybox (init)
Error from server (BadRequest): container "nginx" in pod "nginx-init-64879c8598-cpp2l" is waiting to start: PodInitializing

```

3. Создать и запустить Service. Убедиться, что Init запустился.  
[nginx-service.yaml](https://github.com/PatKolzin/kuber-1.3/blob/main/svc-nginx-init.yml)
```
pat@yubunta:~/kuber/1-3$ k apply -f svc-nginx-init.yml 
service/svc-nginx-init created
pat@yubunta:~/kuber/1-3$ k get svc -o wide
NAME             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE   SELECTOR
kubernetes       ClusterIP   10.152.183.1     <none>        443/TCP   12d   <none>
svc-nginx-init   ClusterIP   10.152.183.231   <none>        80/TCP    13s   app=web-init


```

4. Проверка пода после запуска сервиса.
```
pat@yubunta:~/kuber/1-3$ k get po -o wide
NAME                          READY   STATUS     RESTARTS   AGE    IP            NODE      NOMINATED NODE   READINESS GATES
nginx-init-64879c8598-cpp2l   1/1     Init:1/1   0          6m5s   10.1.198.67   yubunta   <none>           <none>

```


------

### Правила приема работы

1. Домашняя работа оформляется в своем Git-репозитории в файле README.md. Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд `kubectl` и скриншоты результатов.
3. Репозиторий должен содержать файлы манифестов и ссылки на них в файле README.md.


