# devops-prj

Projet DevOps de déploiement continu d'une application web 3-tiers sur Kubernetes (AKS), avec pipeline CI/CD et supervision.

Ce projet est réalisé dans le cadre du titre professionnel **Administrateur Système DevOps (ASD)**.
Il vise à valider principalement :
- **CCP2** : Déployer en continu une application  
- **CCP3** : Superviser les services déployés

---

## 1. Exécution locale avec Docker Compose

L'application peut être lancée localement via Docker Compose, incluant :

- **MySQL** (base de données)
- **Backend Node.js / Express**
- **Frontend Angular (servi par Nginx)**

Commande (à la racine du projet) :

```bash
docker compose -f docker-compose.local.yml up --build
```

Accès aux services :

- **Backend (Swagger)** : http://localhost:3000/api/docs  
- **Frontend Angular** : http://localhost:4200  

### Remarque

Les images Docker utilisées sont :

- `devops-prj-backend:latest`
- `devops-prj-frontend:latest`

Elles sont construites à partir des Dockerfile présents dans :

- `app/backend/Dockerfile`
- `app/frontend/Dockerfile`

---

## 2. Déploiement Kubernetes en local (Docker Desktop)

Les manifestes Kubernetes pour l'environnement local se trouvent dans :

```text
k8s/local/
  ├─ namespace.yaml
  ├─ mysql.yaml
  ├─ backend.yaml
  ├─ frontend.yaml
  └─ ingress.yaml
```

### Déploiement

```bash
kubectl apply -f k8s/local/namespace.yaml
kubectl apply -f k8s/local/mysql.yaml
kubectl apply -f k8s/local/backend.yaml
kubectl apply -f k8s/local/frontend.yaml
kubectl apply -f k8s/local/ingress.yaml
```

### Accès via Ingress (en local)

Après configuration de l'entrée DNS (`hosts` Windows) pointant `devops-prj.local` vers `127.0.0.1` :

- **Frontend** : http://devops-prj.local/  
- **Backend (Swagger)** : http://devops-prj.local/api/docs  

L'ingress route les requêtes `/` vers le frontend et `/api` vers le backend.

---

## 3. Provisioning de l’infrastructure Azure avec Terraform

L'infrastructure Azure (groupe de ressource, ACR, AKS, réseau) est décrite dans :

```text
infra/terraform/
  ├─ main.tf
  ├─ providers.tf
  ├─ variables.tf
  ├─ aks.tf
  └─ outputs.tf
```

### Prérequis

- Azure CLI connecté :  
  ```bash
  az login
  az account set --subscription "<ID_DE_LA_SUBSCRIPTION>"
  ```

### Commandes Terraform

Depuis `infra/terraform` :

```bash
terraform init
terraform plan
terraform apply
```

Les ressources principales créées sont :

- **Resource Group** : `rg-devops-prj`
- **Azure Container Registry (ACR)** : `acrdevopsprj`
- **Azure Kubernetes Service (AKS)** : `aks-devops-prj`

---

## 4. Déploiement sur AKS

### 4.1 Récupération du contexte AKS

Après `terraform apply` :

```bash
az aks get-credentials -g rg-devops-prj -n aks-devops-prj --overwrite-existing
kubectl get nodes
```

### 4.2 Build & push des images vers ACR (manuel)

Depuis la racine du projet :

```bash
# Build des images
docker build -t devops-prj-backend:latest ./app/backend
docker build -t devops-prj-frontend:latest ./app/frontend

# Connexion à l’ACR
az acr login -n acrdevopsprj

# Tag vers l’ACR
docker tag devops-prj-backend:latest  acrdevopsprj.azurecr.io/devops-prj-backend:latest
docker tag devops-prj-frontend:latest acrdevopsprj.azurecr.io/devops-prj-frontend:latest

# Push
docker push acrdevopsprj.azurecr.io/devops-prj-backend:latest
docker push acrdevopsprj.azurecr.io/devops-prj-frontend:latest
```

### 4.3 Manifests Kubernetes pour AKS

Les manifestes dédiés à AKS se trouvent dans :

```text
k8s/aks/
  ├─ namespace.yaml
  ├─ mysql.yaml
  ├─ backend.yaml
  └─ frontend.yaml
```

Déploiement :

```bash
kubectl apply -f k8s/aks/namespace.yaml
kubectl apply -f k8s/aks/mysql.yaml
kubectl apply -f k8s/aks/backend.yaml
kubectl apply -f k8s/aks/frontend.yaml
```

### 4.4 Accès à l’application sur AKS

Le frontend est exposé via un service de type **LoadBalancer** dans le namespace `devops-prj` :

```bash
kubectl get svc -n devops-prj
```

Une IP publique est attribuée au service `frontend`.  
L’application est alors accessible via :

```text
http://<IP-PUBLIC>/
```

---

## 5. Pipeline CI/CD GitHub Actions

Un pipeline CI/CD permet d’automatiser :

- le build des images Docker backend et frontend
- le push vers l’ACR Azure
- le déploiement sur le cluster AKS via `kubectl apply`

### 5.1 Workflow GitHub Actions

Le workflow est défini dans :

```text
.github/workflows/ci-cd.yml
```

Principales étapes :

1. **Checkout** du code
2. **Login** à l’ACR via les secrets GitHub
3. **Build** des images :
   - `${ACR_LOGIN_SERVER}/devops-prj-backend:latest`
   - `${ACR_LOGIN_SERVER}/devops-prj-frontend:latest`
