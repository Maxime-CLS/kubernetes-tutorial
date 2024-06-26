---
title: "Kubectl"
date: 2020-06-26T15:17:20+02:00
draft: false
weight: 5
---

## Prérequis

- Une machine Linux Ubuntu ou Debian 
- 8 CPU, 16GB RAM, 100GB Disk
- Minikube [Install](https://kubernetes.io/fr/docs/tasks/tools/install-minikube/#installez-minikube-par-t%C3%A9l%C3%A9chargement-direct) 
- kubectl [Install](https://kubernetes.io/fr/docs/tasks/tools/install-kubectl/)
- Stern [Docs](https://kubernetes.io/blog/2016/10/tail-kubernetes-with-stern/) [Release](https://github.com/stern/stern/releases)
- jq [Install](https://stedolan.github.io/jq/download/)
- 3 terminal SSH

### Installation de minikube

#### Linux

```
mkdir bin && cd bin
curl -Lo minikube https://storage.googleapis.com/minikube/releases/v1.29.0/minikube-linux-amd64
chmod +x minikube
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.26.1/bin/linux/amd64/kubectl
chmod +x kubectl
cd ..
```

#### MacOs 

```
mkdir bin && cd bin
curl -Lo minikube https://storage.googleapis.com/minikube/releases/v1.29.0/minikube-darwin-amd64
chmod +x minikube
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.26.1/bin/darwin/amd64/kubectl
chmod +x kubectl
cd ..
```

Et ajouter les variables d'environnement : 

```
export MINIKUBE_HOME=$(pwd);
export PATH=$MINIKUBE_HOME/bin:$PATH
export KUBECONFIG=$MINIKUBE_HOME/.kube/config
export KUBE_EDITOR="code -w"
```

Conserver les paramètres de Vim dans .vimrc

Nous examinons les paramètres importants de Vim si vous souhaitez travailler avec YAML pendant le TP K8s.

**Paramètres**

Créez d'abord ou ouvrez (s'il existe déjà) le fichier .vimrc :

```
vim ~/.vimrc
```

Saisissez ensuite (en mode insertion activé avec i) les lignes suivantes :

```
alias k=kubectl
```

Sauvegardez et fermez le fichier en appuyant sur Esc suivi de :x et Enter.

### Démarrer le cluster Kubernetes

#### Linux

```
minikube start --memory=8192 --cpus=3 --kubernetes-version=v1.26.1 --vm-driver=docker
```

Avec un proxy :

```
minikube start --memory=8192 --cpus=3 --docker-env HTTPS_PROXY=$HTTPS_PROXY --docker-env HTTP_PROXY=$HTTP_PROXY --docker-env=NO_PROXY=$NO_PROXY --kubernetes-version=v1.26.1 --vm-driver=docker
```


#### MacOs 

```
minikube start --memory=8192 --cpus=3 --kubernetes-version=v1.26.1 --vm-driver=docker
```

Avec un proxy :

```
minikube start --memory=8192 --cpus=3 --docker-env HTTPS_PROXY=$HTTPS_PROXY --docker-env HTTP_PROXY=$HTTP_PROXY --docker-env=NO_PROXY=$NO_PROXY --kubernetes-version=v1.26.1 --vm-driver=docker
```

Et le résultat doit être quelque chose de similaire :

```
😄  [devnation] minikube v1.20.0 on Darwin 11.3
✅  Created a new profile : devnation
✅  minikube profile was successfully set to devnation
😄  [default] minikube v1.29.0 on Darwin 11.3
✨  Selecting 'virtualbox' driver from user configuration (alternates: [hyperkit])
🔥  Creating virtualbox VM (CPUs=2, Memory=8192MB, Disk=50000MB) ...
🐳  Preparing Kubernetes v1.26.1 on Docker '20.10.6' ...
    ▪ apiserver.enable-admission-plugins=LimitRanger,NamespaceExists,NamespaceLifecycle,ResourceQuota,ServiceAccount,DefaultStorageClass,MutatingAdmissionWebhook
🚜  Pulling images ...
🚀  Launching Kubernetes ...
⌛  Waiting for cluster to come online ...
🏄  Done! kubectl is now configured to use "devnation"
```

Enfin, configurez l'utilisation de minikube internal docker comme docker host :

```
eval $(minikube docker-env)
```

### Installation de l'ingress controller 

```
minikube addons enable ingress
```

Vérifier que l'ingress controller est bien installé :

``` 
kubectl get pods -n ingress-nginx
```

Résultat :

```
NAME                                        READY   STATUS      RESTARTS    AGE
ingress-nginx-admission-create-g9g49        0/1     Completed   0          11m
ingress-nginx-admission-patch-rqp78         0/1     Completed   1          11m
ingress-nginx-controller-59b45fb494-26npt   1/1     Running     0          11m
```

### Parlez à votre Cluster

```
echo $KUBECONFIG
kubectl config view
```

 
Affiche les paramètres fusionnés de kubeconfig
  

### Vue du noeud

```
kubectl get nodes
kubectl get nodes --show-labels
kubectl get namespaces
```

 
Affiche tous les noeuds, les labels définit et les espaces de noms.
  

### Voir les Pods prêts à l'emploi

Votre fournisseur de Kubernetes comprend probablement de nombreuses [espaces de noms](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/) prêtes à l'emploi :  

```
kubectl get pods --all-namespaces
kubectl get pods --all-namespaces --show-labels
kubectl get pods --all-namespaces -o wide
```
 
Affiche tous les espaces de noms, les labels définit et les sorties.
Les espaces de noms sont destinés à être utilisés dans des environnements avec de nombreux utilisateurs répartis sur plusieurs équipes ou projets. 
Les espaces de noms sont un moyen de diviser les ressources du cluster entre plusieurs utilisateurs
  

### Déployer quelque chose

Créer un espace de nommage et déployer quelque chose :  

```
kubectl create namespace mystuff
kubectl config set-context --current --namespace=mystuff

kubectl create deployment myapp --image=quay.io/rhdevelopers/quarkus-demo:v1
```

 
La commande "kubectl config set-context" permet une bascule rapide entre les namespaces du cluster kubernetes.
  

### Tout en surveillant les événements

**terminal 2**

```
watch kubectl get events --sort-by=.metadata.creationTimestamp
```

```
LAST SEEN   TYPE     REASON              OBJECT                        MESSAGE
<unknown>   Normal   Scheduled           pod/myapp-5dcbf46dfc-ghrk4    Successfully assigned mystuff/myapp-5dcbf46dfc-ghrk4 to g
cp-5xldg-w-a-5ptpn.us-central1-a.c.ocp42project.internal
29s         Normal   SuccessfulCreate    replicaset/myapp-5dcbf46dfc   Created pod: myapp-5dcbf46dfc-ghrk4
29s         Normal   ScalingReplicaSet   deployment/myapp              Scaled up replica set myapp-5dcbf46dfc to 1
21s         Normal   Pulling             pod/myapp-5dcbf46dfc-ghrk4    Pulling image "quay.io/burrsutter/quarkus-demo:1.0.0"
15s         Normal   Pulled              pod/myapp-5dcbf46dfc-ghrk4    Successfully pulled image "quay.io/burrsutter/quarkus-dem
o:1.0.0"
15s         Normal   Created             pod/myapp-5dcbf46dfc-ghrk4    Created container quarkus-demo
15s         Normal   Started             pod/myapp-5dcbf46dfc-ghrk4    Started container quarkus-demo
```

 
La commande "watch" permet d'initer une écoute en temps réel des modifications d'un objet.
  

### Objets créés

**Déploiements**  

```
kubectl get deployments
```

```
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
myapp   1/1     1            1           95s
```

 
La sortie observer permet de connaitre l'état du déploiment d'un ou plusieurs pods. Le déploiement fournit des mises à jour déclaratives pour Pods et ReplicaSets. Il permet de décrire l'état désiré et le controlleur du déploiement change l'état réel à l'état souhaité.
  

**Replicasets**

```
kubectl get replicasets
```

```
NAME               DESIRED   CURRENT   READY   AGE
myapp-5dcbf46dfc   1         1         1       2m1s
```

 
La sortie observer permet de connaitre l'état d'un ensemble stable de Pods à un moment donné. Cet objet est souvent utilisé pour garantir la disponibilité d'un certain nombre identique de Pods.
  

**Pods**

```
kubectl get pods --show-labels
```

```
NAME                     READY   STATUS    RESTARTS   AGE     LABELS
myapp-5dcbf46dfc-ghrk4   1/1     Running   0          2m18s   app=myapp,pod-template-hash=5dcbf46dfc
```

 
La sortie observer permet de connaitre l'état du pod. Les Pods sont les plus petites unités informatiques déployables qui peuvent être créées et gérées dans Kubernetes.

Un pod (terme anglo-saxon décrivant un groupe de baleines ou une gousse de pois) est un groupe d'un ou plusieurs conteneurs (comme des conteneurs Docker), ayant du stockage/réseau partagé, et une spécification sur la manière d'exécuter ces conteneurs. Les éléments d'un pod sont toujours co-localisés et co-ordonnancés, et s'exécutent dans un contexte partagé. Un pod modélise un "hôte logique" spécifique à une application - il contient un ou plusieurs conteneurs applicatifs qui sont étroitement liés — dans un monde pré-conteneurs, être exécuté sur la même machine physique ou virtuelle signifierait être exécuté sur le même hôte logique.
  

**Logs**

```
kubectl logs -l app=myapp
```

```
2020-03-22 14:41:30,497 INFO  [io.quarkus] (main) Quarkus 0.22.0 started in 0.021s. Listening on: http://0.0.0.0:8080
2020-03-22 14:41:30,497 INFO  [io.quarkus] (main) Installed features: [cdi, resteasy]
```

### Exposer un service

```
kubectl expose deployment myapp --port=8080 --type=LoadBalancer
```

**terminal 2**
```
watch kubectl get services
```

```
NAME    TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
myapp   LoadBalancer   172.30.103.41   <pending>     8080:31974/TCP   4s
```

 
Kubernetes ServiceTypesvous permet de spécifier le type de service que vous souhaitez. La valeur par défaut est ClusterIP.  

Type les valeurs et leurs comportements sont:  

- ClusterIP: Expose le service sur une IP interne au cluster. Le choix de cette valeur rend le service uniquement accessible à partir du cluster. C'est la valeur par défaut ServiceType.  

- NodePort: Expose le service sur l'IP de chaque nœud à un port statique (le NodePort). Un ClusterIPservice, vers lequel le NodePortservice est acheminé, est automatiquement créé. Vous pourrez contacter le NodePortService, depuis l'extérieur du cluster, en faisant la demande <NodeIP>:<NodePort>.  

- LoadBalancer: Expose le service en externe à l'aide de l'équilibreur de charge d'un fournisseur de cloud. NodePortet les ClusterIPservices, vers lesquels les itinéraires de l'équilibreur de charge externe, sont automatiquement créés.  

- ExternalName: Mappe le service au contenu du externalNamechamp (par exemple foo.bar.example.com), en renvoyant un CNAME enregistrement avec sa valeur. Aucun mandataire d'aucune sorte n'est mis en place. 
  

### Parler aux applications

terminal 3

```
sudo minikube tunnel
```

terminal 1 

```
IP=$(kubectl get service myapp -o jsonpath="{.status.loadBalancer.ingress[0].ip}")
PORT=$(kubectl get service myapp -o jsonpath="{.spec.ports[*].port}")
```

Sondez le résultat :

```
while true
do curl $IP:$PORT
sleep .3
done
```

Résultats du sondage :

```
Supersonic Subatomic Java with Quarkus myapp-5dcbf46dfc-ghrk4:289
Supersonic Subatomic Java with Quarkus myapp-5dcbf46dfc-ghrk4:290
Supersonic Subatomic Java with Quarkus myapp-5dcbf46dfc-ghrk4:291
Supersonic Subatomic Java with Quarkus myapp-5dcbf46dfc-ghrk4:292
Supersonic Subatomic Java with Quarkus myapp-5dcbf46dfc-ghrk4:293
```

Terminal 3
Changer les répliques :

```
kubectl scale deployment myapp --replicas=3
```

```
NAME                     READY   STATUS              RESTARTS   AGE
myapp-5dcbf46dfc-6sn2s   0/1     ContainerCreating   0          4s
myapp-5dcbf46dfc-ghrk4   1/1     Running             0          5m32s
myapp-5dcbf46dfc-z6hqw   0/1     ContainerCreating   0          4s
```

Commencez une mise à jour continue en changeant l'image : 

```
kubectl set image deployment/myapp quarkus-demo=quay.io/rhdevelopers/myboot:v1
```

```
Supersonic Subatomic Java with Quarkus myapp-5dcbf46dfc-6sn2s:188
Supersonic Subatomic Java with Quarkus myapp-5dcbf46dfc-z6hqw:169
Aloha from Spring Boot! 0 on myapp-58b97dbd95-vxd87
Aloha from Spring Boot! 1 on myapp-58b97dbd95-vxd87
Supersonic Subatomic Java with Quarkus myapp-5dcbf46dfc-6sn2s:189
Supersonic Subatomic Java with Quarkus myapp-5dcbf46dfc-z6hqw:170
Aloha from Spring Boot! 2 on myapp-58b97dbd95-vxd87
```

```
kubectl set image deployment/myapp quarkus-demo=quay.io/rhdevelopers/myboot:v2
```

```
Bonjour from Spring Boot! 2 on myapp-7d58855c6b-6c8gd
Bonjour from Spring Boot! 3 on myapp-7d58855c6b-6c8gd
Aloha from Spring Boot! 7 on myapp-58b97dbd95-mjlwx
Bonjour from Spring Boot! 4 on myapp-7d58855c6b-6c8gd
Aloha from Spring Boot! 8 on myapp-58b97dbd95-mjlwx
Bonjour from Spring Boot! 5 on myapp-7d58855c6b-6c8gd
```

```
kubectl set image deployment/myapp quarkus-demo=quay.io/rhdevelopers/quarkus-demo:v1
```

```
Bonjour from Spring Boot! 14 on myapp-7d58855c6b-dw67s
Supersonic Subatomic Java with Quarkus myapp-5dcbf46dfc-tcfwp:3
Supersonic Subatomic Java with Quarkus myapp-5dcbf46dfc-tcfwp:4
Bonjour from Spring Boot! 15 on myapp-7d58855c6b-dw67s
Supersonic Subatomic Java with Quarkus myapp-5dcbf46dfc-tcfwp:5
Bonjour from Spring Boot! 13 on myapp-7d58855c6b-72wp8
Supersonic Subatomic Java with Quarkus myapp-5dcbf46dfc-7rkxj:1
Supersonic Subatomic Java with Quarkus myapp-5dcbf46dfc-7rkxj:2
Supersonic Subatomic Java with Quarkus myapp-5dcbf46dfc-7lf9t:1
Supersonic Subatomic Java with Quarkus myapp-5dcbf46dfc-7rkxj:3
Supersonic Subatomic Java with Quarkus myapp-5dcbf46dfc-7lf9t:2
Supersonic Subatomic Java with Quarkus myapp-5dcbf46dfc-7lf9t:3
Supersonic Subatomic Java with Quarkus myapp-5dcbf46dfc-tcfwp:6
```

### Nettoyage

```
kubectl delete namespace mystuff
kubectl config set-context --current --namespace=default
```
