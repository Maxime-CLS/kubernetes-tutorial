---
title: "Operator"
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

Les ressources personnalisées étendent l'API

Les contrôleurs personnalisés fournissent la fonctionnalité - qui maintient continuellement l'état souhaité - pour surveiller son état et rapprocher la ressource de la configuration.

[Docs Custom Resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)

[Docs Custom Resource Definition](https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definitions/)

Définitions de ressources personnalisées (CRD) dans la version 1.7


## CRDs

```
kubectl get crds --all-namespaces
kubectl api-resources
```

Exemple CRD

```
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: pizzas.mykubernetes.acme.org
  labels:
    app: pizzamaker
    mylabel: stuff
spec:
  group: mykubernetes.acme.org
  scope: Namespaced
  version: v1beta2
  names:
    kind: Pizza
    listKind: PizzaList
    plural: pizzas
    singular: pizza
    shortNames:
    - pz
```

Ajoutez des Pizzas

```
mkdir -p apps/pizzas/
vim pizza-crd.yaml
```

pizza-crd.yaml

```
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: pizzas.mykubernetes.acme.org
  labels:
    app: pizzamaker
    mylabel: stuff
spec:
  group: mykubernetes.acme.org
  scope: Namespaced
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        description: "A custom resource for making yummy pizzas" #<.>
        type: object
        properties:
          spec:
            type: object
            description: "Information about our pizza"
            properties:
              toppings: #<.>
                type: array
                items:
                  type: string
                description: "List of toppings for our pizza"
              sauce: #<.>
                type: string
                description: "The name of the sauce to use on our pizza"
  names:
    kind: Pizza #<.>
    listKind: PizzaList
    plural: pizzas
    singular: pizza
    shortNames:
    - pz
```

```
kubectl create namespace pizzahat
kubectl config set-context --current --namespace=pizzahat

kubectl apply -f apps/pizzas/pizza-crd.yaml
```

Fait maintenant partie de l'API

```
kubectl get crds | grep pizza
```

Résultat

```
NAME                           CREATED AT
pizzas.mykubernetes.acme.org   2022-01-23T16:26:11Z
```


```
kubectl api-resources | grep pizzas
```

Résultat

```
pizzas                            pz           mykubernetes.acme.org          true         Pizza
```

Déploiement de l'opérateur

```
vim apps/pizzas/pizza-deployment.yaml
```

pizza-deployment.yaml

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: quarkus-operator-example
rules:
- apiGroups:
  - ''
  resources:
  - pods
  verbs:
  - get
  - list
  - watch
  - create
  - update
  - delete
  - patch
- apiGroups:
  - apiextensions.k8s.io
  resources:
  - customresourcedefinitions
  verbs:
  - list
- apiGroups:
  - mykubernetes.acme.org
  resources:
  - pizzas
  verbs:
  - list
  - watch
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: quarkus-operator-example
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: quarkus-operator-example
subjects:
- kind: ServiceAccount
  name: quarkus-operator-example
  namespace: pizzahat
roleRef:
  kind: ClusterRole
  name: quarkus-operator-example
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: quarkus-operator-example
spec:
  selector:
    matchLabels:
      app: quarkus-operator-example
  replicas: 1
  template:
    metadata:
      labels:
        app: quarkus-operator-example
    spec:
      serviceAccountName: quarkus-operator-example
      containers:
      - image: quay.io/rhdevelopers/pizza-operator:1.0.1
        name: quarkus-operator-example
        imagePullPolicy: IfNotPresent
```

```
kubectl apply -f apps/pizzas/pizza-deployment.yaml

kubectl get pods
```

```
NAME                                        READY   STATUS    RESTARTS   AGE
quarkus-operator-example-5f5bf777bc-glfg9   1/1     Running   0          58s
```

Faire des pizzas

```
vim apps/pizzas/cheese-pizza.yaml
```

cheese-pizza.yaml

```
apiVersion: mykubernetes.acme.org/v1
kind: Pizza
metadata:
  name: cheesep
spec:
  toppings:
  - mozzarella
  sauce: regular
```

```
kubectl apply -f apps/pizzas/cheese-pizza.yaml
kubectl get pizzas
```

```
NAME      AGE
cheesep   4s
```

```
kubectl describe pizza cheesep
```

```
Name:         cheesep
Namespace:    pizzahat
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"mykubernetes.acme.org/v1beta2","kind":"Pizza","metadata":{"annotations":{},"name":"cheesep","namespace":"pizzahat"},"spec":...
API Version:  mykubernetes.acme.org/v1beta2
Kind:         Pizza
...
```

```
kubectl get pods
```

```
NAME                                        READY   STATUS      RESTARTS   AGE
cheesep-pod                                 0/1     Completed   0          3s
quarkus-operator-example-5f5bf777bc-glfg9   1/1     Running     0          44m
```

Et vérifiez les logs du Pod de fromage :

```
kubectl logs cheesep-pod
```

```
__  ____  __  _____   ___  __ ____  ______
 --/ __ \/ / / / _ | / _ \/ //_/ / / / __/
 -/ /_/ / /_/ / __ |/ , _/ ,< / /_/ /\ \
