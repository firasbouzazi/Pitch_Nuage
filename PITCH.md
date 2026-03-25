# Guide de Présentation — Voting App

Application de vote déployée sur 3 couches : Docker Compose → Kubernetes (GKE) → Terraform.
Chaque personne présente son service à travers ces 3 couches.

---

## Personne 1 — Vote + Nginx (Frontend de vote)

### Rôle dans l'architecture
Le service `vote` est un serveur Flask (Python) qui affiche l'interface de vote et écrit les choix dans Redis.
`nginx` sert de load balancer devant `vote`, c'est le point d'entrée utilisateur sur le port 8000.

### Docker

**Dockerfile vote** (`vote/Dockerfile`) :
- Image `python:3.11-slim`, installe `curl` pour le healthcheck
- Copie `requirements.txt`, installe les dépendances pip, puis copie le code
- Lance `python app.py` sur le port 5000

**Dockerfile nginx** (`nginx/Dockerfile`) :
- Image `nginx`, remplace la config par défaut par `nginx.conf`
- `nginx.conf` définit un upstream qui balance vers `vote:5000`

**docker-compose.yaml** :
- `vote` est sur `front-net` + `back-net` (accès utilisateur + communication avec Redis)
- `nginx` est sur `front-net` uniquement, expose le port `8000:8000`
- `vote` dépend de `redis` (condition: `service_healthy`)
- `nginx` dépend de `vote` et `result`
- Healthcheck sur vote : `curl -f http://localhost:5000`

### Kubernetes

**vote-deployment.yaml** (`k8s/vote-deployment.yaml`) :
- 3 replicas, image depuis GCP Artifact Registry
- `envFrom: configMapRef: app-config` pour les variables (OPTION_A, OPTION_B, REDIS_HOST...)
- Ressources limitées : requests `10m CPU / 64Mi`, limits `250m / 256Mi`
- `livenessProbe` httpGet sur `/` port 5000

**vote-service.yaml** (`k8s/vote-service.yaml`) :
- Type `LoadBalancer` — accessible depuis l'extérieur sur le port 5000

**vote-hpa.yaml** (`k8s/vote-hpa.yaml`) :
- `HorizontalPodAutoscaler` : min 1, max 5 replicas
- Scale sur CPU à 50% d'utilisation moyenne
- Nécessite `minikube addons enable metrics-server` en local

**Communication** :
- vote → Redis (ClusterIP interne) : écrit les votes via `redis.rpush('votes', data)`
- Le code (`vote/app.py`) lit `REDIS_HOST` et `REDIS_PASSWORD` depuis les variables d'environnement (ConfigMap)
- nginx n'est pas déployé en K8s car le Service LoadBalancer remplace son rôle

### Terraform Docker (`terraform/modules/vote/main.tf`)
- `docker_image` build depuis `vote/` avec trigger sur le hash du code source (rebuild auto si le code change)
- `docker_container` sur `front-net` + `back-net`, healthcheck identique au compose
- `docker_registry_image` optionnel pour push vers Artifact Registry

### Terraform Docker (`terraform/modules/nginx/main.tf`)
- Build depuis `nginx/`, expose port 8000, réseau `front-net` uniquement

### Points clés à mentionner
- Le HPA permet de scaler automatiquement vote sous charge (seed envoie 3000 requêtes)
- nginx en Docker vs LoadBalancer en K8s : même rôle, implémentation différente
- `front-net` isole le trafic utilisateur du backend

---

## Personne 2 — Result (Frontend des résultats) — Firas

### Rôle dans l'architecture
Le service `result` est un serveur Node.js qui lit les résultats depuis PostgreSQL et les affiche en temps réel via Socket.IO.

### Docker

**Dockerfile** (`result/Dockerfile`) :
- Image `node:18-slim`, workdir `/usr/local/app`
- Installe `nodemon` globalement, puis `npm ci` + cache clean
- Déplace `node_modules` vers `/node_modules` (optimisation de layer Docker)
- Variable `PORT=4000`, expose le port 4000
- Lance `node server.js`

