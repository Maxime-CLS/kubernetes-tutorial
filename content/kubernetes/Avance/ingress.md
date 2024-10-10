---
title: "Ingress"
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

Cette section ne fonctionne pas sur MacOs. Vous devez absolument être sur une machine Linux.

## Activer le contrôleur d'entrée

Si vous utilisez minikube, vous devez activer le contrôleur NGNIX Ingress.

```
minikube addons enable ingress
```

Attendez une minute ou deux et vérifiez qu'il a été déployé correctement :

```
kubectl get pods -n ingress-nginx
```

```
ingress-nginx-admission-create-lqfh2        0/1     Completed   0          6m28s
ingress-nginx-admission-patch-z2lzj         0/1     Completed   2          6m28s
ingress-nginx-controller-69ccf5d9d8-95xgp   1/1     Running     0          6m28s
```

## Déployer l'application

```
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: quarkus-demo-deployment
spec:
  replicas: 1
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

Exposez le service :


```
kubectl expose deployment quarkus-demo-deployment --type=LoadBalancer --port=8080

kubectl get service quarkus-demo-deployment
```

```
NAME                      TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
quarkus-demo-deployment   NodePort   10.105.106.66   <none>        8080:30408/TCP   11s
```

```
IP=$(kubectl get service myapp -o jsonpath="{.status.loadBalancer.ingress[0].ip}")
PORT=$(kubectl get service myapp -o jsonpath="{.spec.ports[*].port}")
```

Réaliser une requete sur le service :

```
curl $IP:$PORT
```

## Configuration l'Ingress

Une ressource d'entrée est définie comme suit :

```
vim apps/kubefiles/demo-ingress.yaml
```


```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
  rules:
  - host: kube-team.info
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: quarkus-demo-deployment
            port:
              number: 8080

```

```
kubectl apply -f apps/kubefiles/demo-ingress.yaml
```

Obtenir les informations de la ressource Ingress :

```
kubectl get ingress
```

```
NAME              CLASS    HOSTS                 ADDRESS          PORTS   AGE
example-ingress   <none>   kube-team.info   192.168.99.115   80      68s
```

Vous devez attendre que le champ d'adresse soit défini. Cela peut prendre quelques minutes.

Modifiez le fichier /etc/hosts pour faire pointer le nom d'hôte vers l'adresse Ingress.

```
minikube ip
```

```
10.240.145.124
```

```
sudo vim /etc/hosts
```

```
10.240.145.124 kube-team.info
```

```
curl kube-team.info
```

Si vous avez un proxy :

```
curl --noproxy kube-team.info kube-team.info
```

```
Supersonic Subatomic Java with Quarkus quarkus-demo-deployment-8cf45f5c8-qmzwl:1
```

## Deuxième déploiement

Déployer une deuxième version du service :

```
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mynode-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mynode
  template:
    metadata:
      labels:
        app: mynode
    spec:
      containers:
      - name: mynode
        image: quay.io/rhdevelopers/mynode:v1
        ports:
        - containerPort: 8000
EOF
```

```
kubectl expose deployment mynode-deployment --type=NodePort --port=8000
```

## Mise à jour de l'Ingress

Ensuite, vous devez mettre à jour la ressource Ingress avec le nouveau chemin :


```
vim apps/kubefiles/demo-ingress-2.yaml
```

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
  rules:
  - host: kube-team.info
    http:
      paths:
      - path: /
        backend:
          serviceName: quarkus-demo-deployment
          servicePort: 8080
      - path: /v2
        backend:
          serviceName: mynode-deployment
          servicePort: 8000
```

```
kubectl apply -f apps/kubefiles/demo-ingress-2.yaml
```

Tester :

```
curl kube-team.info
```

```
Supersonic Subatomic Java with Quarkus quarkus-demo-deployment-8cf45f5c8-qmzwl:2
```

```
curl kube-team.info/v2
```

```
Node Bonjour on mynode-deployment-77c7bf857d-5nfl4 0
```

Supprimer les ressources

