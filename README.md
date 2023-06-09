# crossplane-anthos
Ressource d'un workshop de Devoxx pour nous présenter Crossplane et Anthos.<br/>
GitLab: https://gitlab.com/musk8teers/crossplane-anthos/

## Speaker
[Ludovic Piot](https://gitlab.com/ludovic-piot)


## Provisionning du 1er *cluster* `Anthos` depuis `CrossPlane`

### Démarrage de `Minikube`

:warning: `CrossPlane` est assez gourmand en ressources. À moins de 4 Go de RAM, `Minikube` s'avère instable.

```bash
minikube start
kubectl get nodes
kubectl get namespaces
```

### Poser des variables d'environnement

```bash
gcloud config set core/project crossplane-anthos
export GCP_ZONE=us-central1-a
export GCP_REGION=us-central1
export GCP_PROJECT_ID=`gcloud config get-value core/project`
export GCP_PROJECT_NUMBER=`gcloud projects describe ${GCP_PROJECT_ID} --format json | jq --raw-output '.projectNumber'`
```

### Création d'un service account `GCP`

On créé un *service account*, qui sera utilisé par les outils d'*Infra-as-Code*.

```bash
GCP_SERVICE_ACCOUNT_NAME=crossplane-anthos-sa
GCP_SERVICE_ACCOUNT="${GCP_SERVICE_ACCOUNT_NAME}@${GCP_PROJECT_ID}.iam.gserviceaccount.com"
gcloud iam service-accounts create ${GCP_SERVICE_ACCOUNT_NAME} --project ${GCP_PROJECT_ID}
```

On donne les droits nécessaires au *service account*.

```bash
# role for GCP bucket creation
gcloud projects add-iam-policy-binding --role="roles/storage.admin" ${GCP_PROJECT_ID} --member "serviceAccount:${GCP_SERVICE_ACCOUNT}"

# role for GKE cluster creation
gcloud projects add-iam-policy-binding --role="roles/container.clusterAdmin" ${GCP_PROJECT_ID} --member "serviceAccount:${GCP_SERVICE_ACCOUNT}"
gcloud projects add-iam-policy-binding --role="roles/iam.serviceAccountUser" ${GCP_PROJECT_ID} --member "serviceAccount:${GCP_SERVICE_ACCOUNT}"
gcloud projects add-iam-policy-binding --role="roles/compute.instanceAdmin.v1" ${GCP_PROJECT_ID} --member "serviceAccount:${GCP_SERVICE_ACCOUNT}"

# role for GKE cluster registration into an Anthos fleet
gcloud projects add-iam-policy-binding --role="roles/gkehub.admin" ${GCP_PROJECT_ID} --member "serviceAccount:${GCP_SERVICE_ACCOUNT}"

# role for Cloud SQL creation
gcloud projects add-iam-policy-binding --role="roles/cloudsql.admin" ${GCP_PROJECT_ID} --member "serviceAccount:${GCP_SERVICE_ACCOUNT}"
```

On génère une clé secrète pour ce *service account*.

```bash
gcloud iam service-accounts keys create service_account_credentials.json --project ${GCP_PROJECT_ID} --iam-account ${GCP_SERVICE_ACCOUNT}
```

On créé un *secret* dans `K8s` pour stocker la clé du *service account* `GCP`.

```bash
kubectl create secret generic gcp-service-account-credentials \
    --namespace="crossplane-system" \
    --from-file=file-content=${HOME}/crossplane-anthos/.credentials/service_account_credentials.json
```

### Enable `GCP` *API*

🚧 TBD

### Installation de `CrossPlane` dans `MiniKube`

On installe `CrossPlane` via `Helm`.

```bash
kubectl create namespace crossplane-system
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update
helm install crossplane --namespace crossplane-system crossplane-stable/crossplane

# On vérifie que tout a été installé correctement.
helm list -n crossplane-system
kubectl get all -n crossplane-system
```

On installe la *CLI* `CrossPlane` (il s'agit d'un plugin à `kubectl`).

```bash
curl -sL https://raw.githubusercontent.com/crossplane/crossplane/master/install.sh | sh
sudo mv kubectl-crossplane /home/ludovic_piot/.local/bin
kubectl crossplane --help
```

### Visualisation des resources créées par CrossPlane

```bash
kubectl get crossplane
```

# 1er provisionning de ressource `GCP` avec `CrossPlane`

Avec `CrossPlane`, on va provisionner une 1ère ressource `GCP` : un simple *bucket* `Google Cloud Storage` en chargeant les *manifests* `CrossPlane` dans `Minikube`.  

```bash
cd ${HOME}/crossplane-anthos/infrastructure/bootstrap

# Installation des providers GCP
kubectl apply -f ./providers.yaml
kubectl get providers

# Installation de la configuration du provider GCP avec fourniture du secret
kubectl apply -f ./provider-config.yaml

# Déploiement du bucket
kubectl apply -f ./bucket.yaml
```

### *Bootstrap* de la `Anthos` *fleet* avec un 1er *cluster* `GKE`

On provisionne un 1er *cluster* `GKE` que l'on enregistre à la `Anthos` *fleet*.

```bash
# Déploiement du cluster GKE et enregistrement dans la Anthos fleet
kubectl apply -f gke-cluster-1.yaml
kubectl get cluster.container.gcp.upbound.io

gcloud container hub memberships list
gcloud container fleet features list
```

Pour que ce *cluster* puisse s'auto-configurer, on a besoin de fournir une source de configuration *GitOps* à l'outil `Anthos Config Management` (`Config-Sync`).  
Pour cela, on va utiliser le dossier `anthos-config` de notre dépôt `Git`.  
Côté Gitlab, on crée un *personal access token*… que l'on stocke dans un *secret* sur le cluster `GKE`.  

```bash
# On configure kubectl pour qu'il s'adresse à notre cluster GKE
gcloud container clusters get-credentials anthos-gke-cluster-1 --zone ${GCP_ZONE}
kubectl config get-contexts
kubectl get nodes
```

```bash
kubectl create ns config-management-system
kubectl create secret generic git-creds \
  --namespace="config-management-system" \
  --from-literal=username=ludovic-piot \
  --from-literal=token=$(cat ${HOME}/crossplane-anthos/.credentials/gitlab-access-token)
```

On configure la source de vérité de `Config-Sync` depuis le dépôt `Gitlab`

```bash
cd ${HOME}
gcloud beta container fleet config-management apply                       \
      --membership=anthos-gke-cluster-1                                   \
      --config=crossplane-anthos/infrastructure/bootstrap/apply-spec.yaml \
      --project=crossplane-anthos


gcloud beta container fleet config-management status \
    --project=crossplane-anthos
```


# Sources

- Google Kubernetes Engine with Crossplane - https://ce.qwiklabs.com/classrooms/9396/labs/65084
- Une explication de la cinématique GitOps avec Anthos, Crossplane et ArgoCD -  https://medium.com/@vincn.ledan/construisez-une-plateforme-moderne-avec-crossplane-anthos-et-argocd-lavenir-de-la-gestion-a73e8631afd8
- https://www.googlecloudcommunity.com/gc/Cloud-Events/Managing-multicloud-deployments-with-Anthos-and-Crossplane/ec-p/495591
- CrossPlane XRD and Composition - https://prune998.medium.com/playing-with-crossplane-for-real-f591e66065ae


## Installation de krew

https://krew.sigs.k8s.io/docs/user-guide/setup/install/

## Installation de krew/get-all

https://github.com/corneliusweig/ketall


