# Pitch oral - Personne 4 (Redis + DB + Infra GKE)

## Role

Je presente la couche donnees (Redis + PostgreSQL) et toute l'infrastructure GKE/Terraform GCP.

Message central:

- Redis = stockage temporaire des votes
- PostgreSQL = stockage persistant des resultats
- Terraform = deploiement reproductible local + GCP

---

## Script oral (a lire) + Code a afficher en parallele

## 0) Ouverture (15s)

Ce que je dis:
"Je suis la personne 4. Je couvre la couche donnees Redis/PostgreSQL et l'infra GKE avec Terraform. Je vais montrer comment on passe d'un deploiement local a un deploiement cloud reproductible, puis l'extension Part 3 avec Redis sur VM."

Code a afficher:

- `docker-compose.yaml`
- `terraform-gcp/gke.tf`

---

## 1) Docker data layer (45s)

Ce que je dis:
"En local, Redis et PostgreSQL sont deploies sans Dockerfile custom: `redis:alpine` et `postgres:15-alpine`. Les deux sont sur le reseau interne. PostgreSQL monte un volume `db-data` pour persister les donnees."

"Les credentials PostgreSQL sont passes par variables d'environnement: `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`."

Code a afficher:

- `docker-compose.yaml` (services `redis` et `db`, reseaux, volume `db-data`)

Bloc a montrer:

```yaml
db:
	image: postgres:15-alpine
	environment:
		POSTGRES_USER: postgres
		POSTGRES_PASSWORD: ...
		POSTGRES_DB: postgres
	volumes:
		- db-data:/var/lib/postgresql/data
```

---

## 2) Healthchecks shell (30s)

Ce que je dis:
"Les checks sont explicites et metier: Redis doit repondre `PONG`, PostgreSQL doit repondre `SELECT 1 = 1`."

Code a afficher:

- `healthchecks/redis.sh`
- `healthchecks/postgres.sh`

Bloc a montrer:

```sh
# redis.sh
ping="$(redis-cli -h "$host" ping)"
[ "$ping" = "PONG" ]
```

```sh
# postgres.sh
select="$(echo 'SELECT 1' | psql ... )"
[ "$select" = "1" ]
```

---

## 3) Kubernetes Redis/DB (1 min)

Ce que je dis:
"Sur Kubernetes, Redis et DB restent internes: services `ClusterIP` uniquement, donc pas d'exposition internet."

"Pour les healthchecks sans image custom, on utilise un pattern propre: initContainer `busybox` qui ecrit le script dans un volume `emptyDir`, puis livenessProbe `exec` dans le conteneur principal."

"Pour PostgreSQL, le PVC garantit la persistance, et `subPath: data` evite les problemes de repertoire non vide."

Code a afficher:

- `k8s/redis-deployment.yaml`
- `k8s/redis-service.yaml`
- `k8s/db-deployment.yaml`
- `k8s/db-pvc.yaml`
- `k8s/db-service.yaml`

Blocs a montrer:

```yaml
# redis-service.yaml
spec:
	type: ClusterIP
	ports:
		- port: 6379
```

```yaml
# db-deployment.yaml
volumeMounts:
	- name: db-data
		mountPath: /var/lib/postgresql/data
		subPath: data
```

---

## 4) Config centralisee (25s)

Ce que je dis:
"La config est centralisee dans une ConfigMap unique, consommee par les services via `envFrom`. On a aussi Kustomize pour generer/mettre a jour proprement la config."

Code a afficher:

- `k8s/app-configmap.yaml`
- `k8s/kustomize/kustomization.yaml`

---

## 5) Terraform GCP (1 min)

Ce que je dis:
"Cote cloud, Terraform cree le cluster GKE et le node pool separe (best practice), avec 1 noeud conforme au sujet."

"Ensuite, Terraform lit les YAML Kubernetes et les applique via `kubernetes_manifest`. On injecte dynamiquement le namespace et les secrets necessaires."

Code a afficher:

- `terraform-gcp/gke.tf`
- `terraform-gcp/gke-config.tf`
- `terraform-gcp/k8s-manifests.tf`
- `terraform-gcp/variables.tf`

Blocs a montrer:

```hcl
resource "google_container_cluster" "primary" { ... }
resource "google_container_node_pool" "primary" { ... }
```

```hcl
resource "kubernetes_manifest" "app" {
	for_each = local.manifest_map
	manifest = each.value
}
```

---

## 6) Part 3 - Redis sur VM (1 min)

Ce que je dis:
"La Part 3 decouple Redis du cluster. Terraform cree une VM Debian, installe Redis, active mot de passe, et le service Kubernetes devient headless avec un Endpoint manuel vers l'IP interne de la VM."

"Donc l'application garde le nom logique `redis`, mais le backend Redis est externe au cluster."

Code a afficher:

- `terraform-gcp/redis-vm.tf`
- `terraform-gcp/k8s-manifests.tf` (bloc `redis_service` + `redis_endpoints`)
- `vote/app.py`
- `worker/Program.cs`

Blocs a montrer:

```hcl
resource "google_compute_instance" "redis_vm" { ... }
```

```hcl
resource "kubernetes_manifest" "redis_endpoints" {
	manifest = {
		kind = "Endpoints"
		...
	}
}
```

```python
# vote/app.py
redis_host = os.getenv("REDIS_HOST", "redis")
redis_password = os.getenv("REDIS_PASSWORD")
```

---

## 7) Demo commandes (ce que je dis pendant execution)

Commande 1:

```bash
gcloud container clusters get-credentials projet-nuage-tf --zone europe-west3-a --project lab-gcp-kube
```

Je dis:
"Je connecte kubectl au cluster cree par Terraform."

Commande 2:

```bash
kubectl get nodes
kubectl get pods,svc,pvc,hpa,jobs -o wide
```

Je dis:
"Je valide l'etat runtime de toutes les ressources."

Commande 3:

```bash
terraform -chdir=terraform-gcp output redis_vm_internal_ip
kubectl get svc redis
kubectl get endpoints redis -o wide
```

Je dis:
"Je prouve la Part 3: service headless + endpoint vers IP interne de la VM Redis."

---

## Reponses flash aux questions

Q: Pourquoi ClusterIP pour Redis/DB ?
R: "Par securite: services internes seulement, jamais exposes publiquement."

Q: Pourquoi initContainer ?
R: "Pour injecter les scripts de healthcheck sans maintenir des images custom Redis/DB."

Q: Pourquoi apply cible d'abord ?
R: "Pour creer cluster/nodepool avant d'appliquer les manifests Kubernetes, sinon provider k8s non configure."

Q: Si un pod est Pending ?
R: "Sur cluster 1-node, c'est en general une limite de capacite, pas un probleme de declaration IaC."

---

## Closing (10s)

"Notre resultat: une architecture data + infra propre, securisee, reproductible et verifiable de bout en bout avec Terraform et Kubernetes."
