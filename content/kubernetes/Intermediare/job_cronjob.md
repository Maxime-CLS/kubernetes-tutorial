---
title: "Job & CronJob"
date: 2020-06-26T15:17:20+02:00
draft: false
weight: 35
---

## Prérequis

- Minikube [Install](https://kubernetes.io/fr/docs/tasks/tools/install-minikube/#installez-minikube-par-t%C3%A9l%C3%A9chargement-direct)  [Driver none](https://kubernetes.io/docs/setup/learning-environment/minikube/#specifying-the-vm-driver)
- kubectl [Install](https://kubernetes.io/fr/docs/tasks/tools/install-kubectl/)
- Stern [Docs](https://kubernetes.io/blog/2016/10/tail-kubernetes-with-stern/) [Release](https://github.com/stern/stern/releases)
- jq [Install](https://stedolan.github.io/jq/download/)
- 3 terminal SSH


## Preparation

Si vous exécutez ce tutoriel dans Minikube, vous devez déployer un seul noeud :

```
minikube stop
minikube delete --all
minikube start --vm-driver=none
```



La plupart du temps, vous utilisez Kubernetes comme plateforme pour exécuter des processus "longs" dont l'objectif est de fournir des réponses à une requête entrante donnée.

Mais Kubernetes vous permet également d'exécuter des processus dont le but est d'exécuter une certaine logique (par exemple, mise à jour de la base de données, traitement par lots, ...) et de mourir.

Les Jobs Kubernetes sont des tâches qui exécutent une certaine logique une fois.

Les CronJobs de Kubernetes sont des tâches qui se répètent en suivant un modèle Cron.

Ajouter des nœuds supplémentaires pour exécuter cette partie du tutoriel. Vérifiez le nombre de nœuds que vous avez delpoyés en exécutant :

## Job

Un job est créé à l'aide de la ressource Kubernetes Job :

```
vim  apps/kubefiles/whalesay-job.yaml
```

```
apiVersion: batch/v1
kind: Job
metadata:
  name: whale-say-job
spec:  
  template:
    spec:
      containers:
      - name: whale-say-container
        image: docker/whalesay
        command: ["cowsay","Hello Kubernetes Team"]
      restartPolicy: Never
```

```
kubectl apply -f apps/kubefiles/whalesay-job.yaml

watch kubectl get pods
```

```
NAME                  READY   STATUS              RESTARTS   AGE
whale-say-job-lp4n5   0/1     ContainerCreating   0          9s

NAME                  READY   STATUS    RESTARTS   AGE
whale-say-job-lp4n5   1/1     Running   0          19s

NAME                  READY   STATUS      RESTARTS   AGE
whale-say-job-lp4n5   0/1     Completed   0          25s
```

Vous pouvez obtenir des emplois comme toute autre ressource Kubernetes :

```
kubectl get jobs
```

```
NAME            COMPLETIONS   DURATION   AGE
whale-say-job   1/1           20s        36s
```

Pour obtenir le résultat de l'exécution du job :

```
kubectl logs whale-say-job-lp4n5
```

```
 _______________________
< Hello Kubernetes Team >
 -----------------------
    \
     \
      \
                    ##        .
              ## ## ##       ==
           ## ## ## ##      ===
       /""""""""""""""""___/ ===
  ~~~ {~~ ~~~~ ~~~ ~~~~ ~~ ~ /  ===- ~~~
       \______ o          __/
        \    \        __/
          \____\______/
```

Supprimer les ressources

```
kubectl delete -f apps/kubefiles/whalesay-job.yaml
```

## CronJobs

Un CronJob est défini à l'aide de la ressource Kubernetes CronJob :

```
vim apps/kubefiles/whalesay-cronjob.yaml
```


```
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: whale-say-cronjob
spec:
  schedule: "*/1 * * * *" (1)
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: whale-say-container
            image: docker/whalesay
            command: ["cowsay","Hello Kubernetes Team"]
          restartPolicy: Never
```

1- Le travail est exécuté toutes les minutes.


```
kubectl apply -f apps/kubefiles/whalesay-cronjob.yaml

kubectl get pods
```

```
NAME                  READY   STATUS      RESTARTS   AGE
```

Aucun Pod n'est en cours d'exécution car le CronJob est exécuté après 1 minute.

```
kubectl get cronjobs
```

```
NAME                SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
whale-say-cronjob   */1 * * * *   False     0        <none>          34s
```

Attendez une minute :

```
kubectl get pods
```

```
NAME                                 READY   STATUS      RESTARTS   AGE
whale-say-cronjob-1593436740-z9tf2   0/1     Completed   0          23s
```

```
kubectl get cronjobs
```

```
NAME                SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
whale-say-cronjob   */1 * * * *   False     0        48s             3m41s
```

Remarquez qu'un champ important est le Last Schedule, qui nous indique quand un travail a été exécuté pour la dernière fois.

Il est important de noter qu'un CronJob crée un travail :


```
kubectl get jobs
```

```
NAME                           COMPLETIONS   DURATION   AGE
whale-say-cronjob-1593436800   1/1           3s         44s
```

Supprimer les ressources

```
kubectl delete -f apps/kubefiles/whalesay-cronjob.yaml
```
