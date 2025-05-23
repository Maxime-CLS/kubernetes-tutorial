---
title: "Pod, Replicaset, Deployment"
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

### A vous de jouez !

#### Défi 1 : 

Créer un pod avec des Resource Requests et Limits

Créer un namespace de noms limit.  

Dans ce Namespace, créez un Pod nommé resource-checker de l'image httpd:alpine.  

Le conteneur doit être nommé my-container.  

Il doit demander 30m de CPU et être limité à 300m de CPU.

Il doit demander 30Mi de mémoire et être limité à 30Mi de mémoire.


#### Défi 2 :

Déployez cette application :

```
vi deployement-fix.yaml
```

```
---
apiVersion: v1
kind: Service
metadata:
  annotations:
    app.quarkus.io/commit-id: 91d4fef5457795ed2a1a38daeeaee4837254b390
    app.quarkus.io/build-timestamp: 2022-05-30 - 12:42:01 +0000
  labels:
    app.kubernetes.io/name: hello-fix
    app.kubernetes.io/version: 2.0.0
  name: hello-fix
spec:
  ports:
    - name: http
      port: 80
      targetPort: 9090
  selector:
    app.kubernetes.io/name: hello-fix
    app.kubernetes.io/version: 2.0.0
  type: LoadBalancer
---
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    app.quarkus.io/commit-id: 91d4fef5457795ed2a1a38daeeaee4837254b390
    app.quarkus.io/build-timestamp: 2022-05-30 - 12:42:01 +0000
  labels:
    app.kubernetes.io/version: 2.0.0
    app.kubernetes.io/name: hello-fix
  name: hello-fix
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/version: 2.0.0
      app.kubernetes.io/name: hello-fix
  template:
    metadata:
      annotations:
        app.quarkus.io/commit-id: 91d4fef5457795ed2a1a38daeeaee4837254b390
        app.quarkus.io/build-timestamp: 2022-05-30 - 12:42:01 +0000
      labels:
        app.kubernetes.io/version: 2.0.0
        app.kubernetes.io/name: hello-fix
    spec:
      containers:
        - env:
            - name: KUBERNETES_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          image: quay.io/rhdevelopers/hello-fix:2.0.0
          imagePullPolicy: Always
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /q/health/live
              port: 8080
              scheme: HTTP
            initialDelaySeconds: 0
            periodSeconds: 30
            successThreshold: 1
            timeoutSeconds: 10
          name: hello-fix
          ports:
            - containerPort: 8080
              name: http
              protocol: TCP
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /q/health/ready
              port: 8080
              scheme: HTTP
            initialDelaySeconds: 0
            periodSeconds: 30
            successThreshold: 1
            timeoutSeconds: 10
      serviceAccountName: hello-fix
```


```
kubectl apply -f deployement-fix.yaml
```

Lancez maintenant une commande pour vérifier l'état des pods :

```
kubectl get pods 
```

Le résultat devrait être similaire à :

```
No resources found in default namespace.
```

Vous devez trouver pourquoi il échoue et corriger le déploiement pour être en état READY et obtenir le résultat suivant : 

```
NAME                        READY   STATUS    RESTARTS   AGE
hello-fix-8787bd4fc-lfqj7   1/1     Running   0          17s
```

> [!INFO]
> Pensez a supprimer le déploiement avant d'appliquer de nouvelle modification avec la commande kubectl delete -f deployement-fix.yaml
