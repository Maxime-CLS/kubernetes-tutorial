---
title: "Service Magic"
date: 2020-06-26T15:17:20+02:00
draft: false
weight: 25
---

## Prérequis

- Minikube [Install](https://kubernetes.io/fr/docs/tasks/tools/install-minikube/#installez-minikube-par-t%C3%A9l%C3%A9chargement-direct)  [Driver none](https://kubernetes.io/docs/setup/learning-environment/minikube/#specifying-the-vm-driver)
- kubectl [Install](https://kubernetes.io/fr/docs/tasks/tools/install-kubectl/)
- Stern [Docs](https://kubernetes.io/blog/2016/10/tail-kubernetes-with-stern/) [Release](https://github.com/wercker/stern/releases)
- jq [Install](https://stedolan.github.io/jq/download/)
- 3 terminal SSH


Créer un namespace


```
kubectl create namespace funstuff
kubectl config set-context --current --namespace=funstuff
```

Deployer une application mypython

```
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mypython-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mypython
  template:
    metadata:
      labels:
        app: mypython
    spec:
      containers:
      - name: mypython
        image: quay.io/rhdevelopers/mypython:v1
        ports:
        - containerPort: 8000
EOF
```


Deployer une application mygo

```
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mygo-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mygo
  template:
    metadata:
      labels:
        app: mygo
    spec:
      containers:
      - name: mygo
        image: quay.io/rhdevelopers/mygo:v1
        ports:
        - containerPort: 8000
EOF
```

Deployer une application mynode

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

Vérifier l'état des applications

```
watch kubectl get pods --show-labels
```

```
NAME                                   READY   STATUS    RESTARTS   AGE     LABELS
mygo-deployment-6d944c5c69-kcvmk       1/1     Running   0          2m11s   app=mygo,pod-template-hash=6d944c5c69
mynode-deployment-fb5457c5-hhz7h       1/1     Running   0          2m1s    app=mynode,pod-template-hash=fb5457c5
mypython-deployment-6874f84d85-2kpjl   1/1     Running   0          3m53s   app=mypython,pod-template-hash=6874f84d85
```

Déployer un service générique

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: my-service
  labels:
    app: mystuff
spec:
  ports:
  - name: http
    port: 8000
  selector:
    inservice: mypods
  type: LoadBalancer
EOF
```

Observer la description de ce service

```
kubectl describe service my-service
```


Observer la présence de EndPoint

```
kubectl get endpoints
```

```
NAME         ENDPOINTS   AGE
my-service   <none>      2m6s
```

Recherche l'adresse IP du EndPoint

```
kubectl get endpoints my-service -o json | jq '.subsets[].addresses[].ip'

```

```
jq: error (at <stdin>:18): Cannot iterate over null (null)
```

Définir les variables IP et PORT

```
IP=$(minikube ip -p devnation)
PORT=$(kubectl get service/my-service -o jsonpath="{.spec.ports[*].nodePort}")
```


Faire une requete au service

```
curl $IP:$PORT
```

Faire une boucle de requete au service

```
while true
do curl $IP:$PORT
sleep .3
done
```


```
curl: (7) Failed to connect to 35.224.233.213 port 8000: Connection refused
curl: (7) Failed to connect to 35.224.233.213 port 8000: Connection refused
```

Ajouter un label à un pod

```
kubectl label pod -l app=mypython inservice=mypods
```

```
curl: (7) Failed to connect to 35.224.233.213 port 8000: Connection refused
Python Hello on mypython-deployment-6874f84d85-2kpjl
Python Hello on mypython-deployment-6874f84d85-2kpjl
Python Hello on mypython-deployment-6874f84d85-2kpjl
```


```
kubectl label pod -l app=mynode inservice=mypods
```

```
Python Hello on mypython-deployment-6874f84d85-2kpjl
Python Hello on mypython-deployment-6874f84d85-2kpjl
Node Hello on mynode-deployment-fb5457c5-hhz7h 0
Node Hello on mynode-deployment-fb5457c5-hhz7h 1
Python Hello on mypython-deployment-6874f84d85-2kpjl
Python Hello on mypython-deployment-6874f84d85-2kpjl
Python Hello on mypython-deployment-6874f84d85-2kpjl
```

```
kubectl label pod -l app=mygo inservice=mypods
```

```
Node Hello on mynode-deployment-fb5457c5-hhz7h 59
Node Hello on mynode-deployment-fb5457c5-hhz7h 60
Go Hello on mygo-deployment-6d944c5c69-kcvmk
Python Hello on mypython-deployment-6874f84d85-2kpjl
Python Hello on mypython-deployment-6874f84d85-2kpjl
```

Recherche les IP du EndPoint

```
kubectl get endpoints my-service -o json | jq '.subsets[].addresses[].ip'
```

```
"10.130.2.43"
"10.130.2.44"
"10.130.2.45"
```

Afficher les pods avec les labels

```
kubectl get pods -o wide
```

Supprimer l'application mypython du service

```
kubectl label pod -l app=mypython inservice-
```

```
kubectl get endpoints my-service -o json | jq '.subsets[].addresses[].ip'-
```

```
"10.130.2.44"
"10.130.2.45"
```

Supprimer le namespace

```
kubectl delete namespace funstuff
```