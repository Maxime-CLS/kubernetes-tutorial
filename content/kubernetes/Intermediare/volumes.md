---
title: "Volumes"
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

Les conteneurs sont éphémères par définition, ce qui signifie que tout ce qui est stocké au moment de l'exécution est perdu lorsque le conteneur est arrêté. Cela peut poser des problèmes avec les conteneurs qui ont besoin de conserver leurs données, comme les conteneurs de base de données.

Un volume Kubernetes est simplement un répertoire accessible aux conteneurs d'un pod. Le concept est similaire aux volumes Docker, mais dans Docker, vous faites correspondre le conteneur à un ordinateur hôte. Dans le cas des volumes Kubernetes, le support qui le soutient et son contenu sont déterminés par le type de volume particulier utilisé.

Certains de ces types de volumes sont :

* awsElasticBlockStore
* azureDisk
* cephfs
* nfs
* local
* répertoire vide
* chemin de l'hôte

Commençons par deux exemples de Volumes.

## Volumes
#### EmptyDir

Un volume emptyDir est créé pour la première fois lorsqu'un Pod est affecté à un nœud et existe tant que ce Pod fonctionne sur ce nœud. Comme son nom l'indique, il est initialement vide. Tous les conteneurs d'un même pod peuvent lire et écrire dans le même volume emptyDir. Lorsqu'un Pod est redémarré ou supprimé, les données contenues dans le volume "emptyDir" sont perdues à jamais.

Déployons un service qui expose deux points de terminaison, l'un pour écrire du contenu dans un fichier et l'autre pour récupérer le contenu de ce fichier.

```
vim apps/kubefiles/myboot-pod-volume.yml
```

```
apiVersion: v1
kind: Pod
metadata:
  name: myboot-demo
spec:
  containers:
  - name: myboot-demo
    image: quay.io/rhdevelopers/myboot:v4

    volumeMounts:
    - mountPath: /tmp/demo
      name: demo-volume

  volumes:
  - name: demo-volume
    emptyDir: {}
```

Dans la section volumes, vous définissez le volume, et dans la section volumeMounts, comment le volume est monté à l'intérieur du conteneur.


```
kubectl apply -f apps/kubefiles/myboot-pod-volume.yml

kubectl get pods
```

```
NAME          READY   STATUS    RESTARTS   AGE
myboot-demo   1/1     Running   0          83s
```

Accédons au conteneur et exécutons ces méthodes :

```
kubectl exec -ti myboot-demo /bin/bash

curl localhost:8080/appendgreetingfile
curl localhost:8080/readgreetingfile
```

```
Jambo
```

Dans ce cas, le emptyDir a été défini à /tmp/demo, vous pouvez donc vérifier le contenu du répertoire en exécutant ls :

```
ls /tmp/demo
```

```
greeting.txt
```

Quitter le shell du conteneur :

```
exit
```

Et supprimez le pod.

```
kubectl delete pod myboot-demo
```

Ensuite, si vous déployez à nouveau le même service, vous remarquerez que le contenu du répertoire est vide.

```
kubectl exec -ti myboot-demo /bin/bash

ls /tmp/demo
exit
```

Et supprimez le pod.

```
kubectl delete pod myboot-demo
```

emptyDir est partagé entre les conteneurs d'un même Pod. Le déploiement suivant crée un pod avec deux conteneurs montant le même volume :

```
vim apps/kubefiles/myboot-pods-volume.yml
```

```
apiVersion: v1
kind: Pod
metadata:
  name: myboot-demo
spec:
  containers:
  - name: myboot-demo-1 #<.>
    image: quay.io/rhdevelopers/myboot:v4
    volumeMounts:
    - mountPath: /tmp/demo
      name: demo-volume

  - name: myboot-demo-2 #<.>
    image: quay.io/rhdevelopers/myboot:v4 #<.>

    env:
    - name: SERVER_PORT #<.>
      value: "8090"

    volumeMounts:
    - mountPath: /tmp/demo
      name: demo-volume

  volumes:
  - name: demo-volume #<.>
    emptyDir: {}
```

```
kubectl apply -f apps/kubefiles/myboot-pods-volume.yml

kubectl get pods
```

```
NAME          READY   STATUS    RESTARTS   AGE
myboot-demo   2/2     Running   0          4s
```

Accédons au premier conteneur et générons du contenu dans le répertoire /tmp/demo.

```
kubectl exec -ti myboot-demo -c myboot-demo-1 /bin/bash

curl localhost:8080/appendgreetingfile

exit
```
Et lire le contenu du fichier dans l'autre conteneur :

```
kubectl exec myboot-demo -c myboot-demo-2 "cat /tmp/demo/greeting.txt"
```

```
Jambo
```

Vous pouvez obtenir les informations sur le volume d'un Pod en exécutant :

```
kubectl describe pod myboot-demo
```

```
Volumes:
  demo-volume:
    Type:       EmptyDir (a temporary directory that shares a pods lifetime)
    Medium:
    SizeLimit:  <unset>
```

Supprimer les ressources

```
kubectl delete -f apps/kubefiles/myboot-pods-volume.yml
```

