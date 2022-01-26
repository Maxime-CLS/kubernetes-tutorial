---
title: "Service"
date: 2020-06-26T15:17:20+02:00
draft: false
weight: 15
---

## Prérequis

- Minikube [Install](https://kubernetes.io/fr/docs/tasks/tools/install-minikube/#installez-minikube-par-t%C3%A9l%C3%A9chargement-direct)  [Driver none](https://kubernetes.io/docs/setup/learning-environment/minikube/#specifying-the-vm-driver)
- kubectl [Install](https://kubernetes.io/fr/docs/tasks/tools/install-kubectl/)
- Stern [Docs](https://kubernetes.io/blog/2016/10/tail-kubernetes-with-stern/) [Release](https://github.com/wercker/stern/releases)
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
curl $IP:$PORT
```

```
Supersonic Subatomic Java with Quarkus quarkus-demo-deployment-5979886fb7-grf59:1
```