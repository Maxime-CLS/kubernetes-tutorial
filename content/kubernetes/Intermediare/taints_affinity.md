---
title: "Taints et Affinity"
date: 2020-06-26T15:17:20+02:00
draft: false
weight: 30
---

## Prérequis

- Minikube [Install](https://kubernetes.io/fr/docs/tasks/tools/install-minikube/#installez-minikube-par-t%C3%A9l%C3%A9chargement-direct)  [Driver none](https://kubernetes.io/docs/setup/learning-environment/minikube/#specifying-the-vm-driver)
- kubectl [Install](https://kubernetes.io/fr/docs/tasks/tools/install-kubectl/)
- Stern [Docs](https://kubernetes.io/blog/2016/10/tail-kubernetes-with-stern/) [Release](https://github.com/wercker/stern/releases)
- jq [Install](https://stedolan.github.io/jq/download/)
- 3 terminal SSH


Jusqu'à présent, lorsque nous déployions un Pod dans le cluster Kubernetes, il était exécuté sur n'importe quel nœud répondant aux exigences (c'est-à-dire les exigences en matière de mémoire, de CPU, ...).

Cependant, dans Kubernetes, il existe deux concepts qui vous permettent de configurer davantage le planificateur, de sorte que les pods soient affectés aux nœuds en fonction de certains critères commerciaux.

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

## Taints et Tolérance

Une taint est appliquée à un nœud Kubernetes qui signale au planificateur d'éviter ou de ne pas planifier certains pods.

Une tolérance est appliquée à la définition d'un pod et fournit une exception au taint.

Décrivons les nœuds actuels, dans ce cas comme un cluster Minikube est utilisé, vous pouvez voir plusieurs nœuds :

```
kubectl describe nodes | egrep "Name:|Taints:"
```

```
Name:               minikube
Taints:             <none>
Name:               minikube-m02
Taints:             <none>
```

Notez que dans ce cas, les nœuds ne contient pas de taint.

#### Taints

Ajoutons une tache à tous les noeuds :

```
kubectl taint nodes --all=true color=blue:NoSchedule
```

```
node/minikube tainted
node/minikube-m02 tainted
```

La couleur=bleue est simplement une paire clé=valeur pour identifier la taint et NoSchedule est l'effet spécifique.

Maintenant, déployez un nouveau pod.

```
vim apps/kubefiles/myboot-nginx-deployment.yml
```

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
        image: nginx
        ports:
          - containerPort: 8080
```



```
kubectl apply -f apps/kubefiles/myboot-nginx-deployment.yml

kubectl get pods
```

Le pod restera en statut Pending car il n'a pas de nœud programmable disponible.


```
NAME                     READY   STATUS    RESTARTS   AGE
myboot-7f889dd6d-n5z55   0/1     Pending   0          55s
```

```
kubectl describe pod myboot-7f889dd6d-n5z55
```

```
Name:           myboot-7f889dd6d-n5z55
Namespace:      kubetut
Priority:       0
Node:           <none>
Labels:         app=myboot
                pod-template-hash=7f889dd6d
Annotations:    openshift.io/scc: restricted
Status:         Pending

Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type     Reason            Age        From               Message
  ----     ------            ----       ----               -------
  Warning  FailedScheduling  <unknown>  default-scheduler  0/9 nodes are available: 9 node(s) had taints that the pod didn't tolerate.
  Warning  FailedScheduling  <unknown>  default-scheduler  0/9 nodes are available: 9 node(s) had taints that the pod didn't tolerate.
```

Maintenant, enlevons une tare d'un noeud et voyons ce qui se passe :


```
kubectl get nodes
```

```
NAME           STATUS   ROLES                  AGE    VERSION
minikube       Ready    control-plane,master   120m   v1.23.1
minikube-m02   Ready    <none>                 119m   v1.23.1
```

Choisissez un nœud.

```
kubectl taint node minikube-m02 color:NoSchedule-
```

```
node/minikube-m02 untainted
```

```
kubectl describe nodes | egrep "Name:|Taints:"
```

```
Name:               minikube
Taints:             color=blue:NoSchedule
Name:               minikube-m02
Taints:             <none>
```

Et vous devriez voir le pod en attente programmé sur le nouveau nœud non taint.

```
kubectl get pods
```

```
NAME                      READY   STATUS             RESTARTS   AGE
myboot-7f84d7cfc9-2m4lh   1/1     Running            0          7m
```

Supprimer les ressources

```
kubectl delete -f apps/kubefiles/myboot-nginx-deployment.yml
kubectl taint node minikube-m02 color=blue:NoSchedule
```

## Tolerations

Créons un Pod mais contenant une tolérance, afin qu'il puisse être programmé vers un nœud taint.

```
spec:
  tolerations:
  - key: "color"
    operator: "Equal"
    value: "blue"
    effect: "NoSchedule"
  containers:
  - name: myboot
    image: nginx
```

```
vim apps/kubefiles/myboot-toleration.yaml
```

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
      tolerations:
      - key: "color"
        operator: "Equal"
        value: "blue"
        effect: "NoSchedule"
      containers:
      - name: myboot
        image: nginx
        ports:
          - containerPort: 8080
```

```
kubectl apply -f apps/kubefiles/myboot-toleration.yaml

kubectl get pods
```

```
NAME                      READY   STATUS    RESTARTS   AGE
myboot-84b457458b-mbf9r   1/1     Running   0          3m18s
```

Maintenant, bien que tous les nœuds contiennent une taint, le Pod est programmé et exécuté comme nous avons défini une tolérance contre la taint color=blue.


Supprimer les ressources

```
kubectl delete -f apps/kubefiles/myboot-toleration.yaml
```

#### Pas d'exécution de taint

Jusqu'à présent, vous avez vu l'effet du défaut NoSchedule qui signifie que les pods nouvellement créés ne seront pas planifiés à cet endroit, à moins qu'ils ne disposent d'une tolérance prioritaire. Mais remarquez que si nous ajoutons cette taint à un nœud qui a déjà des pods en cours d'exécution/planifiés, cette taint ne les arrêtera pas.

Changeons cela en utilisant l'effet NoExecution.

Tout d'abord, supprimons toutes les altérations précédentes.


```
kubectl taint nodes --all=true color=blue:NoSchedule-
```


Ensuite, déployez un service :


```
kubectl apply -f apps/kubefiles/myboot-nginx-deployment.yml

kubectl get pods
```

```
NAME                     READY   STATUS    RESTARTS   AGE
myboot-7f889dd6d-bkfg5   1/1     Running   0          16s
```

```
kubectl get pod myboot-7f889dd6d-bkfg5 -o json | jq '.spec.nodeName'
```

```
"minikube-m02"
```

```
kubectl taint node minikube-m02 color=blue:NoExecute

kubectl get pods
```

```
NAME                     READY   STATUS        RESTARTS   AGE
myboot-7f889dd6d-8fm2v   1/1     Running       0          14s
myboot-7f889dd6d-bkfg5   1/1     Terminating   0          5m51s
```

Si vous avez plus de nœuds disponibles, alors le Pod est terminé et déployé sur un autre nœud, si ce n'est pas le cas, alors le Pod restera en statut Pending.

Vous pouvez observer cette "reprogrammation" à l'aide de -o wide.

```
watch kubectl get pods -o wide
```

```
NAME                     READY   STATUS        RESTARTS   AGE     IP           NODE
myboot-7f889dd6d-b9tks   1/1     Running       0          6s      10.88.0.5    minikube
myboot-7f889dd6d-l897f   1/1     Terminating   0          9m11s   172.17.0.4   minikube-m02
```

Supprimer les ressources

```
kubectl delete -f apps/kubefiles/myboot-deployment.yml
```

Et supprimer la taint NoExecute

```
kubectl taint node minikube-m02 color=blue:NoExecute-
```

## Affinité et antiaffinité

Il existe une autre façon de changer l'endroit où les Pods sont programmés en utilisant l'affinité et l'anti-affinité Node/Pod. Vous pouvez créer des règles qui non seulement interdisent les endroits où les Pods peuvent s'exécuter mais aussi qui favorisent les endroits où ils doivent s'exécuter.

En plus de créer des affinités entre les pods et les nœuds, vous pouvez également créer des affinités entre les pods. Vous pouvez décider qu'un groupe de pods doit toujours être déployé ensemble sur le(s) même(s) nœud(s). Des raisons telles qu'une communication réseau importante entre les Pods et vous voulez éviter les appels réseau externes ou peut-être les périphériques de stockage partagés.

#### Affinité des nœuds

Déployons un nouveau pod avec une affinité de nœud :

```
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: color
            operator: In
            values:
            - blue
      containers:
      - name: myboot
        image: nginx
```

```
vim apps/kubefiles/myboot-node-affinity.yml
```

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
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: color
                operator: In
                values:
                - blue
      containers:
      - name: myboot
        image: nginx
        ports:
          - containerPort: 8080
```

```
kubectl apply -f apps/kubefiles/myboot-node-affinity.yml

kubectl get pods
```

```
NAME                           READY   STATUS    RESTARTS   AGE
myboot-54d788fdc8-f6wks        0/1     Pending   0          13s
```

Créons une étiquette sur un nœud correspondant à l'expression d'affinité :

```
kubectl get nodes
```

```
NAME           STATUS   ROLES                  AGE   VERSION
minikube       Ready    control-plane,master   27m   v1.23.1
minikube-m02   Ready    <none>                 25m   v1.23.1
```

```
NAME           STATUS   ROLES                  AGE   VERSION
minikube       Ready    control-plane,master   27m   v1.23.1
minikube-m02   Ready    <none>                 25m   v1.23.1
```

```
kubectl label nodes minikube-m02 color=blue
```

```
node/minikube-m02 labeled
```

```
kubectl get pods
```

```
NAME                          READY   STATUS    RESTARTS   AGE
myboot-54d788fdc8-f6wks       1/1     Running   0          7m57s
```

Supprimons l'étiquette du nœud :

```
kubectl label nodes minikube-m02 color-

kubectl get pods
```

```
NAME                         READY   STATUS    RESTARTS   AGE
myboot-54d788fdc8-f6wks      1/1     Running   0          7m57s
```

Comme pour les taints, la règle est définie pendant la phase d'ordonnancement, par conséquent, le Pod n'est pas supprimé.

Il s'agit d'un exemple de règle stricte, si le planificateur de Kubernetes ne trouve pas de nœud avec l'étiquette requise, le pod est rappelé dans l'état Pending. Il est également possible de créer une règle souple, dans laquelle le planificateur Kubernetes tente de répondre aux règles, mais s'il n'y parvient pas, le pod est programmé vers n'importe quel nœud. Dans l'exemple ci-dessous, vous pouvez voir l'utilisation du mot préféré par rapport au mot requis.

```
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: color
            operator: In
            values:
            - blue
```

Supprimer les ressources

```
kubectl delete -f apps/kubefiles/myboot-node-affinity.yml
```

## Pod Affinity/Anti-Affinity

Déployons un nouveau pod avec un Pod Affinity :

```
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - topologyKey: kubernetes.io/hostname (1)
        labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - myboot (2)
  containers:
```

1- La clé de l'étiquette du nœud. Si deux noeuds sont étiquetés avec cette clé et ont des valeurs identiques, l'ordonnanceur traite les deux noeuds comme étant dans la même topologie. Dans ce cas, le nom d'hôte est une étiquette qui est différente pour chaque noeud.
2- L'affinité est avec les pods étiquetés avec app=myboot.

```
vim apps/kubefiles/myboot-pod-affinity.yml
```

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: myboot2
  name: myboot2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myboot2
  template:
    metadata:
      labels:
        app: myboot2
    spec:
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - topologyKey: kubernetes.io/hostname
            labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - myboot
      containers:
      - name: myboot
        image: nginx
        ports:
          - containerPort: 8080
```


```
kubectl apply -f apps/kubefiles/myboot-pod-affinity.yml

kubectl get pods
```

```
NAME                                                              READY   STATUS    RESTARTS   AGE
myboot2-784bc58c8d-j2l74                                          0/1     Pending   0          19s
```

Le Pod myboot2 est en attente car il n'a pas pu trouver de Pod correspondant à la règle d'affinité. Déployons l'application myboot étiquetée avec app=myboot.

```
kubectl apply -f apps/kubefiles/myboot-nginx-deployment.yml

kubectl get pods
```

```
NAME                                                              READY   STATUS    RESTARTS   AGE
myboot-7f889dd6d-tr7gr                                            1/1     Running   0          3m27s
myboot2-64566b697b-snm7p                                          1/1     Running   0          18s
```

Maintenant, les deux applications sont exécutées dans le même nœud :

```
kubectl get pod myboot-7f889dd6d-tr7gr -o json | jq '.spec.nodeName'
```

```
minikube
```

```
kubectl get pod myboot2-64566b697b-snm7p -o json | jq '.spec.nodeName'
```

```
minikube
```

Ce que vous avez vu ici est une règle stricte, vous pouvez également utiliser des règles "douces" dans Pod Affinity.

```
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        podAffinityTerm:
          topologyKey: kubernetes.io/hostname
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - myboot
```

L'anti-affinité est utilisée pour s'assurer que deux pods ne fonctionnent PAS ensemble sur le même nœud.

```
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - topologyKey: kubernetes.io/hostname
        labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - myboot
```

Déployer un myboot3 avec une règle d'anti-affinité


```
vim apps/kubefiles/myboot-pod-antiaffinity.yaml
```


```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: myboot3
  name: myboot3
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myboot3
  template:
    metadata:
      labels:
        app: myboot3
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - topologyKey: kubernetes.io/hostname
            labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - myboot
      containers:
      - name: myboot
        image: nginx
        ports:
          - containerPort: 8080
```

```
kubectl apply -f apps/kubefiles/myboot-pod-antiaffinity.yaml
```

Puis utilisez la commande kubectl get pods -o wide pour voir quels pods atterrissent sur quels noeuds.

```
kubectl get pods -o wide
```

```
NAME                       READY   STATUS    RESTARTS   AGE    IP          NODE
myboot-7f889dd6d-tr7gz     1/1     Running   0          4m27s  10.88.0.9   minikube
myboot2-64566b697b-snm7p   1/1     Running   0          48s    10.88.0.10  minikube
myboot3-78656b637r-suy1t   1/1     Running   0          1s     172.17.0.2  minikube-m02
```

Le pod myboot3 est déployé dans un nœud différent de celui du pod myboot.

Supprimer les ressources

```
kubectl delete -f apps/kubefiles/myboot-pod-affinity.yml
kubectl delete -f apps/kubefiles/myboot-pod-antiaffinity.yml
kubectl delete -f apps/kubefiles/myboot-deployment.yml
```