## HostPath

Un volume hostPath monte un fichier ou un répertoire du système de fichiers du nœud dans le Pod.


```
vim apps/kubefiles/myboot-pod-volume-hostpath.yaml
```

```
apiVersion: v1
kind: Pod
metadata:
  name: myboot-demo
spec:
  containers:
  - name: myboot-demo
    image: quay.io/rhdevelopers/myboot:v4

    volumeMounts:
    - mountPath: /tmp/demo
      name: demo-volume

  volumes:
  - name: demo-volume
    hostPath:
      path: "/mnt/data"
```

Dans ce cas, vous définissez le répertoire de l'hôte/nœud où le contenu sera stocké.

```
kubectl apply -f apps/kubefiles/myboot-pod-volume-hostpath.yaml
```

Maintenant, si vous décrivez le Pod, dans la section des volumes, vous verrez :

```
kubectl describe pod myboot-demo
```

```
Volumes:
  demo-volume:
    Type:          HostPath (bare host directory volume)
    Path:          /mnt/data
    HostPathType:
```

Notez que maintenant le contenu stocké dans /tmp/demo à l'intérieur du Pod est stocké dans le chemin de l'hôte /mnt/data, donc si le Pod meurt, le contenu n'est pas perdu. Mais cela ne résout pas tous les problèmes car si le Pod tombe en panne et qu'il est reprogrammé dans un autre nœud, les données ne seront pas dans cet autre nœud.

Voyons un autre exemple, dans ce cas pour un volume Amazon EBS :

```
apiVersion: v1
kind: Pod
metadata:
  name: test-ebs
spec:
...
  volumes:
    - name: test-volume
      awsElasticBlockStore:
        volumeID: <volume-id>
        fsType: ext4
```

Ce que nous voulons que vous remarquiez dans l'extrait précédent, c'est que vous mélangez des éléments de votre application (c'est-à-dire le conteneur, les sondes, les ports, ...) qui sont plus du côté du développement avec des éléments plus liés au cloud (c'est-à-dire le stockage physique), qui est plus du côté des opérations.

Pour éviter ce mélange de concepts, Kubernetes offre une certaine couche d'abstractions, de sorte que les développeurs demandent simplement de l'espace pour stocker les données (-persistent volume claim_), et l'équipe des opérations offre la configuration du stockage physique.

Supprimer les ressources

```
kubectl delete pod myboot-demo
```

## Persistent Volume & Persistent Volume Claim

Un PersistentVolume (PV) est une ressource Kubernetes qui est créée par un administrateur ou dynamiquement à l'aide de Storage Classes indépendamment de Pod. Il capture les détails de l'implémentation du stockage, il peut s'agir de NFS, Ceph, iSCSI, ou d'un système de stockage spécifique au fournisseur de cloud.

Une PersistentVolumeClaim (PVC) est une demande de stockage par un utilisateur. Il peut demander une taille de volume spécifique ou, par exemple, le mode d'accès.

#### Volume persistant/claim avec hostPath

Utilisons la stratégie hostPath sans la configurer directement en tant que volume, mais en utilisant le volume persistant et la réclamation de volume persistant.

```
vim apps/kubefiles/demo-persistent-volume-hostpath.yaml
```

```
kind: PersistentVolume
apiVersion: v1
metadata:
  name: my-persistent-volume
  labels:
    type: local
spec:
  storageClassName: pv-demo
  capacity:
    storage: 100Mi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/persistent-volume"
```

Désormais, les informations relatives au volume ne se trouvent plus dans le pod mais dans l'objet volume persistant.

```
kubectl apply -f apps/kubefiles/demo-persistent-volume-hostpath.yaml

kubectl get pv
```

```
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                                           STORAGECLASS   REASON   AGE
my-persistent-volume                       100Mi      RWO            Retain           Available                                                   pv-demo                 5s
```

Ensuite, du côté du développeur, nous devons réclamer ce dont nous avons besoin sur le PV. Dans l'exemple suivant, nous demandons un espace de 10Mi.

```
vim apps/kubefiles/myboot-persistent-volume-claim.yaml
```

```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: myboot-volumeclaim
spec:
  storageClassName: pv-demo
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Mi
```

```
kubectl apply -f apps/kubefiles/myboot-persistent-volume-claim.yaml

kubectl get pvc
```

```
NAME                 STATUS   VOLUME                 CAPACITY   ACCESS MODES   STORAGECLASS   AGE
myboot-volumeclaim   Bound    my-persistent-volume   100Mi      RWO            pv-demo        3s
```

La grande différence est que maintenant, dans le pod, vous définissez simplement dans la section volumes, non pas la configuration du volume directement, mais la revendication du volume persistant à utiliser.

```
vim apps/kubefiles/myboot-pod-volume-pvc.yaml
```

```
apiVersion: v1
kind: Pod
metadata:
  name: myboot-demo
spec:
  containers:
  - name: myboot-demo
    image: quay.io/rhdevelopers/myboot:v4

    volumeMounts:
    - mountPath: /tmp/demo
      name: demo-volume

  volumes:
  - name: demo-volume
    persistentVolumeClaim:
      claimName: myboot-volumeclaim
```