**docker-compose.yaml** :
- `result` est sur `front-net` + `back-net` (affichage utilisateur + lecture depuis PostgreSQL)
- Expose `4000:4000`
- Dépend de `db` (condition: `service_healthy`)

**server.js** (`result/server.js`) :
- Connexion PostgreSQL : `postgres://postgres:postgres@db/postgres`
  - `db` est résolu par DNS Docker/K8s vers le service PostgreSQL
- Query toutes les secondes : `SELECT vote, COUNT(id) AS count FROM votes GROUP BY vote`
- Émet les scores via Socket.IO en temps réel

### Kubernetes

**result-deployment.yaml** (`k8s/result-deployment.yaml`) :
- 1 replica, image depuis GCP Artifact Registry
- `envFrom: configMapRef: app-config` (PORT, POSTGRES_HOST, etc.)
- Ressources : requests `50m CPU / 64Mi`, limits `250m / 256Mi`
- `livenessProbe` httpGet sur `/` port 4000

**result-service.yaml** (`k8s/result-service.yaml`) :
- Type `LoadBalancer` — accessible depuis l'extérieur sur le port 4000
- Choix LoadBalancer (et non ClusterIP) car c'est un frontend utilisateur

**Communication** :
- result → db (ClusterIP interne) : le nom `db` est résolu par le DNS Kubernetes
- result ne communique PAS avec Redis — il lit uniquement depuis PostgreSQL
- Les résultats sont poussés au navigateur via WebSocket (Socket.IO)

### Terraform Docker (`terraform/modules/result/main.tf`)
- `docker_image` build depuis `result/` avec trigger sur hash du code source
- `docker_container` : port 4000, env `PORT=4000`, réseaux `front-net` + `back-net`
- `docker_registry_image` optionnel pour push vers le registry

### Terraform GCP (`terraform-gcp/k8s-manifests.tf`)
- Le manifest `k8s/result-deployment.yaml` est appliqué via `kubernetes_manifest`
- Terraform lit le YAML, injecte le namespace, et l'applique au cluster GKE

### Points clés à mentionner
- LoadBalancer vs ClusterIP : result est un frontend donc LoadBalancer
- La connexion à `db` fonctionne grâce au Service K8s `db` de type ClusterIP qui expose le port 5432
- Socket.IO pour le temps réel : pas de polling côté client, le serveur push les scores
- `subPath: data` sur le PVC de db garantit que les données persistent même si le pod db redémarre

---

## Personne 3 — Worker + Seed (Traitement backend)

### Rôle dans l'architecture
Le `worker` est un programme .NET qui lit les votes depuis Redis et les écrit dans PostgreSQL.
Le `seed` est un Job qui envoie massivement des votes pour tester la charge.

### Docker

**Dockerfile worker** (`worker/Dockerfile`) — multistage build :
- Stage 1 (`build`) : image `mcr.microsoft.com/dotnet/sdk:8.0` avec `--platform=${BUILDPLATFORM}`
  - ARG `TARGETPLATFORM`, `TARGETARCH`, `BUILDPLATFORM` pour le cross-compilation
  - `dotnet restore` puis `dotnet publish` vers `/app`
- Stage 2 : image `mcr.microsoft.com/dotnet/runtime:8.0` (plus légère, pas de SDK)
  - Copie `/app` depuis le stage build, lance `dotnet Worker.dll`

**Dockerfile seed** (`seed-data/Dockerfile`) :
- Image `python:3.9-slim`, installe `apache2-utils` (pour `ab` — Apache Bench)
- `make-data.py` génère les fichiers `posta` et `postb` (données URL-encodées)
- `generate-votes.sh` envoie 3000 requêtes via `ab` (2000 option A, 1000 option B)

**docker-compose.yaml** :
- `worker` sur `back-net` uniquement — il n'a pas besoin d'accès frontend
- Dépend de `redis` ET `db` (condition: `service_healthy`)
- `seed` sur `front-net`, dépend de `nginx` (envoie les votes via le LB)