--\___\_\____/_/ |_/_/|_/_/|_|\____/___/
2022-07-23 09:03:11,537 INFO  [io.quarkus] (main) pizza-maker 1.0-SNAPSHOT (powered by Quarkus 1.4.0.CR1) started in 0.006s.
2022-07-23 09:03:11,537 INFO  [io.quarkus] (main) Profile prod activated.
2022-07-23 09:03:11,537 INFO  [io.quarkus] (main) Installed features: [cdi]
Doing The Base
Adding Sauce regular
Adding Toppings [mozzarella]
Baking
Baked
Ready For Delivery
2022-01-23 09:03:12,038 INFO  [io.quarkus] (main) pizza-maker stopped in 0.000s
```

Faire plus de pizzas

```
vim apps/pizzas/meat-lovers.yaml
```

meat-lovers.yaml

```
apiVersion: mykubernetes.acme.org/v1
kind: Pizza
metadata:
  name: meatsp
spec:
  toppings:
  - mozzarella
  - pepperoni
  - sausage
  - bacon
  sauce: extra
```

```
vim apps/pizzas/veggie-lovers.yaml
```

```
apiVersion: mykubernetes.acme.org/v1
kind: Pizza
metadata:
  name: veggiep
spec:
  toppings:
  - mozzarella
  - black olives
  sauce: extra
```

```
kubectl apply -f apps/pizzas/meat-lovers.yaml
kubectl apply -f apps/pizzas/veggie-lovers.yaml
kubectl get pizzas --all-namespaces
```

Manger toutes les pizzas

```
kubectl delete pizzas --all
```

Supprimer les ressources

```
kubectl delete all --all
```


## Créer un peu de Kafka

Kafka pour Minikube
Créez un nouvel espace de nom pour cette expérience :

```
kubectl create namespace franz
kubectl config set-context --current --namespace=franz
```

Déployer l'opérateur Kafka :

```
curl -sL https://github.com/operator-framework/operator-lifecycle-manager/releases/download/v0.20.0/install.sh | bash -s v0.20.0
```

```
kubectl create -f https://operatorhub.io/install/strimzi-kafka-operator.yaml
```

```
kubectl get csv
```

Attendez un peu jusqu'à l'état succès :

```
watch kubectl get csv
```

```
NAME                               DISPLAY   VERSION   REPLACES                           PHASE
strimzi-cluster-operator.v0.27.1   Strimzi   0.27.1    strimzi-cluster-operator.v0.27.0   Succeeded
```

Démarrer une veille dans un autre terminal :

```
watch kubectl get pods
```

Ensuite, déployez la ressource en demandant un cluster Kafka :

```
vim apps/kubefiles/mykafka.yml
```

mykafka.yml

```
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    version: 3.0.0
    replicas: 3
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      log.message.format.version: '3.0'
      inter.broker.protocol.version: '3.0'
    storage:
      type: ephemeral
  zookeeper:
    replicas: 3
    storage:
      type: ephemeral
  entityOperator:
    topicOperator: {}
    userOperator: {}
```


```
kubectl apply -f apps/kubefiles/mykafka.yml
```

Résultat :

```
NAME                                          READY   STATUS    RESTARTS   AGE
my-cluster-entity-operator-66676cb9fb-fzckz   2/2     Running   0          29s
my-cluster-kafka-0                            2/2     Running   0          60s
my-cluster-kafka-1                            2/2     Running   0          60s
my-cluster-kafka-2                            2/2     Running   0          60s
my-cluster-zookeeper-0                        2/2     Running   0          92s
my-cluster-zookeeper-1                        2/2     Running   0          92s
my-cluster-zookeeper-2                        2/2     Running   0          92s
```

```
kubectl get kafkas
```

```
NAME         DESIRED KAFKA REPLICAS   DESIRED ZK REPLICAS   READY   WARNINGS
my-cluster   3                        3                     True    True
```

Supprimer les ressources

```
kubectl delete namespace pizzahat
kubectl delete -f apps/pizzas/pizza-crd.yaml
kubectl delete kafka my-cluster
kubectl delete namespace franz
```