```
kubectl apply -f apps/kubefiles/myboot-pod-volume-pvc.yaml

kubectl describe pod myboot-demo
```

```
Volumes:
  demo-volume:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  myboot-volumeclaim
    ReadOnly:   false
```

Remarquez que maintenant la description du pod montre que le volume n'est pas défini directement mais par le biais d'une réclamation de volume de persistance.

```
kubectl delete pod myboot-demo

kubectl get pvc
```

Même si le pod a été supprimé, le PVC (et le PV) sont toujours là et doivent être supprimés manuellement.

```
NAME                 STATUS   VOLUME                 CAPACITY   ACCESS MODES   STORAGECLASS   AGE
myboot-volumeclaim   Bound    my-persistent-volume   100Mi      RWO            pv-demo        14m
```

Supprimer les ressources

```
kubectl delete -f apps/kubefiles/myboot-persistent-volume-claim.yaml
kubectl delete -f apps/kubefiles/demo-persistent-volume-hostpath.yaml
```

## Provisonnement Statique vs Dynamique

Les Persistent Volumes peuvent être provisionnés de manière dynamique ou statique.

Le provisionnement statique permet aux administrateurs de clusters de mettre à disposition d'un cluster une unité de stockage existante. Lorsqu'il est effectué de cette manière, le PV et le PVC doivent être fournis manuellement.

Jusqu'à présent, dans le dernier exemple, vous avez vu le provisionnement statique.

Avec le provisionnement dynamique, les administrateurs de clusters n'ont plus besoin de pré-provisionner le stockage. Au lieu de cela, il provisionne automatiquement le stockage lorsqu'il est demandé par les utilisateurs. Pour le faire fonctionner, vous devez fournir un objet Storage Class et un PVC s'y référant. Une fois le PVC créé, le périphérique de stockage et le PV sont automatiquement créés pour vous. L'objectif principal du provisionnement dynamique est de travailler avec des solutions de fournisseurs de cloud.

Normalement, l'implémentation de Kubernetes propose une classe de stockage par défaut afin que chacun puisse démarrer rapidement avec le provisionnement dynamique. Vous pouvez obtenir des informations sur la classe de stockage par défaut en exécutant :

```
kubectl get sc
```
```
NAME                 PROVISIONER                RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
standard (default)   k8s.io/minikube-hostpath   Delete          Immediate           false                  47d
```

Ensuite, vous pouvez créer une réclamation de volume persistant qui créera automatiquement un volume persistant.

```
vim apps/kubefiles/demo-dynamic-persistent.yaml
```

```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: myboot-volumeclaim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Mi
```

Puisque nous n'avons pas spécifié de classe de stockage mais qu'il y en a une définie par défaut, le PVC se réfère implicitement à celle-ci.


```
kubectl apply -f apps/kubefiles/demo-dynamic-persistent.yaml

kubectl get pvc
```

```
NAME                 STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
myboot-volumeclaim   Pending                                      gp2            46sç
```

Remarquez que le PVC est en état d'attente, car rappelez-vous que nous créons un stockage dynamique et que cela signifie que tant que le pod ne demande pas le volume, le PVC restera en état d'attente et le PV ne sera pas créé.

```
kubectl apply -f apps/kubefiles/myboot-pod-volume-pvc.yaml

kubectl get pods
```

```
NAME          READY   STATUS    RESTARTS   AGE
myboot-demo   1/1     Running   0          2m36s
```

Lorsque le pod est en état de fonctionnement, vous pouvez obtenir les paramètres PVC et PV.

```
kubectl get pvc
```

```
NAME                 STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
myboot-volumeclaim   Bound    pvc-6de4f27e-bd40-4b58-bb46-91eb08ca5bd7   1Gi        RWO            gp2            116s
```

Remarquez que maintenant la demande de volume est liée à un volume.

Enfin, vous pouvez vérifier que le PV a été créé automatiquement :


kubectl get pv

```
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                        STORAGECLASS   REASON   AGE
pvc-6de4f27e-bd40-4b58-bb46-91eb08ca5bd7   1Gi        RWO            Delete           Bound    default/myboot-volumeclaim   gp2                     77s
```

Notez que le champ CLAIM pointe vers le PVC responsable de la création du PV.

Supprimer les ressources

```
kubectl delete -f apps/kubefiles/myboot-pod-volume-pvc.yaml
kubectl delete -f apps/kubefiles/demo-dynamic-persistent.yaml
```

## Systèmes de fichiers distribués

Il est important de noter que les fournisseurs de cloud computing proposent des stockages distribués afin que les données soient toujours disponibles dans tous les nœuds. Comme vous l'avez vu dans le dernier exemple, cette classe de stockage garantit que tous les nœuds voient le même contenu de disque.

Si par exemple, vous utilisez Kubernetes on-prem ou si vous ne voulez pas relayer vers une solution fournisseur, il existe également une prise en charge des systèmes de fichiers distribués dans Kubernetes. Si c'est le cas, nous vous recommandons d'utiliser NFS, GlusterFS ou Ceph.





