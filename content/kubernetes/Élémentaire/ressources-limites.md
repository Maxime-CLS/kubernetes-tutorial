---
title: "Ressources et limites"
date: 2020-06-26T15:17:20+02:00
draft: false
weight: 5
---

## Prérequis

- Minikube [Install](https://kubernetes.io/fr/docs/tasks/tools/install-minikube/#installez-minikube-par-t%C3%A9l%C3%A9chargement-direct)  [Driver none](https://kubernetes.io/docs/setup/learning-environment/minikube/#specifying-the-vm-driver)
- kubectl [Install](https://kubernetes.io/fr/docs/tasks/tools/install-kubectl/)
- Stern [Docs](https://kubernetes.io/blog/2016/10/tail-kubernetes-with-stern/) [Release](https://github.com/wercker/stern/releases)
- jq [Install](https://stedolan.github.io/jq/download/)
- 3 terminal SSH


Assurez-vous que vous êtes dans le bon espace de noms :

```
kubectl config set-context --current --namespace=myspace
```

Assurez-vous que rien n'est en cours d'exécution dans votre espace de nom :

```
kubectl get all
```

```
No resources found in myspace namespace.
```

Déployez d'abord une application sans aucune Requête ni Limite :


Créer un fichier de déploiement


```
mkdir -p apps/kubefiles/
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


Déployer la version 1 de l'applciation myboot

```
kubectl apply -f apps/kubefiles/myboot-deployment.yml
```

Décrivez le pod :

```
PODNAME=$(kubectl get pod -l app=myboot -o name)
kubectl describe $PODNAME
```

Il n'y a pas de limites de ressources configurées pour le pod.

```
Name:         myboot-66d7d57687-jzbzj
Namespace:    myspace
Priority:     0
Node:         gcp-5xldg-w-b-rlp45.us-central1-b.c.ocp42project.internal/10.0.32.5
Start Time:   Sun, 29 Mar 2020 14:24:24 -0400
Labels:       app=myboot
              pod-template-hash=66d7d57687
Annotations:  k8s.v1.cni.cncf.io/networks-status:
                [{
                    "name": "openshift-sdn",
                    "interface": "eth0",
                    "ips": [
                        "10.130.2.23"
                    ],
                    "dns": {},
                    "default-route": [
                        "10.130.2.1"
                    ]
                }]
              openshift.io/scc: restricted
Status:       Running
IP:           10.130.2.23
IPs:
  IP:           10.130.2.23
Controlled By:  ReplicaSet/myboot-66d7d57687
Containers:
  myboot:
    Container ID:   cri-o://2edfb0a5a93f375516ee49d33df20bee40c14792b37ec1648dc5205244095a53
    Image:          quay.io/burrsutter/myboot:v1
    Image ID:       quay.io/burrsutter/myboot@sha256:cdf39f191f5d322ebe6c04cae218b0ad8f6dbbb8a81e81a88c0fbc6e3c05f860
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Sun, 29 Mar 2020 14:24:32 -0400
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-vlzsl (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  default-token-vlzsl:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-vlzsl
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age        From                                                                Message
  ----    ------     ----       ----                                                                -------
  Normal  Scheduled  <unknown>  default-scheduler                                                   Successfully assigned myspace/myboot-66d7d57687-jzbzj to gcp-5xldg-w-b-rlp45.us-central1-b.c.ocp42project.internal
  Normal  Pulled     12m        kubelet, gcp-5xldg-w-b-rlp45.us-central1-b.c.ocp42project.internal  Container image "quay.io/burrsutter/myboot:v1" already present on machine
  Normal  Created    12m        kubelet, gcp-5xldg-w-b-rlp45.us-central1-b.c.ocp42project.internal  Created container myboot
  Normal  Started    12m        kubelet, gcp-5xldg-w-b-rlp45.us-central1-b.c.ocp42project.internal  Started container myboot
```

Supprimez ce déploiement :

```
kubectl delete deployment myboot
```

Créez un nouveau déploiement avec des demandes de ressources :

Créer un fichier de déploiement

```
vi apps/kubefiles/myboot-deployment-resources.yml
```

myboot-deployment-resources.yml

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
        resources:
          requests:
            memory: "300Mi"
            cpu: "10000m" # 10 cores
```

```
kubectl apply -f apps/kubefiles/myboot-deployment-resources.yml
```

Et vérifiez le statut du Pod :


```
kubectl get pods
```

```
NAME                      READY   STATUS    RESTARTS   AGE
myboot-7b7d754c86-kjwlr   0/1     Pending   0          19s
```


Si vous voulez obtenir plus d'informations sur l'erreur :

```
kubectl get events --sort-by=.metadata.creationTimestamp
```

```
<unknown>   Warning   FailedScheduling    pod/myboot-7b7d754c86-kjwlr    0/6 nodes are available: 6 Insufficient cpu.
<unknown>   Warning   FailedScheduling    pod/myboot-7b7d754c86-kjwlr    0/6 nodes are available: 6 Insufficient cpu.
```

Les "demandes de ressources" de la spécification du pod exigent qu'au moins un nœud de travail ait N cœurs et X mémoires disponibles. Si aucun nœud de travail ne répond à ces exigences, vous recevez le message "PENDING" et les notations appropriées dans la liste des événements.

Vous pouvez également utiliser kubectl describe sur le pod pour trouver plus d'informations sur l'échec.

```
PODNAME=$(kubectl get pod -l app=myboot -o name)
kubectl describe $PODNAME
```

Supprimez le déploiement :

```
kubectl delete -f apps/kubefiles/myboot-deployment-resources.yml
```

Créez un nouveau déploiement avec une demande de ressources plus raisonnable et une limite stricte :

Créer un fichier de déploiement

```
vi apps/kubefiles/myboot-deployment-resources-limits.yml
```

myboot-deployment-resources-limits.yml

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
        resources:
          requests:
            memory: "300Mi"
            cpu: "250m" # 1/4 core
          # NOTE: These are the same limits we tested our Docker Container with earlier
          # -m matches limits.memory and --cpus matches limits.cpu
          limits:
            memory: "400Mi"
            cpu: "1000m" # 1 core
```

```
kubectl apply -f apps/kubefiles/myboot-deployment-resources-limits.yml
```

Décrivez le Pod :

```
PODNAME=$(kubectl get pod -l app=myboot -o name)
kubectl describe $PODNAME
```

Déployer le service :

Créer un fichier pour votre service

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

Et surveillez votre Pod:

```
watch kubectl get pods
```

Créer les variables IP et Port

```
IP=$(minikube ip)
PORT=$(kubectl get service/myboot -o jsonpath="{.spec.ports[*].nodePort}")
```

Réaliser une requete sur le sevrice

```
curl $IP:$PORT
```

Exécuter une boucle

```
while true
do curl $IP:$PORT
sleep .3
done
```

Dans une autre fenêtre de terminal, curl le point de terminaison /sysresources

```
curl $IP:$PORT/sysresources
```

```
PODNAME=$(kubectl get pod -l app=myboot -o name)
kubectl get $PODNAME -o json | jq ".spec.containers[0].resources"
```

```
{
  "limits": {
    "cpu": "1",
    "memory": "400Mi"
  },
  "requests": {
    "cpu": "250m",
    "memory": "300Mi"
  }
}
```

Puis curl le point de terminaison /consume :

```
curl $IP:$PORT/consume
```

```
curl: (52) Empty reply from server
```

Et vous devriez remarquer que votre boucle échoue également :

```
Aloha from Spring Boot! 1120 on myboot-d78fb6d58-69kl7
curl: (56) Recv failure: Connection reset by peer
```

Décrivez le Pod pour voir l'erreur :

```
PODNAME=$(kubectl get pod -l app=myboot -o name)
kubectl describe $PODNAME
```

Et cherchez la partie suivante :

```
   Last State:     Terminated
      Reason:       OOMKilled
      Exit Code:    137
```

```
kubectl get $PODNAME -o json | jq ".status.containerStatuses[0].lastState.terminated"
```

```
{
  "containerID": "cri-o://7b9be70ce4b616d6083d528dee708cea879da967373dad0d396fb999bd3898d3",
  "exitCode": 137,
  "finishedAt": "2020-03-29T19:14:56Z",
  "reason": "OOMKilled",
  "startedAt": "2020-03-29T18:50:15Z"
}
```

Vous pourriez même voir la colonne STATUS avec **watch kubectl get pods** pour visualiser le OOMKilled :

```
NAME                     READY   STATUS      RESTARTS   AGE
myboot-d78fb6d58-69kl7   0/1     OOMKilled   1          30m
```

Et vous remarquerez que la colonne RESTARTS s'incrémente à chaque plantage du pod Spring Boot.

