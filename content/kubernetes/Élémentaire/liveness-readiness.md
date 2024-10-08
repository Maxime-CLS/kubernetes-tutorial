---
title: "Liveness & Readiness"
date: 2020-06-26T15:17:20+02:00
draft: false
weight: 15
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

Assurez-vous que rien d'autre n'est déployé :

```
kubectl get all
```

```
No resources found in myspace namespace.
```

Déployez une application avec le jeu de sondes Live and Ready :

```
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myboot
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myboot
  template:
    metadata:
      labels:
        app: myboot
        env: dev
    spec:
      containers:
      - name: myboot
        image: quay.io/rhdevelopers/myboot:v1
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "300Mi"
            cpu: "250m" # 1/4 core
          limits:
            memory: "800Mi"
            cpu: "1000m" # 1 core
        livenessProbe:
          httpGet:
              port: 8080
              path: /
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 2
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 3
EOF
```

Décrivez le déploiement :

```
kubectl describe deployment myboot
```

```
...
    Image:      quay.io/rhdevelopers/myboot:v1
    Port:       8080/TCP
    Host Port:  0/TCP
    Limits:
      cpu:     1
      memory:  400Mi
    Requests:
      cpu:        250m
      memory:     300Mi
    Liveness:     http-get http://:8080/ delay=10s timeout=2s period=5s #success=1 #failure=3
    Readiness:    http-get http://:8080/health delay=10s timeout=1s period=3s #success=1 #failure=3
...
```

Déployer un service :


```
kubectl apply -f apps/kubefiles/myboot-service.yml
```

Créer les variables IP et PORT

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

Changez l'image :

```
kubectl set image deployment/myboot myboot=quay.io/rhdevelopers/myboot:v2
```

Et remarquez la mise à jour sans erreur :

```
Aloha from Spring Boot! 131 on myboot-845968c6ff-k4rvb
Aloha from Spring Boot! 134 on myboot-845968c6ff-9wvt9
Aloha from Spring Boot! 122 on myboot-845968c6ff-9824z
Bonjour from Spring Boot! 0 on myboot-8449d5468d-m88z4
Bonjour from Spring Boot! 1 on myboot-8449d5468d-m88z4
Aloha from Spring Boot! 135 on myboot-845968c6ff-9wvt9
Aloha from Spring Boot! 133 on myboot-845968c6ff-k4rvb
Aloha from Spring Boot! 137 on myboot-845968c6ff-9wvt9
Bonjour from Spring Boot! 3 on myboot-8449d5468d-m88z4
```

Regardez les points d'extrémité pour voir quels pods font partie du service :

```
kubectl get endpoints myboot -o json | jq '.subsets[].addresses[].ip'
```

Ce sont les IP des Pods qui ont passé leur test de préparation :

```
"10.129.2.40"
"10.130.2.37"
"10.130.2.38"
```

### Readiness Probe

Exec en un seul Pod et changer son indicateur de disponibilité :

```
kubectl exec -it myboot-845968c6ff-k5lcb /bin/bash
```

```
curl localhost:8080/misbehave
exit
```

Vérifiez que le pod n'est plus prêt :

```
NAME                      READY   STATUS    RESTARTS   AGE
myboot-845968c6ff-9wshg   1/1     Running   0          11m
myboot-845968c6ff-k5lcb   0/1     Running   0          12m
myboot-845968c6ff-zsgx2   1/1     Running   0          11m
```

Maintenant, vérifiez les points de terminaison :

```
kubectl get endpoints myboot -o json | jq '.subsets[].addresses[].ip'
```

Et ce pod est maintenant absent de l'équilibreur de charge du service :

```
"10.130.2.37"
"10.130.2.38"
```

Ce qui est aussi une évidence dans la boucle de la boucle :

```
Aloha from Spring Boot! 845 on myboot-845968c6ff-9wshg
Aloha from Spring Boot! 604 on myboot-845968c6ff-zsgx2
Aloha from Spring Boot! 846 on myboot-845968c6ff-9wshg
```

### Liveness Probe

```
kubectl set image deployment/myboot myboot=quay.io/rhdevelopers/myboot:v3
```

Laissez le déploiement se terminer sur les 3 répliques :

```
watch kubectl get pods
NAME                      READY   STATUS    RESTARTS   AGE
myboot-56659c9d69-6sglj   1/1     Running   0          2m2s
myboot-56659c9d69-mdllq   1/1     Running   0          97s
myboot-56659c9d69-zjt6q   1/1     Running   0          72s
```

Et comme on le voit dans la boucle de curl/poller :

```
Jambo from Spring Boot! 40 on myboot-56659c9d69-mdllq
Jambo from Spring Boot! 26 on myboot-56659c9d69-zjt6q
Jambo from Spring Boot! 71 on myboot-56659c9d69-6sglj
```

Modifiez le déploiement pour qu'il pointe vers l'URL /alive :

```
kubectl edit deployment myboot
```

Et changez la sonde de Liveness probe :

```
...
    spec:
      containers:
      - image: quay.io/rhdevelopers/myboot:v3
        imagePullPolicy: Always
        livenessProbe:
          failureThreshold: 3
          httpGet:
            path: /alive
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 10
          periodSeconds: 5
          successThreshold: 1
          timeoutSeconds: 2
        name: myboot
...
```

