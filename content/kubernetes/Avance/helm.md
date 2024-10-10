---
title: "Helm"
date: 2020-06-26T15:17:20+02:00
draft: false
weight: 6
---

## Prérequis

- HELM [Install](https://helm.sh/fr/docs/intro/install/)  

Vous pouvez valider votre installation en exécutant :

```
helm version
```

> [!INFO]
> Ce tutoriel a été testé avec succès en utilisant une Helmversion supérieure à 3.7.0.

### Notions de base et principes fondamentaux


Helm est un packager pour Kubernetes qui regroupe les fichiers manifestes associés et les empaquete dans une seule unité de déploiement logique : Chart. En termes simples, Helm permet à de nombreux ingénieurs de commencer facilement à utiliser Kubernetes avec des applications réelles.

Les charts Helm sont utiles pour gérer les complexités d'installation et les mises à niveau simples d'applications particulièrement sans état, comme les applications Web. Dites adieu aux nombreux fichiers yaml longs et codés en dur et adoptez un moyen plus simple de gérer vos applications déployées !

#### Initialiser un référentiel Helm

Vous pouvez trouver un chart avec le logiciel souhaité dans l'[Artifact Hub](https://artifacthub.io/) , paramétrer ses ressources via Helm pour le déployer sur Kubernetes et cela fait éventuellement apparaître ledit logiciel.

Parfois, votre microservice conservera ses données dans une base de données et dans cette section, nous utiliserons Helm pour installer une instance de base de données PostgreSQL à partir d'un référentiel de charts. Un référentiel de charts est un serveur HTTP qui héberge un fichier index.yaml et éventuellement des charts empaquetés.

Vérifiez d’abord si le dépôt Helm https://charts.bitnami.com/bitnamiest déjà présent :

```
helm repo list
```

Si le dépôt n'est pas présent, veuillez exécuter les commandes suivantes pour l'ajouter :

```
helm repo add bitnami https://charts.bitnami.com/bitnami
```


> [!INFO]
> Assurez-vous que nous obtenons la dernière liste des helm en exécutant une mise à jour périodique des dépôts à l'aide de helm repo update

#### Installer un Helm Chart

Créez un nouvel namespace :

```
kubectl create ns dev

#permanently save the namespace for all subsequent kubectl commands
kubectl config set-context --current --namespace=dev
```

Déployez une instance de base de données dans votre espace de noms avec Helm, à l'aide de la commande suivante :


```
helm install faq-db --set global.postgresql.auth.username=faq-default,global.postgresql.auth.password=postgres,global.postgresql.auth.database=faq,primary.persistence.enabled=false \
--version 12.1.2  bitnami/postgresql
```

#### Validez votre installation

Lorsqu'un chart est installé, la bibliothèque Helm crée une version pour suivre cette installation. Vous pouvez valider l'installation via la listcommande pour voir les versions déployées ou ayant échoué.

```
helm list
```

Vous devriez voir quelque chose de similaire à :

```
NAME  	NAMESPACE     	REVISION	UPDATED                              	STATUS  	CHART             	APP VERSION
faq-db	dev	1       	2021-09-20 09:30:55.615499 +0200 CEST	deployed	            postgresql-12.1.2	15.1.0
```

Si vous souhaitez voir des informations sur les notes, les hooks, les valeurs fournies et le fichier manifeste généré de la version donnée, veuillez utiliser la commande suivante :

```
helm get all faq-db
```

Le résultat sera détaillé et similaire à :

```
NAME: faq-db
LAST DEPLOYED: Mon Nov 28 11:43:48 2022
NAMESPACE: asotobue-dev
STATUS: deployed
REVISION: 1
TEST SUITE: None
USER-SUPPLIED VALUES:
global:
  postgresql:
    auth:
      database: faq
      password: postgres
      username: faq-default
primary:
  containerSecurityContext:
    enabled: false
  persistence:
    enabled: false
  podSecurityContext:
    enabled: false

COMPUTED VALUES:
.....
```

Vous pouvez utiliser le sélecteur d’étiquettes suivant pour rechercher des ressources gérées par Helm dans un espace de noms donné :

```
kubectl get all -l='app.kubernetes.io/managed-by=Helm'
```

### Configurez vos premiers Chart

Si vous souhaitez obtenir une gestion des versions flexible et simple lors du déploiement sur Kubernetes, vous pouvez créer des charts Helm pour vos microservices. La technique du chart Helm par microservice peut vous aider à empaqueter rapidement votre application.


#### Créer un Helm Chart


```
mkdir chart
cd chart
helm create faq
cd faq
```

La commande ci-dessus génère la structure suivante pour votre chart :

```
├── Chart.yaml 
├── charts 
├── templates 
│   ├── NOTES.txt 
│   ├── _helpers.tpl 
│   ├── deployment.yaml
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── service.yaml
│   ├── serviceaccount.yaml
│   └── tests 
│       └── test-connection.yaml
└── values.yaml 
```

* Le fichier Chart contient une description du chart.
* Le répertoire charts peut contenir d’autres chart.
* Le répertoire templates/ contient tous les fichiers modèles utilisés pour l'installation d'un chart.
* Le fichier values.yaml contient les valeurs par défaut d'un chart.
* Le répertoire tests peut contenir des tests à exécuter à différentes étapes de l'installation des chart.
* Le fichier NOTES.txt peut contenir des instructions d'installation de charts qui seront affichées aux utilisateurs lors de l'exécution helm install.
* _helpers.tpl c'est là que vous pouvez placer des aides de modèle que vous pouvez réutiliser dans tout le chart.


#### Modifier le modèle associé au déploiement

Accédez à templates/deployment.yaml. Les clés associées aux valeurs définies dans values.yaml doivent être utilisées dans les modèles correspondants à l'aide de directives de modèle.

Modifiez la section image en ajoutant un port conteneur paramétrable :

```
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.image.containerPort }}
```

Étant donné que le microservice utilise également une base de données PostgreSQL, ajoutons ses détails :

```
          env:
            - name: POSTGRES_SERVER
              value: {{ .Values.postgresql.server | default (printf "%s-postgresql" ( .Release.Name )) | quote }} 
            - name: POSTGRES_USERNAME
              value: {{ .Values.postgresql.postgresqlUsername | default (printf "postgres" ) | quote }}
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: {{ .Values.postgresql.secretName | default (printf "%s-postgresql" ( .Release.Name )) | quote }}
                  key: {{ .Values.postgresql.secretKey }}
```

Si la valeur pour le serveur PostgreSQL est définie dans values.yaml, celle-ci sera renseignée. Sinon, une chaîne composée du nom de la version et se terminant par postgresql sera renseignée.

Veuillez noter que le mot de passe de la base de données n'est pas obtenu directement à partir de values.yaml, mais référencé à partir du secret qui contient le mot de passe réel.

Enfin et surtout, les sondes doivent être adaptés afin d'utiliser les clés définies dans values.yaml. Modifiez les sondes à l'aide des éléments suivants :

```
          readinessProbe:
            httpGet:
              path: {{ .Values.readinessProbe.path}}
              port: {{ .Values.service.port }}
            initialDelaySeconds: {{ .Values.readinessProbe.initialDelaySeconds}}
            timeoutSeconds: {{ .Values.readinessProbe.timeoutSeconds}}
            periodSeconds: {{ .Values.readinessProbe.periodSeconds}}
            failureThreshold: {{ .Values.readinessProbe.failureThreshold }}
          livenessProbe:
            httpGet:
              path: {{ .Values.livenessProbe.path}}
              port: {{ .Values.service.port }}
            initialDelaySeconds: {{ .Values.livenessProbe.initialDelaySeconds}}
            timeoutSeconds: {{ .Values.livenessProbe.timeoutSeconds}}
            periodSeconds: {{ .Values.livenessProbe.periodSeconds }}
            failureThreshold: {{ .Values.livenessProbe.failureThreshold}}
```

#### Personnalisation values.yaml

Votre values.yaml est l'endroit où vous pouvez spécifier des valeurs pour différents paramètres utilisés dans les modèles.

Modifiez le fichier values.yaml avec les valeurs d'image suivantes :

```
image:
  repository: quay.io/evanshortiss/helm-faq
  tag: "1.0.1"
  pullPolicy: IfNotPresent
  containerPort: 8080
```

En utilisant les détails de la section précédente lors de l'installation de la base de données, vous pouvez paramétrer la connexion à la base de données. Ajoutez la section postgresql suivante au fichier values.yaml :

```
postgresql:
  server: faq-db-postgresql
  postgresqlUsername: faq-default
  secretName: faq-db-postgresql
  secretKey: password
```

Ajoutez des valeurs de contrôle de santé pour les sondes :


```
readinessProbe:
  path: /q/health/ready
  initialDelaySeconds: 5
  timeoutSeconds: 3
  periodSeconds: 3
  failureThreshold: 3


livenessProbe:
  path: /q/health/live
  initialDelaySeconds: 10
  timeoutSeconds: 2
  periodSeconds: 8
  failureThreshold: 3
```

Votre application déployée doit être accessible depuis l'intérieur et l'extérieur du cluster Kubernetes. Un service Kubernetes de type LoadBalancersera utilisé pour cette installation.

Remplacez les valeurs de service qui exposeront votre microservice par les suivantes :


```
service:
  type: LoadBalancer
  port: 8080
```

#### Déployer votre chart modifiées


Installez maintenant simplement vos chart en utilisant :


```
helm install simple ./
```

Vérifiez l'état de votre installation et obtenez les détails en exécutant :

```
helm status simple 
helm get all simple 
kubectl get svc/simple-faq 
```

Une fois l'application déployée, vous pouvez accéder au service /ask pour poser une question.


```
POD_NAME=$(kubectl get pods -l app.kubernetes.io/name=faq -o name)
kubectl exec $POD_NAME -- /bin/curl -s localhost:8080/ask/
```


```
[{"title":"Existence","region":"BeNeLux","text":"Are you there?"},{"title":"Existence","region":"CEE","text":"Why do we dream?"}]
```

Félicitations , vous pouvez désormais consulter les questions fréquemment posées !

### Chart par approche de service

Utilisez une approche de diagramme Helm par microservice si vous cherchez à obtenir un contrôle de version flexible et simple et des diagrammes de faible complexité pour empaqueter votre application. Soyez prudent, cette technique peut entraîner :

* énorme quantité de duplication
* difficulté de maintenir la cohérence de nombreux modèles
* difficulté d'introduire des changements globaux


#### Injecter des configurations à partir d'un modèle de configmap

Lorsque vous travaillez avec un microservice, il est probable que vous injectiez et gériez des configurations à l’aide d’autres ressources Kubernetes, comme ConfigMap ou Secret.

Injectons une configuration à partir d'un ConfigMap pour le microservice du didacticiel. Accédez au dossier templates et créez un configmap.yaml. Ouvrez le configmap.yaml que vous venez de créer et ajoutez les éléments suivants :

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Values.configmap.region | quote }}
  labels:
    {{- include "faq.labels" . | nindent 4 }}
