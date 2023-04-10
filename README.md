# crossplane-anthos



## Démarrage de Minikube

minikube start
kubectl get nodes
kubectl get namespaces


## Prérequis

### Poser des variables d'environnement

```bash
gcloud config set core/project crossplane-anthos
export GCP_ZONE=us-central1-a
export GCP_REGION=us-central1
export GCP_PROJECT_ID=`gcloud config get-value core/project`
```

### Création d'un service account GCP

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
# role for Cloud Source Repositories creation
gcl```

On génère une clé secrète pour ce *service account*.

```bash
gcloud iam service-accounts keys create service_account_credentials.json --project ${GCP_PROJECT_ID} --iam-account ${GCP_SERVICE_ACCOUNT}
```

On créé un secret dans K8s pour stocker la clé du *service account* GCP.

```bash
kubectl create secret generic gcp-service-account-credentials -n crossplane-system --from-file=file-content=./.credentials/service_account_credentials.json
```

## Enable GKE API


## Installation de CrossPlane

On installe crossplane via `Helm`.
```bash
kubectl create namespace crossplane-system
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update
helm install crossplane --namespace crossplane-system crossplane-stable/crossplane
```

On vérifie que tout a été installé correctement.

```bash
helm list -n crossplane-system
kubectl get all -n crossplane-system
```

On installe la CLI `CrossPlane` (il s'agit d'un plugin à `kubectl`).

```bash
curl -sL https://raw.githubusercontent.com/crossplane/crossplane/master/install.sh | sh
```



# Sources

- Google Kubernetes Engine with Crossplane - https://ce.qwiklabs.com/classrooms/9396/labs/65084
- Une explication de la cinématique GitOps avec Anthos, Crossplane et ArgoCD -  https://medium.com/@vincn.ledan/construisez-une-plateforme-moderne-avec-crossplane-anthos-et-argocd-lavenir-de-la-gestion-a73e8631afd8
- https://www.googlecloudcommunity.com/gc/Cloud-Events/Managing-multicloud-deployments-with-Anthos-and-Crossplane/ec-p/495591


## Installation de krew

https://krew.sigs.k8s.io/docs/user-guide/setup/install/

## Installation de krew/get-all

https://github.com/corneliusweig/ketall

## Visualisation des resources créées par CrossPlane

kubectl get crossplane