Sauvegardez et fermez l'éditeur, ce qui permet à cette modification d'être appliquée.

```
watch kubectl get pods
```

```
NAME                      READY   STATUS        RESTARTS   AGE
myboot-558b4f8678-nw762   1/1     Running       0          59s
myboot-558b4f8678-qbrgc   1/1     Running       0          81s
myboot-558b4f8678-z7f9n   1/1     Running       0          36s
```

Maintenant, choisissez un pod, exécutez-la et tirez dessus :

```
kubectl exec -it myboot-558b4f8678-qbrgc /bin/bash
```

```
curl localhost:8080/shot
```

Et vous verrez qu'il sera redémarré :

```
NAME                      READY   STATUS    RESTARTS   AGE
myboot-558b4f8678-nw762   1/1     Running   0          4m7s
myboot-558b4f8678-qbrgc   1/1     Running   1          4m29s
myboot-558b4f8678-z7f9n   1/1     Running   0          3m44s
```

De plus, votre exécution sera terminée :

```
kubectl exec -it myboot-558b4f8678-qbrgc /bin/bash
```

```
curl localhost:8080/shot
```

```
I have been shot in the head1000610000@myboot-558b4f8678-qbrgc:/app$ command terminated with exit code 137
```


Et vos utilisateurs finaux ne verront pas d'erreurs :

```
Jambo from Spring Boot! 174 on myboot-558b4f8678-z7f9n
Jambo from Spring Boot! 11 on myboot-558b4f8678-qbrgc
Jambo from Spring Boot! 12 on myboot-558b4f8678-qbrgc
Jambo from Spring Boot! 206 on myboot-558b4f8678-nw762
Jambo from Spring Boot! 207 on myboot-558b4f8678-nw762
Jambo from Spring Boot! 175 on myboot-558b4f8678-z7f9n
Jambo from Spring Boot! 176 on myboot-558b4f8678-z7f9n
```

Supprimer les ressources

```
kubectl delete deployment myboot
kubectl delete service myboot
```

## Startup Probe

Certaines applications nécessitent un temps de démarrage supplémentaire lors de leur première initialisation.

Il peut s'avérer difficile d'intégrer ce scénario dans les sondes de disponibilité/préparation car vous devez les configurer pour qu'elles aient un comportement normal afin de détecter les anomalies pendant le temps d'exécution et, en outre, pour couvrir le long temps de démarrage.

Les sondes de démarrage résolvent ce problème, car une fois que la sonde de démarrage a réussi, le reste des sondes prend le relais.

```
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
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
        env: dev
    spec:
      containers:
      - name: myboot
        image: quay.io/rhdevelopers/myboot:zoombie
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "300Mi"
            cpu: "250m" # 1/4 core
          limits:
            memory: "400Mi"
            cpu: "1000m" # 1 core
        livenessProbe:
          httpGet:
              port: 8080
              path: /
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 2
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 3
        startupProbe:
          httpGet:
            path: /alive
            port: 8080
          failureThreshold: 12
          periodSeconds: 5
EOF
```

La sonde de démarrage attend une minute (5 * 12) pour démarrer l'application.

L'application actuelle renvoie une erreur 503 dans le endpoint /alive, elle ne réussira donc jamais à démarrer et la restartPolicy est appliquée.

```
watch kubectl get pods
```

```
NAME                      READY   STATUS    RESTARTS   AGE
myboot-579cc5cc47-2bk5p   0/1     Running   0          67s
```

Attendez 60 secondes jusqu'à ce que vous voyiez que le pod est redémarré.

Et vous verrez qu'il a redémarré :

```
NAME                      READY   STATUS    RESTARTS   AGE
myboot-579cc5cc47-2bk5p   0/1     Running   1          3m7s
```
Pour que le pod réussisse, exécutez-la et faites-la renaître :

```
kubectl exec -it myboot-579cc5cc47-2bk5p /bin/bash
```

```
curl localhost:8080/reborn
```

Et enfin, c'est parti :

```
NAME                      READY   STATUS    RESTARTS   AGE
myboot-579cc5cc47-2bk5p   1/1     Running   1          3m41s
```

Décrire le pod pour obtenir les statistiques des sondes :

```
kubectl describe pod myboot-579cc5cc47-2bk5p
```
```
Limits:
  cpu:     1
  memory:  400Mi
Requests:
  cpu:        250m
  memory:     300Mi
Liveness:     http-get http://:8080/ delay=10s timeout=2s period=5s #success=1 #failure=3
Readiness:    http-get http://:8080/health delay=10s timeout=1s period=3s #success=1 #failure=3
Startup:      http-get http://:8080/alive delay=0s timeout=1s period=5s #success=1 #failure=12
Environment:  <none>
Mounts:
```

Supprimer les ressources

```
kubectl delete deployment myboot
```


### A vous de jouez !

Créer un déploiement avec une ReadinessProbe

Créez un déploiement nommé space-alien-welcome-message-generator d'image httpd:alpine avec 1 replica.

Il devrait avoir un ReadinessProbe qui exécute la commande stat /tmp/ready. Cela signifie qu'une fois que le fichier existe, le pod doit être prêt.

Les valeurs initialDelaySeconds et periodSeconds doivent être respectivement de 10 et 5.

Créez le déploiement et observez que le module n'est pas prêt.

