---
title: "Secret"
date: 2020-06-26T15:17:20+02:00
draft: false
weight: 5
---

## Prérequis

- Minikube [Install](https://kubernetes.io/fr/docs/tasks/tools/install-minikube/#installez-minikube-par-t%C3%A9l%C3%A9chargement-direct)  [Driver none](https://kubernetes.io/docs/setup/learning-environment/minikube/#specifying-the-vm-driver)
- kubectl [Install](https://kubernetes.io/fr/docs/tasks/tools/install-kubectl/)
- Stern [Docs](https://kubernetes.io/blog/2016/10/tail-kubernetes-with-stern/) [Release](https://github.com/stern/stern/releases)
- jq [Install](https://stedolan.github.io/jq/download/)
- 3 terminal SSH


Déployer le service myboot :

```
kubectl apply -f apps/kubefiles/myboot-deployment.yml
```

Déployer le service myboot :

```
kubectl apply -f apps/kubefiles/myboot-service.yml
```

Regardez vos Pods:

```
watch kubectl get pods
```

Regardez vos services:

```
watch kubectl get services
```

```
NAME    TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
myapp   LoadBalancer   172.30.103.41   <pending>     8080:31974/TCP   4s
```

Attendez jusqu'à ce que vous voyez une IP externe assignée.

```
NAME    TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)          AGE
myapp   LoadBalancer   172.30.103.41   34.71.122.153   8080:31974/TCP   44s
```

Créer les variables IP et PORT :

```
IP=$(minikube ip)
PORT=$(kubectl get service/myboot -o jsonpath="{.spec.ports[*].nodePort}")
```

Réaliser une requete du service :

```
curl $IP:$PORT
```

L'exemple de ConfigMap présenté précédemment contenait une chaîne de connexion à une base de données ("user=MyUserName;password=*"). Les données sensibles comme les mots de passe peuvent être placées dans un autre récipient appelé Secret.


## Créer des secrets

```
kubectl create secret generic mysecret --from-literal=user='MyUserName' --from-literal=password='mypassword'
```
```
kubectl get secrets
```

```
NAME                       TYPE                                  DATA   AGE
builder-dockercfg-96ml5    kubernetes.io/dockercfg               1      3d6h
builder-token-h5g82        kubernetes.io/service-account-token   4      3d6h
builder-token-vqjqz        kubernetes.io/service-account-token   4      3d6h
default-dockercfg-bsnjr    kubernetes.io/dockercfg               1      3d6h
default-token-bl77s        kubernetes.io/service-account-token   4      3d6h
default-token-vlzsl        kubernetes.io/service-account-token   4      3d6h
deployer-dockercfg-k6npn   kubernetes.io/dockercfg               1      3d6h
deployer-token-4hb78       kubernetes.io/service-account-token   4      3d6h
deployer-token-vvh6r       kubernetes.io/service-account-token   4      3d6h
mysecret                   Opaque                                2      5s
```

L'utilisateur et le mot de passe ne sont pas immédiatement visibles :

```
Name:         mysecret
Namespace:    myspace
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
password:  10 bytes
user:      10 bytes
```

```
apiVersion: v1
data:
  password: bXlwYXNzd29yZA==
  user: TXlVc2VyTmFtZQ==
kind: Secret
metadata:
  creationTimestamp: "2020-03-31T20:19:26Z"
  name: mysecret
  namespace: myspace
  resourceVersion: "4944690"
  selfLink: /api/v1/namespaces/myspace/secrets/mysecret
  uid: e8c5f12e-bd71-4d6b-8d8c-7af9ed6439f8
type: Opaque
```
Vous pouvez voir les secrets en courant :

```
echo 'bXlwYXNzd29yZA==' | base64 --decode
```

```
mypassword
```

```
echo 'TXlVc2VyTmFtZQ==' | base64 --decode
```

```
MyUserName
```

Ou les obtenir en utilisant kubectl :

```
kubectl get secret mysecret -o jsonpath='{.data.password}' | base64 --decode
```

Les secrets sont fournis au Pod via des montages de volumes :

```
        volumeMounts:
          - name: mysecretvolume
            mountPath: /mystuff/mysecretvolume
```

Nouveau déploiement avec le volume secret :

```
vim apps/kubefiles/myboot-deployment-configuration-secret.yml
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
        image: quay.io/rhdevelopers/myboot:v1
        ports:
          - containerPort: 8080
        volumeMounts:
          - name: mysecretvolume #<.>
            mountPath: /mystuff/secretstuff
            readOnly: true
        resources:
          requests:
            memory: "300Mi"
            cpu: "250m" # 1/4 core
          limits:
            memory: "400Mi"
            cpu: "1000m" # 1 core
      volumes:
        - name: mysecretvolume #<.>
          secret:
            secretName: mysecret
```

```
kubectl replace -f apps/kubefiles/myboot-deployment-configuration-secret.yml
```

Exec dans le Pod nouvellement créé :

```
PODNAME=$(kubectl get pod -l app=myboot -o name)
kubectl exec $PODNAME -- cat /mystuff/secretstuff/password
```

Résultat

```
mypassword
```

Vous pourriez fournir l'emplacement de /mystuff/mysecretvolume au pod via une variable d'environnement afin que l'application sache où chercher.


Supprimer vos ressources

```
kubectl delete deployment myboot
kubectl delete service myboot
```


### A vous de jouez !

CreateContainerConfigError est une erreur d'exécution qui apparaît lorsque le conteneur est incapable de démarrer, avant même que l'application à l'intérieur du conteneur ne démarre.

Déployez cette application :

```
---
apiVersion: v1
kind: Service
metadata:
  annotations:
    app.quarkus.io/commit-id: 91d4fef5457795ed2a1a38daeeaee4837254b390
    app.quarkus.io/build-timestamp: 2022-05-27 - 12:35:51 +0000
  labels:
    app.kubernetes.io/name: hello-fix
    app.kubernetes.io/version: 1.0.0
  name: hello-fix
spec:
  ports:
    - name: http
      port: 80
      targetPort: 8080
  selector:
    app.kubernetes.io/name: hello-fix
    app.kubernetes.io/version: 1.0.0
  type: LoadBalancer
---
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    app.quarkus.io/commit-id: 91d4fef5457795ed2a1a38daeeaee4837254b390
    app.quarkus.io/build-timestamp: 2022-05-27 - 12:35:51 +0000
  labels:
    app.kubernetes.io/version: 1.0.0
    app.kubernetes.io/name: hello-fix
  name: hello-fix
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/version: 1.0.0
      app.kubernetes.io/name: hello-fix
  template:
    metadata:
      annotations:
        app.quarkus.io/commit-id: 91d4fef5457795ed2a1a38daeeaee4837254b390
        app.quarkus.io/build-timestamp: 2022-05-27 - 12:35:51 +0000
      labels:
        app.kubernetes.io/version: 1.0.0
        app.kubernetes.io/name: hello-fix
    spec:
      containers:
        - env:
            - name: KUBERNETES_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          envFrom:
            - secretRef:
                name: maria
          image: quay.io/rhdevelopers/hello-fix:1.0.0
          imagePullPolicy: Always
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /q/health/live
              port: 8080
              scheme: HTTP
            initialDelaySeconds: 0
            periodSeconds: 5
            successThreshold: 1
            timeoutSeconds: 10
          name: hello-fix
          ports:
            - containerPort: 8080
              name: http
              protocol: TCP
---
apiVersion: v1
kind: Secret
metadata:
  name: mariadb
type: Opaque
```

Une fois déployé, veuillez vérifier l'état du pod en utilisant la commande :

```
kubectl get pods
```

Le résultat serait similaire à :

```
NAME                         READY   STATUS                       RESTARTS   AGE
hello-fix-7c5fffc8c8-j2h8g   0/1     CreateContainerConfigError   0          20s
```

Vous devez trouver pourquoi il échoue et corriger le déploiement pour être en état READY.