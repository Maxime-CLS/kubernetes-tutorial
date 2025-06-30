---
title: "Déploiement Blue/Green"
date: 2020-06-26T15:17:20+02:00
draft: false
weight: 30
---

## Prérequis

- Minikube [Install](https://kubernetes.io/fr/docs/tasks/tools/install-minikube/#installez-minikube-par-t%C3%A9l%C3%A9chargement-direct)  [Driver none](https://kubernetes.io/docs/setup/learning-environment/minikube/#specifying-the-vm-driver)
- kubectl [Install](https://kubernetes.io/fr/docs/tasks/tools/install-kubectl/)
- Stern [Docs](https://kubernetes.io/blog/2016/10/tail-kubernetes-with-stern/) [Release](https://github.com/stern/stern/releases)
- jq [Install](https://stedolan.github.io/jq/download/)
- 3 terminal SSH


Créer un namespace


```
kubectl create namespace myspace
kubectl config set-context --current --namespace=myspace
```


Vérifier que le namespace est vide


```
kubectl get all
```

```
No resources found in myspace namespace.
```

Créer un fichier de déploiement


```
mkdir -p apps/kubefiles/
vi apps/kubefiles/myboot-deployment-resources-limits.yml
```

myboot-deployment-resources-limits.yml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: myboot
  name: myboot
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myboot
  template:
    metadata:
      labels:
        app: myboot
    spec:
      containers:
      - name: myboot
        image: quay.io/rhdevelopers/myboot:v1
        ports:
          - containerPort: 8080
        resources:
          requests:
            memory: "300Mi"
            cpu: "250m" # 1/4 core
          # NOTE: These are the same limits we tested our Docker Container with earlier
          # -m matches limits.memory and --cpus matches limits.cpu
          limits:
            memory: "900Mi"
            cpu: "1000m" # 1 core
```


Déployer la version 1 de l'applciation myboot

```
kubectl apply -f apps/kubefiles/myboot-deployment-resources-limits.yml
```

Scale l'application avec 2 replicas

```
kubectl scale deployment/myboot --replicas=2
```

Vérifier l'état de l'application

```
watch kubectl get pods --show-labels
```

Créer un fichier pour votre service


```
vi apps/kubefiles/myboot-service.yml
```

myboot-service.yml

```
apiVersion: v1
kind: Service
metadata:
  name: myboot
  labels:
    app: myboot
spec:
  ports:
  - name: http
    port: 8080
  selector:
    app: myboot
  type: LoadBalancer
```

Déployer le service

```
kubectl apply -f apps/kubefiles/myboot-service.yml
```


```
minikube service myapp --url -n mystuff
```

```
http://127.0.0.1:54134
```

terminal 1 

```
URL=http://127.0.0.1:54134
```

Sondez le résultat :

```
while true
do curl $URL
sleep .3
done
```

Créer un fichier de déploiement pour la version 2


```
vi apps/kubefiles/myboot-deployment-resources-limits-v2.yml
```

myboot-deployment-resources-limits-v2.yml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: myboot-next
  name: myboot-next-5
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myboot-next
  template:
    metadata:
      labels:
        app: myboot-next
    spec:
      containers:
      - name: myboot
        image: quay.io/rhdevelopers/myboot:v3
        ports:
          - containerPort: 8080
        resources:
          requests:
            memory: "300Mi"
            cpu: "250m" # 1/4 core
          limits:
            memory: "900Mi"
            cpu: "1000m" # 1 core
```

Déployer la version 2 de l'applciation myboot

```
kubectl apply -f apps/kubefiles/myboot-deployment-resources-limits-v2.yml
```

Vérifiez que le nouveau pod/déploiement porte le nouveau code :

```
PODNAME=$(kubectl get pod -l app=myboot-next -o name)
kubectl exec -it $PODNAME -- curl localhost:8080
```

```
Bonjour from Spring Boot! 1 on myboot-next-66b68c6659-ftcjr
```

Maintenant, mettez à jour le service unique pour pointer vers le nouveau pod et passez au Green :

```
kubectl patch svc/myboot -p '{"spec":{"selector":{"app":"myboot-next"}}}'
```

```
Aloha from Spring Boot! 240 on myboot-d78fb6d58-929wn
Bonjour from Spring Boot! 2 on myboot-next-66b68c6659-ftcjr
Bonjour from Spring Boot! 3 on myboot-next-66b68c6659-ftcjr
Bonjour from Spring Boot! 4 on myboot-next-66b68c6659-ftcjr
```

Vous déterminez que vous préférez l'hawaïen (bleu) au français (vert) et faites un rollback :

Mettez maintenant à jour le service unique pour pointer vers le nouveau pod et passez en Bleu :

```
kubectl patch svc/myboot -p '{"spec":{"selector":{"app":"myboot"}}}'
```

```
Bonjour from Spring Boot! 17 on myboot-next-66b68c6659-ftcjr
Aloha from Spring Boot! 257 on myboot-d78fb6d58-vqvlb
Aloha from Spring Boot! 258 on myboot-d78fb6d58-vqvlb
```

Supprimer les ressources

```
kubectl delete service myboot
kubectl delete deployment myboot
kubectl delete deployment myboot-next
```


### A vous de jouer !

Effectuer un déploiement Green-Blue d'une application.

Déployer l'application suivante : 

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: wonderful
  name: wonderful-v1
spec:
  replicas: 4
  selector:
    matchLabels:
      app: wonderful
      version: v1
  template:
    metadata:
      labels:
        app: wonderful
        version: v1
    spec:
      containers:
      - image: httpd:alpine
        name: httpd
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: wonderful
  name: wonderful
spec:
  ports:
  - port: 30290
    protocol: TCP
    targetPort: 80
  selector:
    app: wonderful
    version: v1
  type: LoadBalancer
```

L'application "wonderful" est exécutée dans le namespace par défaut.

Créer les variables d'environnement IP et Port

```
IP=$(minikube ip)
PORT=$(kubectl get service/wonderful -o jsonpath="{.spec.ports[*].nodePort}")
```

Vous pouvez appeler l'application en utilisant curl $IP:$PORT .

Et maintenant créer une boucle de requete

```
while true
do curl $IP:$PORT
sleep .3
done
```

L'application a un déploiement avec l'image httpd:alpine , mais devrait être basculée sur nginx:alpine .

Le basculement doit se faire instantanément. Cela signifie qu'à partir du moment où l'application est déployée, toutes les nouvelles requêtes doivent utiliser la nouvelle image.

Créer un nouveau Deployment wonderful-v2 qui utilise l'image nginx:alpine avec 4 répliques. Ses Pods doivent avoir les labels app : wonderful et version : v2.

Une fois que tous les nouveaux Pods sont en cours d'exécution, changez l'étiquette du sélecteur du Service wonderful en version : v2.

Enfin, réduire le déploiement wonderful-v1 à 0 réplicas.
