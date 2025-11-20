# Projet Final - Architecture Dockerisée

## Membres du groupe

- Amaury Mechin
- Kellian Kauffmann

## Architecture globale

- **reverse-proxy (Nginx)** : Point d'entrée unique, fait office de reverse proxy pour router les requêtes vers le frontend ou l'API backend.
- **webapp (Frontend)** : Application web (Vite/React) servie par Nginx.
- **spring-api (Backend)** : API REST Spring Boot connectée à PostgreSQL.
- **db (PostgreSQL)** : Base de données relationnelle.

Tous les services communiquent via un réseau Docker interne. Seul le reverse proxy expose le port 80 à l'extérieur.

```
[Client]
   |
[Reverse Proxy :80]
   |--------> /api/* --------> [Spring Boot API :8080]
   |--------> / (autres) ----> [Webapp :80]
   |
[PostgreSQL :5432] <--- [Spring Boot API]
```

## Commandes pour builder et lancer

1. Cloner le dépôt et se placer dans le dossier :
   ```sh
   git clone <repo-url>
   cd dockerProjetFinal
   ```
2. Adapter le fichier `.env` si besoin (ports, credentials...)
3. Builder et lancer tous les services :
   ```sh
   docker compose up -d
   ```
4. Pour arrêter :
   ```sh
   docker compose down
   ```

## Endpoints API & URL frontend

- **Frontend** :
  - http://localhost/
- **API** :
  - GET    http://localhost/api/health         → Vérifier l'état de l'API
  - GET    http://localhost/api/items          → Liste des items
  - POST   http://localhost/api/items          → Ajouter un item (JSON)

## Choix techniques & raisons

- **Reverse proxy Nginx** :
  - Permet d'avoir un point d'entrée unique, de gérer le routage, la sécurité, les headers CORS et de masquer les ports internes.
- **Spring Boot** :
  - Framework robuste pour l'API, facile à dockeriser, supporte PostgreSQL nativement.
- **PostgreSQL** :
  - Base de données open source fiable, adaptée aux besoins relationnels.
- **Webapp (Vite/React ou Angular)** :
  - Frontend moderne, build rapide, facile à servir via Nginx.
- **Docker Compose** :
  - Orchestration simple de tous les services, reproductibilité de l'environnement, isolation des composants.
- **.env** :
  - Centralisation de la configuration pour faciliter le déploiement sur différents environnements.

---

## Choix de construction des Dockerfile

### reverse-proxy (Nginx)
- **Base :** `nginx:alpine` (image légère et performante)
- **Pourquoi ?**
  - Alpine réduit la taille de l'image.
  - Nginx est idéal pour le reverse proxy, la gestion des headers et le routage.
- **Spécificités :**
  - Copie d'un `nginx.conf` personnalisé pour router / et /api/.

### webapp (Frontend)
- **Multi-stage build :**
  - **Étape 1 :** `node:20-alpine` pour builder l'app (npm ci + npm run build)
  - **Étape 2 :** `nginx:alpine` pour servir les fichiers statiques
- **Pourquoi ?**
  - Sépare le build JS (besoin de Node) et la livraison (Nginx, rapide et sécurisé).
  - L'image finale ne contient que les fichiers nécessaires à la prod (plus légère, plus sûre).
  - Alpine pour la légèreté.
- **Spécificités :**
  - Copie du build dans le dossier Nginx, configuration custom si besoin.

### spring-api (Backend)
- **Multi-stage build :**
  - **Étape 1 :** `maven:3.9-eclipse-temurin-21` pour builder le JAR
  - **Étape 2 :** `eclipse-temurin:21-jre-alpine` pour exécuter le JAR
- **Pourquoi ?**
  - On évite d'avoir Maven et les sources dans l'image finale (plus légère, plus rapide à démarrer, plus sécurisée).
  - Alpine pour la légèreté.
  - Utilisation de `-DskipTests` pour accélérer le build en CI/CD.
  - Utilisation de `mvn dependency:go-offline` pour optimiser le cache Docker.
- **Spécificités :**
  - Seul le JAR final est copié dans l'image de prod.

### db (PostgreSQL)
- **Base :** `postgres:18-alpine`
- **Pourquoi ?**
  - Image officielle, fiable, légère.
  - Configuration via variables d'environnement et persistance via volume Docker.

---

**Remarque** :
- Seul le reverse proxy est exposé, les autres services sont isolés pour plus de sécurité.
- Les logs du reverse proxy sont visibles via `docker logs reverse-proxy`.


