---
title: "Configmap"
date: 2020-06-26T15:17:20+02:00
draft: false
weight: 20
---

## Prérequis

- Minikube [Install](https://kubernetes.io/fr/docs/tasks/tools/install-minikube/#installez-minikube-par-t%C3%A9l%C3%A9chargement-direct)  [Driver none](https://kubernetes.io/docs/setup/learning-environment/minikube/#specifying-the-vm-driver)
- kubectl [Install](https://kubernetes.io/fr/docs/tasks/tools/install-kubectl/)
- Stern [Docs](https://kubernetes.io/blog/2016/10/tail-kubernetes-with-stern/) [Release](https://github.com/wercker/stern/releases)
- jq [Install](https://stedolan.github.io/jq/download/)
- 3 terminal SSH


ConfigMap est la ressource Kubernetes qui vous permet d'externaliser la configuration de votre application.

La configuration d'une application est tout ce qui est susceptible de varier entre les déploiements (staging, production, environnements de développement, etc).

[The Twelve-Factor App](https://12factor.net/config)

Variables d'environnement

MyRESTController.java comprend un petit morceau de code qui s'adresse à l'environnement :

```
@RequestMapping("/configure")
   public String configure() {
        String databaseConn = environment.getProperty("DBCONN","Default");
        String msgBroker = environment.getProperty("MSGBROKER","Default");
        String hello = environment.getProperty("GREETING","Default");
        String love = environment.getProperty("LOVE","Default");
        return "Configuration: \n"
            + "databaseConn=" + databaseConn + "\n"
            + "msgBroker=" + msgBroker + "\n"
            + "hello=" + hello + "\n"
            + "love=" + love + "\n";
   }
```

Les variables d'environnement peuvent être manipulées au niveau du déploiement. Les modifications entraînent le redéploiement du Pod.

Déploiement de myboot :

Créer un fichier de déploiement

```
vi apps/kubefiles/myboot-deployment.yml
```

myboot-deployment.yml

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
        image: quay.io/rhdevelopers/myboot:v2
        ports:
          - containerPort: 8080
```

```
kubectl apply -f apps/kubefiles/myboot-deployment.yml
```

Déployer le service myboot :

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

```
kubectl apply -f apps/kubefiles/myboot-service.yml
```

Et regardez l'état du pod :

```
watch kubectl get pods
```

Créer les variables IP et PORT :

```
IP=$(minikube ip -p devnation)
PORT=$(kubectl get service/myboot -o jsonpath="{.spec.ports[*].nodePort}")
```

Réaliser une requete sur le service :

```
curl $IP:$PORT
```

```
curl $IP:$PORT/configure
```

```
Configuration for : myboot-66d7d57687-jsbz7
databaseConn=Default
msgBroker=Default
greeting=Default
love=Default
```

Définir les variables d'environnement

```
kubectl set env deployment/myboot GREETING="namaste"
kubectl set env deployment/myboot LOVE="Aloha"
kubectl set env deployment/myboot DBCONN="jdbc:sqlserver://45.91.12.123:1443;user=MyUserName;password=*****;"
```

Regardez les pods reborn :

```
NAME                      READY   STATUS        RESTARTS   AGE
myboot-66d7d57687-jsbz7   1/1     Terminating   0          5m
myboot-785ff6bddc-ghwpc   1/1     Running       0          13s
```

```
curl $IP:$PORT/configure
```

```
Configuration for : myboot-5fd9dd9c59-58xbh
databaseConn=jdbc:sqlserver://45.91.12.123:1443;user=MyUserName;password=*****;
msgBroker=Default
greeting=namaste
love=Aloha
```

Décrivez le déploiement :

```
kubectl describe deployment myboot
```

```
...
  Containers:
   myboot:
    Image:      quay.io/burrsutter/myboot:v1
    Port:       8080/TCP
    Host Port:  0/TCP
    Environment:
      GREETING:  namaste
      LOVE:      Aloha
      DBCONN:    jdbc:sqlserver://45.91.12.123:1443;user=MyUserName;password=*****;
    Mounts:      <none>
  Volumes:       <none>
...
```

Supprimez les variables d'environnement :

```
kubectl set env deployment/myboot GREETING-
kubectl set env deployment/myboot LOVE-
kubectl set env deployment/myboot DBCONN-
```

Et vérifiez qu'ils ont été retirés :

```
curl $IP:$PORT/configure
```

```
Configuration for : myboot-66d7d57687-xkgw6
databaseConn=Default
msgBroker=Default
greeting=Default
love=Default
```

Créer un ConfigMap

```
kubectl create cm my-config --from-env-file=apps/config/some.properties
```

```
kubectl get cm
kubectl get cm my-config
kubectl get cm my-config -o json
```

```
...
    "data": {
        "GREETING": "jambo",
        "LOVE": "Amour"
    },
    "kind": "ConfigMap",
...
```

Ou vous pouvez décrire l'objet ConfigMap :


```
kubectl describe cm my-config
```

```
Name:         my-config
Namespace:    myspace
Labels:       <none>
Annotations:  <none>

Data
====
GREETING:
====
jambo
LOVE:
====
Amour
Events:  <none>

```

Maintenant, déployez l'application avec sa requête pour le ConfigMap :

```
kubectl apply -f apps/kubefiles/myboot-deployment-configuration.yml
```

Et obtenir son point de terminaison de configuration :

```
curl $IP:$PORT/configure
```

```
Configuration for : myboot-84bfcff474-x6xnt
databaseConn=Default
msgBroker=Default
greeting=jambo
love=Amour
```

Et passez à l'autre fichier de propriétés en recréant le ConfigMap :

```
kubectl delete cm my-config
kubectl create cm my-config --from-env-file=apps/config/other.properties
kubectl delete pod -l app=myboot
```

```
curl $IP:$PORT/configure
```

```
Configuration for : myboot-694954fc6d-nzdvx
databaseConn=jdbc:sqlserver://123.123.123.123:1443;user=MyUserName;password=*****;
msgBroker=tcp://localhost:61616?jms.useAsyncSend=true
hello=Default
love=Default
```

Il y a beaucoup d'autres façons de s'amuser avec ConfigMaps, la documentation de base vous fait manipuler une spécification Pod au lieu d'un déploiement mais les résultats sont fondamentalement les mêmes https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap.

Supprimer les ressources

```
kubectl delete deployment myboot
kubectl delete cm my-config
kubectl delete service myboot
```

### A vous de jouez !

Votre objectif est de créer un ConfigMap !

Ce ConfigMap doit avoir :
  - Un nom de la ressource égale à trauerweide
  - La configuration doit contenir : tree=trauerweide


