# Évaluation

Toutes les mesures portent sur un **jeu fictif de 30 CV et 9 missions**. Ce sont des ordres de grandeur sur un
petit échantillon, pas des résultats de production.

## 1. Le score de matching face au jugement humain

**Protocole :**
- Un recruteur donne un avis 👍 ou 👎 sur des couples candidat–mission, notamment sur les cas limites
  (rangs 3 à 10), là où le score hésite.
- On mesure l'**AUC** : la probabilité qu'un candidat jugé pertinent soit mieux classé qu'un candidat jugé non
  pertinent.

| Méthode | AUC |
|---|---|
| **Score métier** (40 % compétences, 35 % sémantique, 15 % expérience, 10 % certifications) | **0,895** |
| Régression logistique sur les 4 composantes (validation croisée à 5 plis) | 0,859 |

**Lecture :**
- Le score explicable range correctement environ **9 paires sur 10**.
- Le modèle appris ne fait pas mieux. L'écart (−0,036) tient au hasard avec 31 avis.
- Décision : **le score transparent reste le classement officiel**. Le modèle sert de contrôle.

**Ce que révèlent les coefficients du modèle :**
- **Compétences : 1,59.** C'est ce qui pèse le plus dans le jugement du recruteur.
- **Sémantique : 0,91.**
- **Expérience : ≈ 0.** Presque tous les candidats dépassent le minimum demandé, donc ce critère ne les
  départage pas.
- **Certifications : 0.** Aucune mission n'en exigeait.

## 2. Le choix du modèle d'embeddings

Trois modèles open source ont été comparés pour l'édition locale, sur les 30 CV et 9 missions fictifs. Mesures du
15/10/2026, sur un PC avec une NVIDIA RTX 4070 (12 Go) et un processeur à 24 threads.

**Protocole :**
- un CV est jugé pertinent pour la mission de sa famille de métier ;
- chaque modèle classe les CV pour chaque mission ;
- on mesure la qualité du classement, la finesse du modèle sur des paires de compétences et la vitesse.

| | Qwen3-Embedding-0.6B | BGE-M3 | multilingual-e5-small |
|---|---|---|---|
| Classement CV ↔ mission (nDCG@5 / MRR) | 0,875 / 1,00 | 0,859 / 1,00 | **0,906** / 1,00 |
| Compétences proches ou éloignées bien distinguées (AUC) | **0,95** | 0,78 | 0,77 |
| Requêtes anglaises sur CV français (nDCG@5) | 0,80 | 0,81 | **0,90** |
| Étendue des similarités mission ↔ CV | **0,37 – 0,87** | 0,48 – 0,88 | 0,83 – 0,94 |
| Vitesse sur GPU | 100 CV/s | **245 CV/s** | 143 CV/s |
| Vitesse sur CPU | 0,2 CV/s | 2,3 CV/s | **21 CV/s** |
| Mémoire GPU (fp16) | 2,1 Go | 1,2 Go | **0,3 Go** |
| Stabilité du clustering (ARI)* | **0,84** | 0,56 | 0,70 |
| Dimension · licence | 1024 · Apache-2.0 | 1024 · MIT | 384 · MIT |

\* Mesurée dans les conditions du banc d'essai, avec des profils extraits par règles locales. La valeur de 0,66 de
la section 3 vient de l'application réelle.

**Décision : le modèle dépend du matériel, choisi automatiquement au démarrage.**
- **Avec un GPU : Qwen3-Embedding-0.6B.** C'est le seul qui distingue nettement les compétences proches des
  compétences éloignées. Ses similarités s'étalent de 0,37 à 0,87, donc la partie sémantique du score (35 %)
  départage réellement les profils.
- **Sans GPU : multilingual-e5-small.** Qwen3 tombe à 0,2 CV/s sur processeur, soit plus d'une heure pour 1 000 CV.
  e5-small en traite 21 par seconde et classe aussi bien. Sa limite : ses similarités sont resserrées (0,83 à 0,94),
  donc la partie sémantique départage moins.
- **BGE-M3** reste disponible en option. Il est rapide sur GPU, mais il ne fait mieux que Qwen3 sur aucun critère
  de qualité.

Avec 9 missions, quelques points d'écart ne sont pas significatifs. Le banc d'essai sert à écarter un modèle
inadapté, pas à mesurer une précision réelle.

## 3. Le clustering des profils

**Méthode de sélection automatique.**

Le système compare, de façon non supervisée :
- **3 représentations des profils** : embeddings, TF-IDF des compétences, et une combinaison des deux ;
- **3 algorithmes** : k-means, k-means sphérique et classification hiérarchique cosinus ;
- **un nombre de groupes K de 2 à 9.**

Chaque partition est notée par la silhouette, combinée dans l'espace sémantique et dans l'espace des
compétences, et par sa **stabilité** : l'ARI moyen sur 5 sous-échantillons de 80 %.

Une partition est refusée si elle est instable, si sa structure est trop faible ou si ses groupes sont trop
petits. Entre deux partitions dont le score est à 5 % près, on garde la plus stable, puis la plus simple.

| Édition | Stabilité (ARI) |
|---|---|
| Locale (Qwen3-Embedding) | **0,66** |
| Cloud (text-embedding-3-large) | 0,61 |

Les groupes sont nommés automatiquement par leurs **termes distinctifs** (c-TF-IDF), par exemple
« React · TypeScript · Jest » ou « PySpark · Data Engineer · Azure Databricks ».

La carte 2D propose deux projections :
- **l'ACP**, fidèle aux distances ;
- **la LDA**, qui sépare visuellement les groupes.

Ces projections servent uniquement à l'**affichage** : elles n'influencent pas le clustering.

## 4. L'assistant IA

| Mesure | Valeur typique |
|---|---|
| Latence | 5 à 10 s |
| Coût par question | ≈ 0,01 à 0,02 $ |
| Preuves | chaque affirmation cite une preuve issue d'un outil (E1, E2…) |

**Garde-fous vérifiés par des tests automatisés :**
- preuve citée inexistante ;
- classement affirmé sans score officiel ;
- dénombrement affirmé sans recherche ;
- absence affirmée sans vérification ;
- question hors des données.

## 5. Qualité logicielle

**92 tests automatisés** sur l'édition locale :
- unitaires ;
- intégration sur une vraie base PostgreSQL ;
- tests de l'interface Streamlit.

Le LLM et les embeddings sont simulés dans les tests : ils sont reproductibles et ne coûtent rien.

## Limites connues

- **Peu de données.** 30 CV et 31 avis : les écarts de quelques points ne sont pas significatifs.
- **Équivalences de compétences écrites à la main**, par exemple « REST API » = « API REST », et « pytest »
  couvre « tests automatisés ». Une catégorie absente de la liste n'est captée que par la similarité sémantique.
- **Les avis reflètent un seul évaluateur.**
