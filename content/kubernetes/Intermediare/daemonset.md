---
title: "Daemonset"
date: 2020-06-26T15:17:20+02:00
draft: false
weight: 40
---

## Prérequis

- Minikube [Install](https://kubernetes.io/fr/docs/tasks/tools/install-minikube/#installez-minikube-par-t%C3%A9l%C3%A9chargement-direct)  [Driver none](https://kubernetes.io/docs/setup/learning-environment/minikube/#specifying-the-vm-driver)
- kubectl [Install](https://kubernetes.io/fr/docs/tasks/tools/install-kubectl/)
- Stern [Docs](https://kubernetes.io/blog/2016/10/tail-kubernetes-with-stern/) [Release](https://github.com/stern/stern/releases)
- jq [Install](https://stedolan.github.io/jq/download/)
- 3 terminal SSH


Un DaemonSet garantit que tous les nœuds exécutent une copie d'un Pod. Lorsque des nœuds sont ajoutés au cluster, des Pods leur sont ajoutés automatiquement. Lorsque les nœuds sont supprimés, ils ne sont pas replanifiés mais supprimés.

Ainsi, DaemonSet vous permet de déployer un Pod sur tous les nœuds.

## Preparation

Si vous exécutez ce tutoriel dans Minikube, vous devez déployer un premier noeud en utilisant le driver docker :

```
minikube stop
minikube delete --all
minikube start --driver=docker
```

Si vous avez un proxy

```
minikube stop
minikube delete --all
minikube start --docker-env HTTPS_PROXY=$HTTPS_PROXY --docker-env HTTP_PROXY=$HTTP_PROXY --docker-env=NO_PROXY=$no_proxy
```

Ajouter des nœuds supplémentaires pour exécuter cette partie du tutoriel. Vérifiez le nombre de nœuds que vous avez delpoyés en exécutant :

```
kubectl get nodes
```

Si un seul nœud est présent, vous devez créer un nouveau nœud en suivant les étapes suivantes :

```
NAME       STATUS   ROLES    AGE     VERSION
kube       Ready    master   54m     v1.23.1
```

Ayant minikube installé et dans votre PATH, puis exécutez :

```
minikube node add
```

```
kubectl get nodes
```

```
NAME       STATUS   ROLES    AGE     VERSION
kube       Ready    master   54m     v1.23.1
kube-m02   Ready    <none>   2m50s   v1.23.1
```

## Daemonset

Le DaemonSet est créé à l'aide de la ressource Kubernetes DaemonSet :

```
vim  apps/kubefiles/nginx-daemonset.yaml
```


```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-daemonset
  labels:
    app: nginx-daemonset
spec:
  selector:
    matchLabels:
      app: nginx-daemonset
  template:
    metadata:
      labels:
        app: nginx-daemonset
    spec:
      containers:
      - name: nginx-daemonset
        image: nginx
```

```
kubectl apply -f apps/kubefiles/nginx-daemonset.yaml

kubectl get pods -o wide
```

```
NAME                      READY   STATUS    RESTARTS   AGE   IP           NODE            NOMINATED NODE   READINESS GATES
nginx-daemonset-jl2t5     1/1     Running   0          23s   10.244.0.2   multinode       <none>           <none>
nginx-daemonset-r64ql     1/1     Running   0          23s   10.244.1.2   multinode-m02   <none>           <none>
```

Remarquez qu'une instance du pod Nginx est déployée sur chaque nœud.

Supprimer les ressources

```
kubectl delete -f apps/kubefiles/quarkus-daemonset.yaml
```


### A vous de jouer !!!

Dans K8s, les DaemonSets sont souvent utilisés pour configurer certaines choses sur les nœuds.

Créez un DaemonSet nommé configurator, il doit :

être dans l'espace de noms configurator  
utiliser l'image bash  
monter /configurator en tant que volume HostPath sur le nœud sur lequel il s'exécute  
écrire aba997ac-1c89-4d64 dans le fichier /configurator/config sur son nœud via la commande : section  
être maintenu en fonctionnement à l'aide de sleep 1d ou d'une commande similaire après la commande d'écriture du fichier.  
Il n'y a pas de taint sur aucun nœud, ce qui signifie qu'aucune tolérance n'est nécessaire.
