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

2. Mode développement avec docker-compose.override.yml

Un fichier `docker-compose.override.yml` est fourni et déjà valorisé/prêt à l'emploi pour faciliter le développement local. Il surcharge la configuration principale avec des options adaptées au dev :

- **Ports différents** : les services sont exposés sur d'autres ports pour éviter les conflits avec d'autres applications locales.
- **Variables d'environnement** : activation des profils ou modes "dev" dans les applications (ex : `SPRING_PROFILES_ACTIVE=dev`).

**Exemple d'utilisation** : ce fichier est déjà configuré pour un usage immédiat, il est automatiquement pris en compte avec la commande classique :
```sh
docker compose up -d
```
Pour ignorer ce fichier et n'utiliser que la config principale (production) :
```sh
docker compose -f docker-compose.yml up -d
```

**Remarque** : Ce fichier n'est pas destiné à la production et peut être personnalisé par chaque développeur. Il est conseillé de l'ajouter au `.gitignore` pour garder une config locale propre à chacun.

3. Adapter le fichier `.env` si besoin (ports, credentials...)
> ⚠️ **Attention :** Les valeurs ci-dessous sont fournies uniquement à titre d'exemple pour faciliter la compréhension. Ne jamais utiliser ces identifiants ou mots de passe en production ou dans un dépôt public.

```env
POSTGRES_PORT=5432
POSTGRES_DB=exampledb
POSTGRES_USER=exampleuser
POSTGRES_PASSWORD=examplepassword

SPRING_DATASOURCE_URL= jdbc:postgresql://db:${POSTGRES_PORT}/${POSTGRES_DB}
SPRING_DATASOURCE_USERNAME=${POSTGRES_USER}
SPRING_DATASOURCE_PASSWORD=${POSTGRES_PASSWORD}

REVERSE_PROXY_PORT=80
VITE_API_BASE_URL=http://localhost
```

> Remplacez chaque valeur par vos propres informations pour garantir la sécurité de votre application.

4. Builder et lancer tous les services :

   **En mode dev**
   ```sh
   docker compose up -d
   ```
   
   **En mode production**
   ```sh
   docker compose -f docker-compose.yml up -d
   ```
5. Pour arrêter :
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

## Healthchecks (Vérification d'état automatique)

Des healthchecks sont mis en place pour garantir que les services critiques sont bien démarrés et fonctionnels avant que les autres services qui en dépendent ne démarrent à leur tour.

- **db (PostgreSQL)** :
  - Utilise la commande `pg_isready` pour vérifier que la base de données est prête à accepter des connexions.
  - Si la base n'est pas prête, le service `spring-api` attend avant de démarrer.

- **spring-api (API Spring Boot)** :
  - Un healthcheck interroge l'endpoint HTTP `/api/health` sur le port 8080 pour s'assurer que l'API est bien démarrée et répond.
  - Si l'API n'est pas prête, le service `reverse-proxy` attend avant de démarrer.

- **reverse-proxy (Nginx)** :
  - Démarre uniquement lorsque l'API (`spring-api`) est considérée comme healthy (`condition: service_healthy`) **et** que le frontend (`webapp`) est démarré (`condition: service_started`).
  - Cela garantit que les requêtes HTTP reçues par le reverse proxy ne sont pas envoyées vers un backend ou un frontend indisponible.

**Résumé du fonctionnement** :

1. PostgreSQL démarre et devient healthy.
2. L'API Spring Boot attend que PostgreSQL soit healthy, puis démarre et expose `/api/health`.
3. Le reverse-proxy attend que l'API soit healthy et que le frontend soit démarré avant de router les requêtes.

Cela améliore la robustesse de l'architecture et évite les erreurs de connexion au démarrage.

---

**Remarque** :
- Seul le reverse proxy est exposé, les autres services sont isolés pour plus de sécurité.
- Les logs du reverse proxy sont visibles via `docker logs reverse-proxy`.