data:
  region: {{.Values.configmap.region | upper | quote }}
```

Ajoutez maintenant la valeur par défaut pour une région dansvalues.yaml :


```
configmap:
  key: region
  region: cee
```


Et configurez le deployment.yamlmodèle pour utiliser le nouveau configmap :


```
          env:
            - name: POSTGRES_SERVER
              value: {{ .Values.postgresql.server | default (printf "%s-postgresql" ( .Release.Name )) | quote }}
            - name: POSTGRES_USERNAME
              value: {{ .Values.postgresql.postgresqlUsername | quote }}
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: {{ .Values.postgresql.secretName | default (printf "%s-postgresql" ( .Release.Name )) | quote }}
                  key: {{ .Values.postgresql.secretKey }}
            - name: REGION
              valueFrom:
                configMapKeyRef:
                  name: {{ .Values.configmap.region }}
                  key: {{ .Values.configmap.key }}
```



#### Déployer automatiquement les déploiements lorsqu'un ConfigMap/Secret change

Les Configmaps et les Secrets sont souvent utilisés comme fichiers de configuration, mais lorsque les valeurs de ces fichiers changent, elles ne sont pas automatiquement récupérées par une application en cours d'exécution, sauf si la spécification de déploiement elle-même change. Cela peut potentiellement entraîner des déploiements incohérents où les valeurs ont été mises à jour, mais l'application est toujours en cours d'exécution avec l'ancienne configuration, ce qui n'est pas ce que nous souhaitons.

Une solution à ce problème consiste à utiliser une fonction sha256sum qui peut être utilisée pour garantir que la section d'annotation d'un déploiement est mise à jour si un autre fichier change :

> [!IMPORTANT]
> Ne modifiez pas le fichier de déploiement.yaml. L'extrait suivant n'est qu'un exemple.

```
kind: Deployment
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```

Nous pouvons rendre ce changement encore plus élégant, car Helm a généré automatiquement l'inclusion des annotations lorsque deployment.yaml nous avons initialisé notre chart :


```
  template:
    metadata:
      {{- with .Values.podAnnotations }}
      annotations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

