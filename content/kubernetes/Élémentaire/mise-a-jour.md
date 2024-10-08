---
title: "Mise à jour permanentes"
date: 2020-06-26T15:17:20+02:00
draft: false
weight: 10
---

## Prérequis

- Minikube [Install](https://kubernetes.io/fr/docs/tasks/tools/install-minikube/#installez-minikube-par-t%C3%A9l%C3%A9chargement-direct)  [Driver none](https://kubernetes.io/docs/setup/learning-environment/minikube/#specifying-the-vm-driver)
- kubectl [Install](https://kubernetes.io/fr/docs/tasks/tools/install-kubectl/)
- Stern [Docs](https://kubernetes.io/blog/2016/10/tail-kubernetes-with-stern/) [Release](https://github.com/stern/stern/releases)
- jq [Install](https://stedolan.github.io/jq/download/)
- 3 terminal SSH


Assurez-vous que vous êtes dans le bon espace de noms :

```
kubectl config set-context --current --namespace=myspace
```

Déployez l'application Spring Boot si nécessaire :

```
kubectl apply -f apps/kubefiles/myboot-deployment-resources-limits.yml
kubectl apply -f apps/kubefiles/myboot-service.yml
```

Terminal 1 : regardez les Pods.

```
watch kubectl get pods
```

Terminal 2: curl loop the service.

```
IP=$(minikube ip)
PORT=$(kubectl get service/myboot -o jsonpath="{.spec.ports[*].nodePort}")
```

Réaliser une requete sur le service :

```
curl $IP:$PORT
```

Et lancez le script de la boucle :

```
while true
do curl $IP:$PORT
sleep .3
done
```


Terminal 3 : Exécuter des commandes.

Décrire le déploiement :


```
kubectl describe deployment myboot
```

```
.
.
.
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
.
.
.
```

Les options StrategyType comprennent RollingUpdate et Recreate :

Modifier les repliquas :

```
kubectl edit deployment myboot
```

Recherchez les "réplicas" :

```
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: myboot
```

Et mettez à jour à "2" :

```
spec:
  progressDeadlineSeconds: 600
  replicas: 2
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: myboot
```

Enregistrez et un nouveau pod prendra vie :

```
kubectl get pods
```

```
NAME                     READY   STATUS    RESTARTS   AGE
myboot-d78fb6d58-2fqml   1/1     Running   0          25s
myboot-d78fb6d58-ljkjp   1/1     Running   0          3m
```

Modifiez l'image associée au déploiement :

```
kubectl edit deployment myboot
```

Trouvez l'attribut de l'image :

```
    spec:
      containers:
      - image: quay.io/rhdevelopers/myboot:v1
        imagePullPolicy: IfNotPresent
        name: myboot
```

et changez l'image myboot:v2 :

```
    spec:
      containers:
      - image: quay.io/rhdevelopers/myboot:v2
        imagePullPolicy: IfNotPresent
        name: myboot
```

```
kubectl get pods
```

```
NAME                      READY   STATUS              RESTARTS   AGE
myboot-7fbc4b97df-4ntmk   1/1     Running             0          9s
myboot-7fbc4b97df-qtkzj   0/1     ContainerCreating   0          0s
myboot-d78fb6d58-2fqml    1/1     Running             0          3m29s
myboot-d78fb6d58-ljkjp    1/1     Terminating         0          8m
```


Et la sortie du terminal 2 :

```
Aloha from Spring Boot! 211 on myboot-d78fb6d58-2fqml
Aloha from Spring Boot! 212 on myboot-d78fb6d58-2fqml
Bonjour from Spring Boot! 0 on myboot-7fbc4b97df-4ntmk
Bonjour from Spring Boot! 1 on myboot-7fbc4b97df-4ntmk
```

Vérifiez l'état du déploiement :

```
kubectl rollout status deployment myboot
```

```
deployment "myboot" successfully rolled out
```

Remarquez qu'il y a un nouveau RS :

```
kubectl get rs
```

```
NAME                DESIRED   CURRENT   READY   AGE
myboot-7fbc4b97df   2         2         2       116s
myboot-d78fb6d58    0         0         0       10m
```

Décrire le déploiement :

```
kubectl describe deployment myboot
```

Et consultez la section "Événements" :

```
...
Events:
  Type    Reason             Age    From                   Message
  ----    ------             ----   ----                   -------
  Normal  ScalingReplicaSet  16m    deployment-controller  Scaled up replica set myboot-d78fb6d58 to 1
  Normal  ScalingReplicaSet  6m15s  deployment-controller  Scaled up replica set myboot-d78fb6d58 to 2
  Normal  ScalingReplicaSet  2m55s  deployment-controller  Scaled up replica set myboot-7fbc4b97df to 1
  Normal  ScalingReplicaSet  2m46s  deployment-controller  Scaled down replica set myboot-d78fb6d58 to 1
  Normal  ScalingReplicaSet  2m46s  deployment-controller  Scaled up replica set myboot-7fbc4b97df to 2
  Normal  ScalingReplicaSet  2m37s  deployment-controller  Scaled down replica set myboot-d78fb6d58 to 0
```

Retour à la version 1 :

```
kubectl set image deployment myboot myboot=quay.io/rhdevelopers/myboot:v1
```