4. **Push** des images vers l’ACR
5. **Configuration de kubectl** via un kubeconfig fourni en secret
6. **Déploiement** des manifests `k8s/aks/` sur AKS

### 5.2 Secrets GitHub à définir

Dans le dépôt GitHub, onglet **Settings → Secrets and variables → Actions** :

- `ACR_LOGIN_SERVER` : `acrdevopsprj.azurecr.io`
- `ACR_USERNAME` : username de l’ACR (`az acr credential show`)
- `ACR_PASSWORD` : password de l’ACR
- `KUBECONFIG_B64` : contenu base64 d’un kubeconfig généré via :

  ```bash
  az aks get-credentials -g rg-devops-prj -n aks-devops-prj --admin --file aks-kubeconfig
  ```

  puis encodage base64 du fichier `aks-kubeconfig`.

À chaque push sur la branche `main`, le workflow reconstruit et déploie automatiquement l’application sur AKS.

---

## 6. Supervision & Alerting (CCP3)

### 6.1 Installation de la stack monitoring (kube-prometheus-stack)

La supervision du cluster AKS est réalisée avec **kube-prometheus-stack** (Prometheus, Grafana, Alertmanager), installée via Helm dans le namespace `monitoring` :

```bash
kubectl create namespace monitoring

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring
```

### 6.2 Accès à Grafana

Le service Grafana est exposé en LoadBalancer :

```bash
kubectl patch svc monitoring-grafana -n monitoring -p '{"spec":{"type":"LoadBalancer"}}'
kubectl get svc -n monitoring monitoring-grafana
```

Une IP publique est attribuée, par exemple :

```text
http://<GRAFANA-IP-PUBLIC>/
```

Les identifiants par défaut sont :

- utilisateur : `admin`
- mot de passe : récupéré via :

  ```bash
  kubectl get secret monitoring-grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 -d
  ```

### 6.3 Dashboards

Grafana propose des dashboards préconfigurés permettant de superviser :

- l’état du cluster Kubernetes (nœuds, ressources)
- les namespaces et pods, notamment **`devops-prj`** (backend, frontend, MySQL)

Un dashboard personnalisé **“DevOps Project – devops-prj”** a été ajouté pour suivre :

- la consommation CPU par pod
- la mémoire utilisée par pod

### 6.4 Règle d’alerte personnalisée : BackendDown

Une règle d’alerte Prometheus spécifique au backend de l’application a été ajoutée via une ressource `PrometheusRule` :

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: devops-prj-rules
  namespace: monitoring
  labels:
    release: monitoring
spec:
  groups:
  - name: devops-prj.rules
    rules:
    - alert: BackendDown
      expr: kube_deployment_status_replicas_available{namespace="devops-prj",deployment="backend"} < 1
      for: 30s
      labels:
        severity: warning
      annotations:
        summary: "Le backend n’a aucun replica disponible"
        description: "Le déploiement backend n’a aucun pod en état 'Ready' depuis plus de 30 secondes."
```

Application de la règle :

```bash
kubectl apply -f k8s/monitoring/devops-prj-alerts.yaml
```

### 6.5 Test de l’alerte BackendDown

Pour simuler une indisponibilité du backend :

```bash
kubectl scale deployment backend -n devops-prj --replicas=0
```

Après environ 30 secondes, l’alerte **BackendDown** passe en état **Firing** dans l’interface Prometheus / Alertmanager.

Pour restaurer la situation :

```bash
kubectl scale deployment backend -n devops-prj --replicas=1
```

L’alerte repasse alors à l’état **Resolved**.

---

## 7. Résumé des objectifs atteints

Ce projet démontre :

- la capacité à automatiser le déploiement d’une application 3-tiers via **Docker** et **Kubernetes** (local + AKS),
- la mise en place d’un pipeline **CI/CD GitHub Actions** pour un déploiement continu sur le Cloud,
- l’utilisation de **Terraform** pour le provisionnement d’infrastructure,
- l’exploitation d’une stack de **supervision** (Prometheus, Grafana, Alertmanager) avec création :
  - de dashboards dédiés à l’application,
  - d’une **règle d’alerte personnalisée** sur le backend.

Ces éléments s’inscrivent directement dans les exigences des CCP2 et CCP3 du titre professionnel ASD.

---

## 8. Limites et axes d’amélioration

### HTTPS sur AKS via Ingress

La mise en place d’un Ingress HTTPS avec certificat TLS a été préparée :

- déploiement d’un **Ingress Controller NGINX** dans le namespace `ingress-nginx`,
- génération d’un certificat TLS auto-signé et d’un secret Kubernetes `devops-prj-tls`,
- création d’une ressource `Ingress` pointant vers le service `frontend`.

Cependant, le déploiement final d’un service `LoadBalancer` dédié à l’Ingress n’a pas pu aboutir, en raison d’une **limitation de l’abonnement Azure** :

> `PublicIPCountLimitReached: Cannot create more than 3 public IP addresses for this subscription in this region.`

Dans un environnement de production (ou avec un quota IP publiques plus élevé),  
une amélioration naturelle serait :
- d’exposer l’application via un **Ingress HTTPS** unique (TLS),
- et de supprimer les `LoadBalancer` redondants au profit de cet Ingress.
