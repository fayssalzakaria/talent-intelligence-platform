# Architecture

## Édition locale (Docker)

```mermaid
flowchart TB
    U[Navigateur<br/>127.0.0.1] --> APP
    subgraph Docker Compose
        APP[app<br/>Streamlit · authentification · rôles]
        WK[worker<br/>migrations · file de traitements<br/>embeddings GPU/CPU]
        DB[(db<br/>PostgreSQL 16 + pgvector)]
        APP -- lecture / écriture --> DB
        WK -- traitements --> DB
        APP -- encodage des requêtes --> WK
    end
    WK -. optionnel .-> CL[API LLM externe]
    APP -. optionnel .-> CL
    DB --- V1[(volume pgdata)]
    WK --- V2[(volume fichiers<br/>CV · dossiers)]
```

- **3 conteneurs.**
  - `db` : aucun port publié, sur un réseau interne sans accès Internet.
  - `worker` : exécute les traitements longs, comme l'analyse des CV, le matching, le clustering et l'indexation.
  - `app` : sert l'interface.
- **Une seule base.** PostgreSQL stocke à la fois les données relationnelles et les vecteurs (pgvector). Les
  volumes d'une ESN ne justifient ni Spark, ni une base vectorielle séparée, ni un broker de messages.
- **Une file de traitements dans PostgreSQL.** `SELECT … FOR UPDATE SKIP LOCKED` : pas de dépendance
  supplémentaire, et les reprises sont garanties.
- **Des embeddings locaux.** Le modèle est choisi selon le matériel :
  - Qwen3-Embedding-0.6B (1024 dimensions) sur GPU ;
  - e5-small sur CPU.

  Changer de modèle déclenche le recalcul des vecteurs concernés, et lui seul.
- **Deux modes de confidentialité.**
  - **Local** : aucun appel externe. L'extraction se fait par règles et l'assistant est désactivé.
  - **LLM assisté** : fonctionnalités activables une à une. Les coordonnées sont masquées avant envoi.

## Édition Cloud (Databricks / Azure)

```mermaid
flowchart LR
    VOL[Unity Catalog Volume<br/>CV et missions] --> B[bronze<br/>registre et texte brut]
    B --> S[silver<br/>profils extraits<br/>GPT-4.1-mini]
    S --> G[gold<br/>embeddings · scores<br/>clusters · feedback]
    G --> APP[Databricks Apps<br/>Streamlit]
    S --> VS[Foundry<br/>Vector Store]
    VS --> AG[Talent Agent]
    G --> AG
    AG --> APP
    APP -- actions --> JOB[Databricks Job]
    JOB --> G
```

- Architecture **médaillon** en tables Delta : `bronze` pour le brut, `silver` pour le structuré, `gold` pour
  les résultats.
- Les traitements tournent en **notebooks et Jobs Databricks**, de façon incrémentale, sur les nouvelles versions
  de CV seulement.
- Extraction et agent sur **Azure AI Foundry** (GPT-4.1-mini). Embeddings : `text-embedding-3-large`.
- L'application, sur **Databricks Apps**, ne lance pas Spark. Ses actions (disponibilité d'un consultant,
  recalculs) passent par un Job.

## Pipeline commun

```mermaid
flowchart LR
    A[Import] --> B[Déduplication<br/>SHA-256 + versions]
    B --> C[Extraction<br/>profil structuré]
    C --> D[Embeddings]
    D --> E[Matching<br/>incrémental]
    D --> F[Clustering]
    C --> G[Index de recherche]
```

Chaque étape est **idempotente**. Une signature (version du CV, profil de mission, modèle d'embeddings, version
du score) permet de ne recalculer que les paires candidat–mission qui ont changé.

## Agent IA

```mermaid
sequenceDiagram
    participant U as Recruteur
    participant A as Agent (LLM)
    participant T as Outils
    participant V as Validation
    U->>A: Question
    loop jusqu'à la réponse
        A->>T: appel d'outil (recherche, profil, scores, comparaison…)
        T-->>A: résultat + identifiant de preuve (E1, E2…)
    end
    A->>V: réponse + preuves citées
    V-->>V: preuve inconnue ? classement sans score officiel ?<br/>dénombrement sans recherche ? absence non vérifiée ?
    V-->>U: réponse sourcée, ou refus motivé
```

**Outils disponibles :**
- recherche de consultants et de missions ;
- profil détaillé ;
- meilleurs candidats d'une mission ;
- détail d'un score ;
- comparaison de 2 à 4 consultants ;
- couverture de compétences ;
- groupe (cluster) d'un consultant ;
- recherche dans les documents.

**Télémétrie de chaque réponse :** latence, nombre d'appels, jetons, coût estimé, preuves citées, contrôles
qualité et versions des données lues.
