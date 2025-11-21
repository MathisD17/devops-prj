# devops-prj

Projet DevOps de déploiement continu d'une application web 3-tiers sur Kubernetes (AKS), avec pipeline CI/CD et supervision.

Ce projet est réalisé dans le cadre du titre professionnel Administrateur Système DevOps (ASD).
Il vise à valider principalement les CCP2 (Déployer en continu une application) et CCP3 (Superviser les services déployés).

## Exécution de l'application

### 1. Exécution avec Docker Compose (environnement local)

L'application peut être lancée localement via Docker Compose, incluant :

- **MySQL** (base de données)
- **Backend Node.js / Express**
- **Frontend Angular (servi par Nginx)**

Commande :

```bash
docker compose -f docker-compose.local.yml up --build
```

Accès aux services :

- **Backend (Swagger)** : http://localhost:3000/api/docs  
- **Frontend Angular** : http://localhost:4200  

---

### 2. Déploiement Kubernetes (cluster local Docker Desktop)

Les manifestes Kubernetes se trouvent dans :

```
k8s/local/
  ├─ namespace.yaml
  ├─ mysql.yaml
  ├─ backend.yaml
  └─ frontend.yaml
```

Déploiement :

```bash
kubectl apply -f k8s/local/namespace.yaml
kubectl apply -f k8s/local/mysql.yaml
kubectl apply -f k8s/local/backend.yaml
kubectl apply -f k8s/local/frontend.yaml
```

Accès à l'application dans Kubernetes :

- **Frontend** : http://localhost:30080  

Images utilisées :

- `devops-prj-backend:latest`
- `devops-prj-frontend:latest`

👉 **Note :** les manifestes utilisent `imagePullPolicy: Never` pour exploiter les images Docker locales du poste.

---

### Prochaines étapes prévues

- Ajout d’un **Ingress Controller** (Nginx Ingress)  
- Déploiement dans **Azure Kubernetes Service (AKS)** via Terraform  
- Mise en place d’un pipeline **CI/CD GitHub Actions**  
- Ajout d’une stack **supervision (Prometheus + Grafana)**
