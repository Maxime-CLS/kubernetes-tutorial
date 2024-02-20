---
title: "Ingress"
date: 2020-06-26T15:17:20+02:00
draft: false
weight: 5
---

## Prérequis

- Minikube [Install](https://kubernetes.io/fr/docs/tasks/tools/install-minikube/#installez-minikube-par-t%C3%A9l%C3%A9chargement-direct)  [Driver none](https://kubernetes.io/docs/setup/learning-environment/minikube/#specifying-the-vm-driver)
- kubectl [Install](https://kubernetes.io/fr/docs/tasks/tools/install-kubectl/)
- Stern [Docs](https://kubernetes.io/blog/2016/10/tail-kubernetes-with-stern/) [Release](https://github.com/wercker/stern/releases)
- jq [Install](https://stedolan.github.io/jq/download/)
- 3 terminal SSH

Cette section ne fonctionne pas sur MacOs. Vous devez absolument être sur une machine Linux.

## Activer le contrôleur d'entrée

Si vous utilisez minikube, vous devez activer le contrôleur NGNIX Ingress.

```
minikube addons enable ingress
```

Attendez une minute ou deux et vérifiez qu'il a été déployé correctement :

```
kubectl get pods -n ingress-nginx
```

```
ingress-nginx-admission-create-lqfh2        0/1     Completed   0          6m28s
ingress-nginx-admission-patch-z2lzj         0/1     Completed   2          6m28s
ingress-nginx-controller-69ccf5d9d8-95xgp   1/1     Running     0          6m28s
```

## Déployer l'application

```
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: quarkus-demo-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: quarkus-demo
  template:
    metadata:
      labels:
        app: quarkus-demo
        env: dev
    spec:
      containers:
      - name: quarkus-demo
        image: quay.io/rhdevelopers/quarkus-demo:v1
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
EOF
```

Exposez le service :


```
kubectl expose deployment quarkus-demo-deployment --type=LoadBalancer --port=8080

kubectl get service quarkus-demo-deployment
```

```
NAME                      TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
quarkus-demo-deployment   NodePort   10.105.106.66   <none>        8080:30408/TCP   11s
```

```
IP=$(kubectl get service myapp -o jsonpath="{.status.loadBalancer.ingress[0].ip}")
PORT=$(kubectl get service myapp -o jsonpath="{.spec.ports[*].port}")
```

Réaliser une requete sur le service :

```
curl $IP:$PORT
```

## Configuration l'Ingress

Une ressource d'entrée est définie comme suit :

```
vim apps/kubefiles/demo-ingress.yaml
```


```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
  rules:
  - host: kube-team.info
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: quarkus-demo-deployment
            port:
              number: 8080

```

```
kubectl apply -f apps/kubefiles/demo-ingress.yaml
```

Obtenir les informations de la ressource Ingress :

```
kubectl get ingress
```

```
NAME              CLASS    HOSTS                 ADDRESS          PORTS   AGE
example-ingress   <none>   kube-team.info   192.168.99.115   80      68s
```

Vous devez attendre que le champ d'adresse soit défini. Cela peut prendre quelques minutes.

Modifiez le fichier /etc/hosts pour faire pointer le nom d'hôte vers l'adresse Ingress.

```
minikube ip
```

```
10.240.145.124
```

```
sudo vim /etc/hosts
```

```
10.240.145.124 kube-team.info
```

```
curl kube-team.info
```

Si vous avez un proxy :

```
curl --noproxy kube-team.info kube-team.info
```

```
Supersonic Subatomic Java with Quarkus quarkus-demo-deployment-8cf45f5c8-qmzwl:1
```

## Deuxième déploiement

Déployer une deuxième version du service :

```
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mynode-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mynode
  template:
    metadata:
      labels:
        app: mynode
    spec:
      containers:
      - name: mynode
        image: quay.io/rhdevelopers/mynode:v1
        ports:
        - containerPort: 8000
EOF
```

```
kubectl expose deployment mynode-deployment --type=NodePort --port=8000
```

## Mise à jour de l'Ingress

Ensuite, vous devez mettre à jour la ressource Ingress avec le nouveau chemin :


```
vim apps/kubefiles/demo-ingress-2.yaml
```

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
  rules:
  - host: kube-team.info
    http:
      paths:
      - path: /
        backend:
          serviceName: quarkus-demo-deployment
          servicePort: 8080
      - path: /v2
        backend:
          serviceName: mynode-deployment
          servicePort: 8000
```

```
kubectl apply -f apps/kubefiles/demo-ingress-2.yaml
```

Tester :

```
curl kube-team.info
```

```
Supersonic Subatomic Java with Quarkus quarkus-demo-deployment-8cf45f5c8-qmzwl:2
```

```
curl kube-team.info/v2
```

```
Node Bonjour on mynode-deployment-77c7bf857d-5nfl4 0
```

Supprimer les ressources

```
kubectl delete deployment mynode-deployment
kubectl delete service mynode-deployment

kubectl delete deployment quarkus-demo-deployment
kubectl delete service quarkus-demo-deployment

kubectl delete -f apps/kubefiles/demo-ingress-2.yaml
```