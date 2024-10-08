---
title: "Service"
date: 2020-06-26T15:17:20+02:00
draft: false
weight: 15
---

## Prérequis

- Minikube [Install](https://kubernetes.io/fr/docs/tasks/tools/install-minikube/#installez-minikube-par-t%C3%A9l%C3%A9chargement-direct)  [Driver none](https://kubernetes.io/docs/setup/learning-environment/minikube/#specifying-the-vm-driver)
- kubectl [Install](https://kubernetes.io/fr/docs/tasks/tools/install-kubectl/)
- Stern [Docs](https://kubernetes.io/blog/2016/10/tail-kubernetes-with-stern/) [Release](https://github.com/stern/stern/releases)
- jq [Install](https://stedolan.github.io/jq/download/)
- 3 terminal SSH



Cela fait suite à la création du déploiement dans le chapitre précédent.

Assurez-vous que vous êtes dans le bon espace de noms :

```
kubectl config set-context --current --namespace=myspace
```

Assurez-vous que vous avez le Déploiement :

```
kubectl get deployments
```

```
NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
quarkus-demo-deployment   3/3     3            3           8m33s
```

Assurez-vous que vous avez un RS :

```
kubectl get rs
```

```
NAME                                 DESIRED   CURRENT   READY   AGE
quarkus-demo-deployment-5979886fb7   3         3         3       8m56s
```

Assurez-vous d'avoir des Pods :


```
kubectl get pods
```

```
NAME                                       READY   STATUS    RESTARTS   AGE
quarkus-demo-deployment-5979886fb7-c888m   1/1     Running   0          9m17s
quarkus-demo-deployment-5979886fb7-gdtnz   1/1     Running   0          9m17s
quarkus-demo-deployment-5979886fb7-grf59   1/1     Running   0          9m17s
```

Créer un service

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: the-service
spec:
  selector:
    app: quarkus-demo
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: LoadBalancer
EOF
```

```
watch kubectl get services
```

```
NAME    TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
myapp   LoadBalancer   172.30.103.41   <pending>     8080:31974/TCP   4s
```

```
minikube addons enable ingress
kubectl get pods -n ingress-nginx
```

```
NAME                                        READY   STATUS      RESTARTS    AGE
ingress-nginx-admission-create-g9g49        0/1     Completed   0          11m
ingress-nginx-admission-patch-rqp78         0/1     Completed   1          11m
ingress-nginx-controller-59b45fb494-26npt   1/1     Running     0          11m
```


```
minikube tunnel
```

```
watch kubectl get services
```

```
NAME          TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)          AGE
the-service   LoadBalancer   10.111.248.227   10.111.248.227   80:32591/TCP     52m
```

Créer les variables IP et PORT :

```
minikube service the-service --url -n myspace
```

```
http://172.17.0.15:31637
```

```
URL=http://172.17.0.15:31637
```

```
curl $URL
```

```
Supersonic Subatomic Java with Quarkus quarkus-demo-deployment-5979886fb7-grf59:1
```

### A vous de jouez !

Créer des services pour les déploiements existants

```
kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-conf
data:
  nginx.conf: |
    user nginx;
    worker_processes  3;
    error_log  /var/log/nginx/error.log;
    events {
      worker_connections  10240;
    }
    http {
      server {
          listen       80;
          server_name  _;
          root   /usr/share/nginx/html;
          index  index.html index.htm;

          location /europe {
              alias /usr/share/nginx/html/;
              index  index.html index.html;
          }
          location /asia {
              alias /usr/share/nginx/html/;
              index  index.html index.html;
          }
          location / {
              root   /usr/share/nginx/html;
              index  index.html index.htm;
          }
      }
    }

---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: europe
  name: europe
spec:
  progressDeadlineSeconds: 600
  replicas: 2
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: europe
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: europe
    spec:
      containers:
      - image: nginx:1.21.5-alpine
        imagePullPolicy: IfNotPresent
        name: c
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /usr/share/nginx/html
          name: html
        - mountPath: /etc/nginx
          name: nginx-conf
          readOnly: true
      dnsPolicy: ClusterFirst
      initContainers:
      - command:
        - sh
        - -c
        - echo 'hello, you reached EUROPE' > /html/index.html
        image: busybox:1.28
        imagePullPolicy: IfNotPresent
        name: init-container
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /html
          name: html
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
      - emptyDir: {}
        name: html
      - configMap:
          defaultMode: 420
          items:
          - key: nginx.conf
            path: nginx.conf
          name: nginx-conf
        name: nginx-conf
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: asia
  name: asia
spec:
  progressDeadlineSeconds: 600
  replicas: 2
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: asia
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: asia
    spec:
      containers:
      - image: nginx:1.21.5-alpine
        imagePullPolicy: IfNotPresent
        name: c
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /usr/share/nginx/html
          name: html
        - mountPath: /etc/nginx
          name: nginx-conf
          readOnly: true
      dnsPolicy: ClusterFirst
      initContainers:
      - command:
        - sh
        - -c
        - echo 'hello, you reached ASIA' > /html/index.html
        image: busybox:1.28
        imagePullPolicy: IfNotPresent
        name: init-container
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /html
          name: html
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
      - emptyDir: {}
        name: html
      - configMap:
          defaultMode: 420
          items:
          - key: nginx.conf
            path: nginx.conf
          name: nginx-conf
        name: nginx-conf
```

Il existe maintenant deux déploiements dans le namespace par défaut être rendus accessibles via une entrée.

Premièrement : créer des services LoadBalancer pour les deux déploiements sur le port 30291 et 30292 et le target-port 80. Les services doivent porter le même nom que les déploiements.