**Program.cs** (`worker/Program.cs`) :
- Boucle infinie : `redis.ListLeftPopAsync("votes")` → désérialise → `INSERT INTO votes` dans PostgreSQL
- Reconnexion automatique à Redis et PostgreSQL si la connexion tombe
- Crée la table `votes` au démarrage si elle n'existe pas

### Kubernetes

**worker-deployment.yaml** (`k8s/worker-deployment.yaml`) :
- 1 replica, PAS de Service (le worker est un consommateur, personne ne l'appelle)
- `envFrom: configMapRef: app-config`
- Ressources : requests `100m CPU / 128Mi`, limits `250m / 256Mi`

**seed-job.yaml** (`k8s/seed-job.yaml`) :
- Kind `Job` (pas un Deployment) — s'exécute une fois puis s'arrête
- `restartPolicy: Never`, `backoffLimit: 2`
- Envoie 3000 requêtes `ab` directement vers `vote:5000` (pas via nginx en K8s)

**Communication** :
- worker → Redis (ClusterIP) : lit les votes avec `LPOP`
- worker → db (ClusterIP) : écrit les résultats avec `INSERT/UPDATE`
- seed → vote (LoadBalancer/ClusterIP) : envoie des POST HTTP pour simuler des votes
- Le worker est le PONT entre Redis et PostgreSQL

### Terraform Docker
- `terraform/modules/worker/main.tf` : build avec trigger, réseau `back-net` uniquement
- `terraform/modules/seed/main.tf` : build avec trigger, réseau `front-net`, `restart = "no"`

### Points clés à mentionner
- Multistage build : réduit la taille de l'image worker (SDK ~800Mo → Runtime ~200Mo)
- Worker n'a pas de Service K8s car il ne reçoit aucun trafic entrant
- Job vs Deployment : seed est éphémère, il ne doit pas redémarrer
- Le worker est le composant critique : s'il tombe, les votes restent dans Redis mais ne sont pas comptabilisés

---

## Personne 4 — Redis + DB + Infrastructure GKE

### Rôle dans l'architecture
Redis (stockage temporaire des votes) et PostgreSQL (stockage permanent des résultats) forment la couche données.
Cette personne gère aussi toute l'infrastructure GKE et Terraform GCP.

### Docker

**docker-compose.yaml** — pas de Dockerfile, images directes :
- `redis:alpine` sur `back-net`, volume bind pour les healthchecks
- `postgres:15-alpine` sur `back-net`, volume `db-data` pour la persistance
- Variables d'environnement : `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`
- Healthchecks via scripts shell (`healthchecks/redis.sh`, `healthchecks/postgres.sh`)

**Healthchecks** :
- `redis.sh` : exécute `redis-cli ping`, vérifie que la réponse est `PONG`
- `postgres.sh` : exécute `SELECT 1` via `psql`, vérifie que le résultat est `1`

### Kubernetes

**redis-deployment.yaml** (`k8s/redis-deployment.yaml`) :
- InitContainer `busybox:1.36` qui écrit le script `redis.sh` dans un volume `emptyDir`
- Container principal `redis:alpine` avec `livenessProbe exec` sur le script
- Astuce : pas besoin de créer une image custom pour Redis, l'initContainer injecte le healthcheck

**redis-service.yaml** (`k8s/redis-service.yaml`) :
- Type `ClusterIP` — Redis n'est JAMAIS exposé à l'extérieur
- Port 6379, accessible uniquement par vote et worker via le DNS interne `redis`

**db-deployment.yaml** (`k8s/db-deployment.yaml`) :
- Même pattern initContainer pour le healthcheck postgres
- `envFrom: configMapRef: app-config` pour les credentials
- Volume PVC monté sur `/var/lib/postgresql/data` avec `subPath: data`

**db-pvc.yaml** (`k8s/db-pvc.yaml`) :
- `PersistentVolumeClaim` de 1Gi, `ReadWriteOnce`
- Garantit que les données survivent au redémarrage/suppression du pod

**db-service.yaml** (`k8s/db-service.yaml`) :
- Type `ClusterIP` — PostgreSQL n'est JAMAIS exposé à l'extérieur
- Port 5432, accessible par result et worker via le DNS interne `db`

**app-configmap.yaml** (`k8s/app-configmap.yaml`) :
- ConfigMap centralisée avec toutes les variables : options de vote, credentials DB, hosts
- Utilisée par tous les services via `envFrom`

**Kustomize** (`k8s/kustomize/kustomization.yaml`) :
- Référence tous les manifests + `configMapGenerator` pour générer le ConfigMap
- Alternative au manifest manuel, génère un suffixe de hash pour forcer le redéploiement si les valeurs changent

### Terraform Docker (`terraform/modules/redis/` et `terraform/modules/postgres/`)
- Redis : image `redis:alpine`, healthcheck, réseau `back-net`, bind mount healthchecks
- Postgres : image `postgres:15-alpine`, healthcheck, réseau `back-net`, volume `db-data`

### Terraform GCP (`terraform-gcp/`)

**gke.tf** :
- Crée un cluster GKE avec 1 noeud `e2-medium` dans `europe-west3-a`
- `deletion_protection = false` pour pouvoir détruire avec `terraform destroy`
- Node pool séparé du cluster (best practice GKE)

**gke-config.tf** :
- Configure le provider Kubernetes avec le token OAuth2 du cluster GKE
- `depends_on` sur le node pool pour s'assurer que le cluster est prêt

**k8s-manifests.tf** :
- Lit tous les YAML de `k8s/` et les applique via `kubernetes_manifest`
- Injecte le namespace et le mot de passe PostgreSQL dynamiquement
- Si `enable_redis_vm = true` : exclut `redis-deployment.yaml` et transforme le service Redis en headless

**redis-vm.tf** (optionnel Part 3) :
- VM `e2-micro` Debian 12 avec script cloud-init qui installe Redis
- Configure Redis pour écouter sur `0.0.0.0` avec mot de passe
- `kubernetes_manifest.redis_endpoints` crée un Endpoint manuel pointant vers l'IP interne de la VM

**variables.tf** :
- `gcp_project_id`, `gcp_zone`, `gke_cluster_name`, `gke_node_count`
- `postgres_password` (sensitive), `redis_password` (sensitive)
- `enable_redis_vm` : toggle pour activer/désactiver la VM Redis
- `manifest_files` : liste des YAML à appliquer

### Points clés à mentionner
- ClusterIP vs LoadBalancer : Redis et DB sont TOUJOURS ClusterIP (sécurité)
- PVC : sans le `subPath: data`, PostgreSQL échoue car le répertoire n'est pas vide
- InitContainer : pattern élégant pour injecter des scripts sans créer d'images custom
- ConfigMap : centralise la config, un seul endroit à modifier
- Redis VM (Part 3) : découplage de Redis du cluster, service headless + Endpoints manuels
- `terraform destroy` supprime tout le cluster et les ressources associées

---

## Schéma de communication global

```
Utilisateur
    │
    ▼
┌─────────────────┐     ┌─────────────────┐
│  vote (LB:5000) │     │ result (LB:4000)│
│  Python/Flask    │     │  Node.js        │
└────────┬────────┘     └────────┬────────┘
         │                       │
         ▼                       ▼
┌─────────────────┐     ┌─────────────────┐
│ redis (CIP:6379)│     │  db (CIP:5432)  │
│ Stockage temp.  │     │  PostgreSQL     │
└────────┬────────┘     └────────┬────────┘
         │                       │
         └───────┐   ┌──────────┘
                 ▼   ▼
          ┌─────────────────┐
          │     worker      │
          │   .NET (pas de  │
          │    Service)     │
          └─────────────────┘
```

- `LB` = LoadBalancer (externe)
- `CIP` = ClusterIP (interne uniquement)
- worker n'a pas de Service, il consomme Redis et écrit dans DB
- seed (Job) envoie des votes vers vote pour tester
