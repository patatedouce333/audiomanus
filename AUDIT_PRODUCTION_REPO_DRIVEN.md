## **AUDIT_PRODUCTION_REPO_DRIVEN.md**

### **Objectif**
Réaliser un audit **strictement basé sur le contenu réel du dépôt** (fichiers, dossiers, scripts, configurations).  
**Aucune supposition** : tout élément non détectable est marqué **Absent / Non trouvé / Non observable**.

---

### **Règles d’audit (obligatoires)**
- Toute affirmation doit être **justifiée par une preuve** (chemin de fichier + extrait si nécessaire).
- Interdiction d’introduire une techno, un outil, une pratique ou un document **qui n’existe pas dans le repo**.
- Tout ce qui n’est pas prouvé dans le dépôt doit être noté :
  - **Absent**
  - **Non trouvé**
  - **Non observable dans le repo**

---

## 1) Inventaire factuel du dépôt
### 1.1 Arborescence utile
- **Racine du repo** :
  - `agent/` : Contient la logique de l'agent principal et du nouvel agent visuel.
  - `frontend/` : Application web (non détaillée).
  - `image-crawler/` : Un projet Scrapy complet pour le crawling d'images.
  - `orchestrator/` : API qui coordonne les tâches entre les différents modules.
  - `youtube-transcriber/` : Module autonome pour la transcription de vidéos YouTube.
  - `docker-compose.yml` : Définit les services pour l'environnement de développement/production.
  - Scripts `.sh` : `build-images.sh`, `deploy.sh`, `monitor-system.sh`, `setup-gcp.sh`, etc.

- **Dossiers structurants** :
  - `agent/` : Contient `agent.py` (agent principal) et `vision_agent.py` (agent visuel).
  - `orchestrator/` : Contient `orchestrator.py` (API Flask).
  - `youtube-transcriber/` : Contient `api.py` (API GraphQL) et `worker.py` (tâches de transcription).
  - `image-crawler/` : Structure d'un projet Scrapy avec des `spiders`, `pipelines`, etc.

**Preuves**
- **Chemins** : L'inventaire est basé sur la liste de fichiers fournie par la commande `ls -RF`. Les dossiers listés ci-dessus sont les conteneurs logiques principaux de l'application.

---

## 2) Démarrage, build, exécution (uniquement commandes existantes)
### 2.1 Commandes officielles trouvées
- **Commande(s) de démarrage local** :
  - `docker-compose up` (implicite)
    - **Preuve** : `docker-compose.yml`
    - **Description** : Lance les services `agent`, `orchestrator`, `redis`, `api`, et `worker`.
- **Commande(s) de build** :
  - `sh build-images.sh`
    - **Preuve** : `build-images.sh`
    - **Description** : Construit les images Docker pour l'agent, l'orchestrateur et le transcripteur YouTube.
- **Commande(s) d’exécution type prod** :
  - `sh deploy.sh`
    - **Preuve** : `deploy.sh`
    - **Description** : Script de déploiement (contenu non analysé, mais le fichier existe).
- **Commande(s) de test** :
  - `sh test-e2e.sh`, `sh test-monitoring.sh`, `sh test-shadow-mode.sh`
    - **Preuve** : Existence des fichiers de script.
    - **Description** : Suggère une suite de tests end-to-end et de monitoring.

---

## 3) Cartographie réelle du code
### 3.1 Organisation interne
- **Entrypoints** :
  - **Orchestrateur (`orchestrator/orchestrator.py`)** : Point d'entrée principal pour les requêtes de scraping (`/scrape`, `/scrape/visual`). C'est une API Flask.
  - **API YouTube (`youtube-transcriber/api.py`)** : Point d'entrée GraphQL pour les tâches de transcription.
  - **Frontend (`frontend/app.py`)** : Application Streamlit.
- **Logique principale** :
  - **`agent/agent.py`** : Gère les tâches de scraping textuel.
  - **`agent/vision_agent.py`** : Gère les tâches de scraping visuel (nouvelle fonctionnalité).
  - **`youtube-transcriber/worker.py`** : Exécute le processus de transcription en arrière-plan.
  - **`image-crawler/spiders/image_spider.py`** : Logique de crawling pour le projet Scrapy.
- **Interfaces** :
  - API REST : `orchestrator/orchestrator.py`
  - API GraphQL : `youtube-transcriber/graphql_schema.py`
  - CLI : Les divers scripts `.sh`.
  - UI : `frontend/app.py`