Ce qui précède indique ce qui suit : seulement si .Values.podAnnotations est défini, alors obtenez la définition yaml à partir de là.

Nous pouvons ainsi modifier values.yamlen conséquence :

> [!INFO]
> La clé podAnnotations est peut-être déjà présente dans le fichier values.yaml . Mettez-la à jour si elle existe ou créez une nouvelle entrée si elle est manquante.

podAnnotations:
  checksum/config: '{{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}'


Comme la valeur ci-dessus contient à la fois YAML et un modèle Go, nous devons analyser le yaml ( toYaml) et utiliser le modèle ( tpl) avec deployment.yaml :

```
  template:
    metadata:
      {{- with .Values.podAnnotations }}
      annotations:
      {{ tpl (toYaml .) $ | nindent 8 }}
      {{- end }}
```

#### Mettre à jour la version

Mettre à jour la version existante avec les configurations nouvellement ajoutées :

```
helm upgrade simple ./
```

Exécutez les commandes suivantes pour vérifier la sortie CEE :


```
POD_NAME=$(kubectl get pods -l app.kubernetes.io/name=faq -o name)
kubectl exec $POD_NAME -- /bin/curl -s localhost:8080/ask/CEE
```

La sortie doit être un tableau JSON contenant un objet avec la région définie sur une version en majuscules de cee dans configmap values.yaml .

Exécutez simplement la commande suivante pour désinstaller vos versions précédentes :

```
helm uninstall simple
helm uninstall faq-db
```

#### Répliquer l'installation dans un namespace différent 

Créons un namespace différent :

```
kubectl create ns qa

#permanently save the namespace for all subsequent kubectl commands
kubectl config set-context --current --namespace=qa
```

Et définissez un ensemble de valeurs pour ce namespace en faisant une copie de values.yaml. Nommez la copie values.qa.yaml. Modifiez la région à l'intérieur values.qa.yaml:

```
configmap:
  key: region
  region: benelux
```

Et maintenant, installez les charts :

```
helm install faq-db \
--set postgresqlUsername=faq-default,postgresqlPassword=postgres,postgresqlDatabase=faq,persistence.enabled=false,securityContext.enabled=false,containerSecurityContext.enabled=false \
bitnami/postgresql 

helm install simple ./ --values values.qa.yaml
```

Exécutez simplement la commande suivante pour désinstaller vos versions précédentes :

```
helm uninstall simple
helm uninstall faq-db
```