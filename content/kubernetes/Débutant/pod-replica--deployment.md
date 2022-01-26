---
title: "Pod, Replicaset, Deployment"
date: 2020-06-26T15:17:20+02:00
draft: false
weight: 10
---

## Prérequis

- Minikube [Install](https://kubernetes.io/fr/docs/tasks/tools/install-minikube/#installez-minikube-par-t%C3%A9l%C3%A9chargement-direct)  [Driver none](https://kubernetes.io/docs/setup/learning-environment/minikube/#specifying-the-vm-driver)
- kubectl [Install](https://kubernetes.io/fr/docs/tasks/tools/install-kubectl/)
- Stern [Docs](https://kubernetes.io/blog/2016/10/tail-kubernetes-with-stern/) [Release](https://github.com/wercker/stern/releases)
- jq [Install](https://stedolan.github.io/jq/download/)
- 3 terminal SSH

Commencez par créer un espace de noms dans lequel vous pourrez travailler :

```
kubectl create namespace myspace
kubectl config set-context --current --namespace=myspace
```

### Pod

Créer un [naked pod](https://kubernetes.io/docs/concepts/configuration/overview/#naked-pods-vs-replicasets-deployments-and-jobs) :

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: quarkus-demo
spec:
  containers:
  - name: quarkus-demo
    image: quay.io/rhdevelopers/quarkus-demo:v1
EOF
```

Observez le cycle de vie du pod :

terminal 2 :

```
watch kubectl get pods
```

```
NAME           READY   STATUS              RESTARTS   AGE
quarkus-demo   0/1     ContainerCreating   0          10s
```

De la création de conteneurs à l'exécution avec Ready 1/1 :

```
NAME           READY   STATUS    RESTARTS   AGE
quarkus-demo   1/1     Running   0          18s
```

Vérifiez la demande dans le Pod :

```
kubectl exec -it quarkus-demo /bin/sh
```

Exécutez la commande suivante. Notez que, comme vous êtes dans l'instance du conteneur, le nom d'hôte est localhost.

```
curl localhost:8080
```

```
Supersonic Subatomic Java with Quarkus quarkus-demo:1
exit
```

Supprimons le Pod précédent :

```
kubectl delete pod quarkus-demo
```

terminal 2 

```
watch kubectl get pods
```
```
NAME           READY   STATUS        RESTARTS   AGE
quarkus-demo   0/1     Terminating   0          9m35s

No resources found in myspace namespace.
```

Le pod Naked disparaît à jamais.

   
Un pod naked est un pod nue et il ne sera pas replanifier en cas d'erreur sur le pod ou de suppression.
    

### Replicaset

Créer un replicaset :

```
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: ReplicaSet
metadata:
    name: rs-quarkus-demo
spec:
    replicas: 3
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
EOF
```

Obtenez les pods avec des étiquettes :

terminal 2 

```
watch kubectl get pods --show-labels
```

```
NAME                    READY   STATUS    RESTARTS   AGE   LABELS
rs-quarkus-demo-jd6jk   1/1     Running   0          58s   app=quarkus-demo,env=dev
rs-quarkus-demo-mlnng   1/1     Running   0          58s   app=quarkus-demo,env=dev
rs-quarkus-demo-t26gt   1/1     Running   0          58s   app=quarkus-demo,env=dev
```

Afficher les replicaset créé :

```
kubectl get rs
```

```
NAME              DESIRED   CURRENT   READY   AGE
rs-quarkus-demo   3         3         3       79s
```

Décrire le replicaset 

```
kubectl describe rs rs-quarkus-demo
```

Les pods sont la "propriété" du ReplicaSet :

```
kubectl get pod rs-quarkus-demo-mlnng -o json | jq ".metadata.ownerReferences[]"
```

```
{
  "apiVersion": "apps/v1",
  "blockOwnerDeletion": true,
  "controller": true,
  "kind": "ReplicaSet",
  "name": "rs-quarkus-demo",
  "uid": "1ed3bb94-dfa5-40ef-8f32-fbc9cf265324"
}
```

Supprimez maintenant un pod, tout en regardant les pods :

```
kubectl delete pod rs-quarkus-demo-mlnng
```

Et une nouvelle pod va naître pour le remplacer :

```
NAME                    READY   STATUS              RESTARTS   AGE    LABELS
rs-quarkus-demo-2txwk   0/1     ContainerCreating   0          2s     app=quarkus-demo,env=dev
rs-quarkus-demo-jd6jk   1/1     Running             0          109s   app=quarkus-demo,env=dev
rs-quarkus-demo-t26gt   1/1     Running             0          109s   app=quarkus-demo,env=dev
```

Supprimez le ReplicaSet pour supprimer tous les pods associés :

```
kubectl delete rs rs-quarkus-demo
```

### Deploiement

Créer un déploiement de 3 replicaset d'un même conteneur.

```
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: quarkus-demo-deployment
spec:
  replicas: 3
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

```
kubectl get pods --show-labels
```

```
NAME                                       READY   STATUS    RESTARTS   AGE   LABELS
quarkus-demo-deployment-5979886fb7-c888m   1/1     Running   0          17s   app=quarkus-demo,env=dev,pod-template-hash=5979886fb7
quarkus-demo-deployment-5979886fb7-gdtnz   1/1     Running   0          17s   app=quarkus-demo,env=dev,pod-template-hash=5979886fb7
quarkus-demo-deployment-5979886fb7-grf59   1/1     Running   0          17s   app=quarkus-demo,env=dev,pod-template-hash=5979886f
```

Executer une commande shell dans le pods :

```
kubectl exec -it quarkus-demo-deployment-5979886fb7-c888m -- curl localhost:8080
```

```
Supersonic Subatomic Java with Quarkus quarkus-demo-deployment-5979886fb7-c888m:1
```