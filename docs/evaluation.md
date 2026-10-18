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

## 2. Le clustering des profils

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

## 3. L'assistant IA

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

## 4. Qualité logicielle

**91 tests automatisés** sur l'édition locale :
- unitaires ;
- intégration sur une vraie base PostgreSQL ;
- tests de l'interface Streamlit.

Le LLM et les embeddings sont simulés dans les tests : ils sont reproductibles et ne coûtent rien.

## Limites connues

- **Peu de données.** 30 CV et 31 avis : les écarts de quelques points ne sont pas significatifs.
- **Équivalences de compétences écrites à la main**, par exemple « REST API » = « API REST », et « pytest »
  couvre « tests automatisés ». Une catégorie absente de la liste n'est captée que par la similarité sémantique.
- **Les avis reflètent un seul évaluateur.**
