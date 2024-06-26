---
title: "Logs"
date: 2020-06-26T15:17:20+02:00
draft: false
weight: 20
---

## Prérequis

- Minikube [Install](https://kubernetes.io/fr/docs/tasks/tools/install-minikube/#installez-minikube-par-t%C3%A9l%C3%A9chargement-direct)  [Driver none](https://kubernetes.io/docs/setup/learning-environment/minikube/#specifying-the-vm-driver)
- kubectl [Install](https://kubernetes.io/fr/docs/tasks/tools/install-kubectl/)
- Stern [Docs](https://kubernetes.io/blog/2016/10/tail-kubernetes-with-stern/) [Release](https://github.com/stern/stern/releases)
- jq [Install](https://stedolan.github.io/jq/download/)
- 3 terminal SSH


Il existe plusieurs façons "prêtes à la production" de collecter et de visualiser les messages de logs dans un cluster Kubernetes. Beaucoup de gens aiment certaines fonctionnalités de ELK (ElasticSearch, Logstash, Kibana) ou EFK (ElasticSearch, FluentD, Kibana).

L'accent est mis ici sur les éléments auxquels un développeur doit avoir accès pour l'aider à comprendre le comportement de son application s'exécutant à l'intérieur d'un pod.

Assurez-vous qu'une application (Deployment) est en cours d'exécution :

```
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
        env: dev
    spec:
      containers:
      - name: myapp
        image: quay.io/rhdevelopers/myboot:v1
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
EOF
```

Assurez-vous que vous utilisez 3 répliques (3 pods/instances de votre demande) :

```
kubectl get deployment my-deployment -o json | jq '.status.replicas'
```

If not, scale up to 3:

```
kubectl scale --replicas=3 deployment/my-deployment
```

```
NAME                             READY   STATUS    RESTARTS   AGE
my-deployment-5dc67997c7-5bq4n   1/1     Running   0          34s
my-deployment-5dc67997c7-m7z9f   1/1     Running   0          34s
my-deployment-5dc67997c7-s4jc6   1/1     Running   0          34s
```

```
kubectl logs my-deployment-5dc67997c7-m7z9f
```
```
 .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v1.5.3.RELEASE)
```

Vous pouvez suivre les logs avec le paramètre -f :

```
kubectl logs my-deployment-5dc67997c7-m7z9f -f
```

terminal 2 :

```
kubectl exec -it my-deployment-5dc67997c7-m7z9f /bin/bash
curl localhost:8080
```

```
Aloha from my-deployment-5dc67997c7-m7z9f
```

Deploy a Service for my-deployment:

```
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: the-service
spec:
  selector:
    app: myapp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: LoadBalancer
EOF
```

terminal 2:

```
IP=$(minikube ip)
PORT=$(kubectl get service/the-service -o jsonpath="{.spec.ports[*].nodePort}")
```

Curl the Service: terminal 2:

```
curl $IP:$PORT
```

Commencez à envoyer la demande en boucle : terminal 2:

```
while true
do curl $IP:$PORT
sleep .3
done
```

Utilisez ensuite Stern pour visualiser les logs de tous les pods :

```
stern my-deployment
```

```
my-deployment-5dc67997c7-5bq4n myapp Aloha from my-deployment-5dc67997c7-5bq4n
my-deployment-5dc67997c7-m7z9f myapp Aloha from my-deployment-5dc67997c7-m7z9f
my-deployment-5dc67997c7-s4jc6 myapp Aloha from my-deployment-5dc67997c7-s4jc6
my-deployment-5dc67997c7-s4jc6 myapp Aloha from my-deployment-5dc67997c7-s4jc6
```

Nettoyer l'environnement 

```
kubectl delete service the-service
kubectl delete deployment my-deployment
```