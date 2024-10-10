---
title: "StatefulSets"
date: 2020-06-26T15:17:20+02:00
draft: false
weight: 45
---

## Prérequis

- Minikube [Install](https://kubernetes.io/fr/docs/tasks/tools/install-minikube/#installez-minikube-par-t%C3%A9l%C3%A9chargement-direct)  [Driver none](https://kubernetes.io/docs/setup/learning-environment/minikube/#specifying-the-vm-driver)
- kubectl [Install](https://kubernetes.io/fr/docs/tasks/tools/install-kubectl/)
- Stern [Docs](https://kubernetes.io/blog/2016/10/tail-kubernetes-with-stern/) [Release](https://github.com/stern/stern/releases)
- jq [Install](https://stedolan.github.io/jq/download/)
- 3 terminal SSH


## Preparation

Si vous exécutez ce tutoriel dans Minikube, vous devez déployer un seul noeud :

```
minikube stop
minikube delete --all
minikube start --vm-driver=none
```

## StatefulSet


Un StatefulSet fournit une identité unique aux Pods qui le gèrent. Il peut être utilisé lorsque votre application nécessite un identifiant réseau unique ou un stockage de persistance à travers la (re)programmation des pods ou une certaine garantie sur l'ordre de déploiement et de mise à l'échelle.

L'un des exemples les plus typiques de l'utilisation des StatefulSets est le déploiement de serveurs primaires et secondaires (par exemple, un cluster de base de données) pour lequel vous devez connaître à l'avance le nom d'hôte de chacun des serveurs pour démarrer le cluster. De même, lorsque vous effectuez une montée en charge ou une descente en charge, vous souhaitez le faire dans un ordre précis (par exemple, vous souhaitez démarrer d'abord le nœud primaire, puis le nœud secondaire).

StatefulSet est créé en utilisant la ressource Kubernetes StatefulSet avec un service sans headless :

```
vim  apps/kubefiles/quarkus-statefulset.yaml
```

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: quarkus-statefulset
  labels:
    app: quarkus-statefulset
spec:
  selector:
    matchLabels:
      app: quarkus-statefulset
  serviceName: "quarkus"
  replicas: 2
  template:
    metadata:
      labels:
        app: quarkus-statefulset
    spec:
      containers:
      - name: quarkus-statefulset
        image: quay.io/rhdevelopers/quarkus-demo:v1
        ports:
        - containerPort: 8080
          name: web
```

1- Définit le nom du statefulset utilisé comme nom d'hôte.

Le nom d'hôte suit le même schéma dans tous les cas serviceName + un nombre commençant à 0 et il est incrémenté de un pour chaque réplique.

Et un service headless :

```
---
apiVersion: v1
kind: Service
metadata:
  name: quarkus-statefulset
  labels:
    app: quarkus-statefulset
spec:
  ports:
  - port: 8080
    name: web
  clusterIP: None (1)
  selector:
    app: quarkus-statefulset
```

1- Rend le service headless.

```
kubectl apply -f apps/kubefiles/quarkus-statefulset.yaml

kubectl get pods
```

```
NAME                     READY   STATUS    RESTARTS   AGE
quarkus-statefulset-0   1/1     Running   0          12s
```

Remarquez que le nom du pod est le serviceName avec un 0, car il s'agit de la première instance.

```
kubectl get statefulsets
```

```
NAME                   READY   AGE
quarkus-statefulset   1/1     109s
```

Maintenant, on scale l'application avec 3 instances

```
kubectl scale statefulset quarkus-statefulset --replicas=3

kubectl get pods
```

```
NAME                     READY   STATUS    RESTARTS   AGE
quarkus-statefulset-0   1/1     Running   0          95s
quarkus-statefulset-1   1/1     Running   0          2s
quarkus-statefulset-2   1/1     Running   0          1s
```

Remarquez que le nom des Pods utilise la même nomenclature de serviceName + numéro incrémental.

De plus, si vous vérifiez l'ordre des événements dans le cluster Kubernetes, vous remarquerez que le nom du Pod se terminant par -1 est créé en premier, puis celui se terminant par -2.

```
kubectl get events --sort-by=.metadata.creationTimestamp
```

```
4m4s        Normal   SuccessfulCreate          statefulset/quarkus-statefulset   create Pod quarkus-statefulset-1 in StatefulSet quarkus-statefulset successful
4m3s        Normal   Pulled                    pod/quarkus-statefulset-1         Container image "quay.io/rhdevelopers/quarkus-demo:v1" already present on machine
4m3s        Normal   Scheduled                 pod/quarkus-statefulset-2         Successfully assigned default/quarkus-statefulset-2 to kube
4m3s        Normal   Created                   pod/quarkus-statefulset-1         Created container quarkus-statefulset
4m3s        Normal   Started                   pod/quarkus-statefulset-1         Started container quarkus-statefulset
4m3s        Normal   SuccessfulCreate          statefulset/quarkus-statefulset   create Pod quarkus-statefulset-2 in StatefulSet quarkus-statefulset successful
4m2s        Normal   Pulled                    pod/quarkus-statefulset-2         Container image "quay.io/rhdevelopers/quarkus-demo:v1" already present on machine
4m2s        Normal   Created                   pod/quarkus-statefulset-2         Created container quarkus-statefulset
4m2s        Normal   Started                   pod/quarkus-statefulset-2         Started container quarkus-statefulset
```

Enfin, si nous réduisons à deux instances, celle qui est détruite n'est pas choisie au hasard, mais celle qui a été lancée plus tard (quarkus.statefulset-2).


```
kubectl scale statefulset quarkus-statefulset --replicas=2

kubectl get pods
```

```
NAME                     READY   STATUS        RESTARTS   AGE
quarkus-statefulset-0   1/1     Running       0          9m22s
quarkus-statefulset-1   1/1     Running       0          7m49s
quarkus-statefulset-2   0/1     Terminating   0          7m48s
```

Supprimer les ressources

```
kubectl delete -f apps/kubefiles/quarkus-statefulset.yaml
```