```
kubectl delete deployment mynode-deployment
kubectl delete service mynode-deployment

kubectl delete deployment quarkus-demo-deployment
kubectl delete service quarkus-demo-deployment

kubectl delete -f apps/kubefiles/demo-ingress-2.yaml
```


### A vous de jouer 

Lister l'ensemble des Customs Resources existant sur le cluster. 


---

Pour lister l'ensemble des custom resource d'un cluster Kubernetes, nous devons faire la commande suivante : 

```
kubectl get crd
```

```
NAME                                                  CREATED AT
bgpconfigurations.crd.projectcalico.org               2024-06-08T06:10:14Z
bgppeers.crd.projectcalico.org                        2024-06-08T06:10:14Z
blockaffinities.crd.projectcalico.org                 2024-06-08T06:10:14Z
caliconodestatuses.crd.projectcalico.org              2024-06-08T06:10:14Z
clusterinformations.crd.projectcalico.org             2024-06-08T06:10:14Z
crontabs.stable.killercoda.com                        2024-06-26T15:41:39Z
db-backups.stable.killercoda.com                      2024-06-26T15:41:39Z
felixconfigurations.crd.projectcalico.org             2024-06-08T06:10:14Z
globalnetworkpolicies.crd.projectcalico.org           2024-06-08T06:10:14Z
globalnetworksets.crd.projectcalico.org               2024-06-08T06:10:14Z
hostendpoints.crd.projectcalico.org                   2024-06-08T06:10:14Z
node.crd.projectcalico.org                            2024-06-08T06:10:14Z
ipamblocks.crd.projectcalico.org                      2024-06-08T06:10:14Z
ipamconfigs.crd.projectcalico.org                     2024-06-08T06:10:14Z
ipamhandles.crd.projectcalico.org                     2024-06-08T06:10:14Z
ippools.crd.projectcalico.org                         2024-06-08T06:10:14Z
ipreservations.crd.projectcalico.org                  2024-06-08T06:10:14Z
kubecontrollersconfigurations.crd.projectcalico.org   2024-06-08T06:10:14Z
networkpolicies.crd.projectcalico.org                 2024-06-08T06:10:14Z
networksets.crd.projectcalico.org                     2024-06-08T06:10:14Z
```

La liste de custom resource présent sur le cluster Kubernetes est associée au projet Open Source Calico. 

Si nous prenons l'exemple de la custom resource node, elle permet de définir une ressource de nœud (Node) représente un nœud exécutant Calico. Lors de l'ajout d'un hôte à un cluster Calico, une ressource node doit être créée, qui contient la configuration de l'instance calico/node s'exécutant sur l'hôte.

Lors du démarrage d'une instance calico/node, le nom fourni à l'instance doit correspondre au nom configuré dans la ressource Node.

Par défaut, le démarrage d'une instance calico/node crée automatiquement une ressource node en utilisant le nom d'hôte de l'hôte de calcul.

Traduit avec DeepL.com (version gratuite)


Pour aller plus loin voici la documentation : [Custom Resource](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)

Voici un exemple permettant la création d'une Custom Resource : 

1 - Créer un fichier YAML nommée crd

```
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # name must match the spec fields below, and be in the form: <plural>.<group>
  name: crontabs.stable.example.com
spec:
  # group name to use for REST API: /apis/<group>/<version>
  group: stable.example.com
  # list of versions supported by this CustomResourceDefinition
  versions:
    - name: v1
      # Each version can be enabled/disabled by Served flag.
      served: true
      # One and only one version must be marked as the storage version.
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                cronSpec:
                  type: string
                image:
                  type: string
                replicas:
                  type: integer
  # either Namespaced or Cluster
  scope: Namespaced
  names:
    # plural name to be used in the URL: /apis/<group>/<version>/<plural>
    plural: crontabs
    # singular name to be used as an alias on the CLI and for display
    singular: crontab
    # kind is normally the CamelCased singular type. Your resource manifests use this.
    kind: CronTab
    # shortNames allow shorter string to match your resource on the CLI
    shortNames:
    - ct
```

2 - Créer la Custom resource 

```
kubectl apply -f crd.yaml
```

```
customresourcedefinition.apiextensions.k8s.io/crontabs.stable.example.com created
```