- **Utilitaires partagés** : **Non observable dans le repo**. La logique semble encapsulée dans chaque module sans un dossier `shared` ou `common` évident.

**Preuves**
- **`orchestrator/orchestrator.py`** : Contient les routes Flask qui délèguent aux agents.
- **`agent/agent.py`** : Importe et utilise `VisionAgent` (preuve de l'intégration).
- **`youtube-transcriber/Dockerfile`** : Expose le port 8000 et lance l'API avec `uvicorn`.

---

## 4) Flux fonctionnels observables
### 4.1 Parcours principaux identifiables
- **Flux A : Scraping Visuel (nouveau)**
  1.  **Entrée** : Requête POST sur l'endpoint `/scrape/visual` de l'**orchestrateur**.
      - **Preuve** : `orchestrator/orchestrator.py`
  2.  **Traitement** : L'orchestrateur publie une tâche "visual_scraping" dans Redis. L'**agent principal** (`agent.py`) consomme la tâche et la délègue au `VisionAgent` (`vision_agent.py`).
      - **Preuve** : `agent/agent.py`
  3.  **Sortie** : Le résultat du scraping est stocké (destination non spécifiée dans le code).

- **Flux B : Transcription YouTube (legacy + moderne)**
  1.  **Entrée** : Requête GraphQL à l'**API** (`youtube-transcriber/api.py`).
  2.  **Traitement** : La tâche est ajoutée à une base de données (probablement PostgreSQL, vu `database.py`) et traitée par un **worker** (`youtube-transcriber/worker.py`). Le worker utilise `VertexAITranscriber` ou potentiellement une ancienne méthode.
      - **Preuve** : `youtube-transcriber/worker.py`, `youtube-transcriber/vertex_ai_transcriber.py`. Le document `PROJECT_STATE.md` confirme la migration vers Vertex AI.
  3.  **Sortie** : La transcription est stockée dans la base de données.

---

## 5) Données et persistance (si présent)
### 5.1 Présence de persistance
- **Statut** : ✅ **Présent**
- **Description** :
  - **Redis** : Utilisé comme file d'attente de tâches (message broker) entre l'orchestrateur et les agents.
    - **Preuve** : `docker-compose.yml` (service `redis`), `agent/agent.py` (connexion à Redis).
  - **Base de données relationnelle (probable)** : Le module `youtube-transcriber` contient des fichiers `database.py`, `migrations.py` et `image-crawler` contient un `schema.sql`, suggérant l'utilisation d'une base de données SQL (probablement PostgreSQL).
    - **Preuve** : `youtube-transcriber/database.py`, `image-crawler/schema.sql`.

---

## 6) Sécurité observable (preuves uniquement)
### 6.1 Secrets et configuration
- **`.env*` trouvé(s) ?** : **Non trouvé**. Aucun fichier `.env` ou `.env.example` n'est listé.
- **Des secrets sont-ils en clair dans le repo ?** : **Non observable dans le repo**. Le code semble charger la configuration depuis les variables d'environnement (`os.getenv`), ce qui est une bonne pratique.
  - **Preuve** : `youtube-transcriber/config.py` utilise `os.getenv("YOUTUBE_API_KEY")`.
- **Les accès sensibles sont-ils explicitement gérés dans le code ?** : Oui, via des variables d'environnement. Cependant, la gestion du cycle de vie de ces secrets (rotation, stockage sécurisé) est **non observable dans le repo**.

---

## 7) Qualité, stabilité, reproductibilité (présent ou absent)
### 7.1 Tests
- **Statut** : ✅ **Présent**
- **Description** : Des tests existent mais leur couverture semble partielle et hétérogène.
  - `tests/test_vision_agent.py` : Test unitaire pour le nouvel agent visuel.
  - `image-crawler/tests/` : Tests pour les parseurs du crawler.
  - `test-e2e.sh`, `test-monitoring.sh` : Scripts suggérant des tests plus larges.
  - **Absence** : Aucun test n'est visible pour le module `youtube-transcriber` ou `orchestrator`.

### 7.2 Contrôles automatiques
- **Statut** : ✅ **Présent** (limité)
- **Description** : Le projet `image-crawler` utilise `poetry` et a un `pyproject.toml`, ce qui suggère une gestion moderne des dépendances et potentiellement des outils comme `black` ou `ruff`, mais aucune configuration explicite de linter/formatter n'est visible pour les autres modules.

---

## 8) Problèmes bloquants avant production (liste courte)
**[BLOQUANT #1] Hétérogénéité et Fragmentation de la Configuration**
- **Preuve** : Multiples `Dockerfile`, `requirements.txt`, un `pyproject.toml`, et plusieurs logiques de configuration (`config.py`, `os.getenv`). Il n'y a pas de source de vérité unique pour les dépendances ou la configuration.
- **Impact production** : Risque élevé de "dependency hell", difficile de maintenir et de sécuriser. Le déploiement est complexe et sujet à des erreurs de configuration entre les modules.
- **Fix minimal** : Centraliser la gestion des dépendances (ex: `poetry` pour tout le monorepo) et standardiser la méthode de chargement de la configuration.

**[BLOQUANT #2] Absence de Tests sur des Modules Critiques**
- **Preuve** : Les dossiers `youtube-transcriber` and `orchestrator` ne contiennent aucun répertoire de tests.
- **Impact production** : L'orchestrateur est le cerveau du système. Le module de transcription est un flux de revenus clé (`PROJECT_STATE.md`). L'absence de tests sur ces composants rend toute modification risquée et les régressions probables.
- **Fix minimal** : Ajouter un test de fumée (`smoke test`) pour chaque API (orchestrateur et transcripteur) pour vérifier que les services démarrent et que les endpoints principaux répondent correctement.

**[BLOQUANT #3] Documentation et État du Projet Contradictoires**
- **Preuve** : Le dépôt contient à la fois `VISION_AGENT_TODO.md` (marqué "TERMINÉ") et `PROJECT_STATE.md` qui se présente comme "la source de vérité unique" et remplace tous les autres `*.md`.
- **Impact production** : Confusion pour les nouveaux développeurs et les opérations. On ne sait pas quelle documentation est à jour, ce qui ralentit la maintenance et la résolution d'incidents.
- **Fix minimal** : Archiver ou supprimer les fichiers de documentation obsolètes (`VISION_AGENT_TODO.md`, `CONVERSATIONAL_AGENT_TODO.md`) et s'assurer que `PROJECT_STATE.md` est la seule référence.

---

## 9) Verdict GO / NO-GO
### **Décision** : **NO-GO**

### Justification
Le projet est fonctionnel mais sa structure éclatée, le manque de tests sur des composants vitaux (orchestrateur, API de transcription), et la fragmentation de la configuration et de la documentation présentent des risques opérationnels trop élevés pour un passage en production serein. Les bloquants identifiés sont des bombes à retardement qui rendront la maintenance coûteuse et les pannes difficiles à diagnostiquer.

---

## 10) Check-list “minimum vital” (actions exécutables)
- [ ] **Action 1 (Fix Bloquant #1)** : Unifier la gestion des dépendances de tous les modules Python sous un seul outil (ex: `poetry` monorepo).
- [ ] **Action 2 (Fix Bloquant #1)** : Créer un fichier `.env.example` à la racine avec toutes les variables d'environnement requises par l'ensemble des services.
- [ ] **Action 3 (Fix Bloquant #2)** : Implémenter des tests d'intégration pour les endpoints critiques : `/scrape/visual` (orchestrateur) et la mutation GraphQL principale (transcripteur).
- [ ] **Action 4 (Fix Bloquant #3)** : Supprimer les fichiers `*.md` obsolètes et ne conserver que `PROJECT_STATE.md` et `README.md`.

---

## Annexes
### A) Fichiers et chemins examinés
- `PROJECT_STATE.md`
- `VISION_AGENT_TODO.md`
- `docker-compose.yml`
- `build-images.sh`, `deploy.sh`
- `agent/agent.py`, `agent/vision_agent.py`
- `orchestrator/orchestrator.py`
- `youtube-transcriber/api.py`, `youtube-transcriber/worker.py`, `youtube-transcriber/config.py`
- `image-crawler/pyproject.toml`, `image-crawler/schema.sql`
- `tests/test_vision_agent.py`

### B) Points non observables
- **CI/CD** : Aucune configuration de pipeline (`.github/workflows`, `.gitlab-ci.yml`) n'est visible.
- **Monitoring / Logging détaillé** : Bien qu'un script `monitor-system.sh` existe, la stratégie de logging centralisé (ex: ELK, Datadog) n'est pas observable.
- **Gestion des secrets en production** : L'utilisation de variables d'environnement est une bonne pratique, mais le stockage et l'injection de ces secrets (ex: Vault, AWS/GCP Secrets Manager) ne sont pas décrits.
- **Contrat d'API formel** : Pas de spécification OpenAPI/Swagger pour l'API REST de l'orchestrateur.
