# Choix techniques

## Pourquoi un score explicable et non un modèle appris ?

Un recruteur doit pouvoir justifier une proposition à un client. Le score se décompose en quatre critères
lisibles :
- compétences obligatoires couvertes ;
- similarité sémantique du profil ;
- expérience ;
- certifications.

Quand un critère est inconnu, il est exclu et les poids sont renormalisés : il ne compte pas pour zéro.

Un modèle d'apprentissage (régression logistique) est entraîné sur les avis des recruteurs, mais seulement
comme **contrôle**. Mesuré hors échantillon, il ne fait pas mieux que le score métier (AUC 0,859 contre 0,895) :
il n'est donc pas utilisé pour classer. Voir [evaluation.md](evaluation.md).

## Pourquoi un score hybride ?

- La **comparaison exacte des compétences** est précise, mais elle rate les synonymes et les compétences proches.
- La **similarité sémantique** (embeddings) capte la proximité des profils, mais elle est moins explicable.

Le score combine les deux. Des équivalences explicites (« REST API » = « API REST », un outil comme pytest
couvre « tests automatisés ») corrigent les écarts de vocabulaire, sans rien déduire automatiquement.

## Comment le LLM est encadré

| Usage | Garde-fou |
|---|---|
| Extraction des CV | Sortie structurée (schéma JSON), retour automatique aux règles locales en cas d'échec, cache par empreinte du texte |
| Assistant | Le LLM n'a accès qu'à des outils en lecture. Chaque affirmation doit citer une preuve, sinon la réponse est rejetée. |
| Dossiers de compétences | Toute compétence, tout terme ou tout chiffre absent du CV fait **rejeter en bloc** la version du LLM. La version locale fidèle est alors utilisée. |
| Coûts | Budget mensuel, plafond de jetons par opération, coût enregistré pour chaque appel |

## Confidentialité

- **Mode local** : aucun appel externe. Extraction par règles et embeddings open source sur la machine.
- **Mode assisté** : chaque fonctionnalité LLM s'active séparément. Noms et coordonnées sont masqués avant envoi.
- **Dossiers anonymisés** : initiales et employeurs remplacés par leur secteur. La génération est bloquée si un
  courriel, un téléphone ou un nom subsiste.
- **Sécurité** :
  - mots de passe hachés (scrypt) ;
  - rôles (lecteur, manager, administrateur) ;
  - journal d'audit ;
  - sauvegardes chiffrées (AES-256-GCM) ;
  - suppression complète d'un consultant et de tous ses dérivés (droit à l'effacement).

## Pourquoi deux éditions ?

- **Cloud** : pour une organisation qui dispose déjà de Databricks et d'Azure. Elle profite de leur passage à
  l'échelle, de leur gouvernance (Unity Catalog) et de leurs modèles managés.
- **Locale** : pour une petite structure ou des données sensibles. Elle s'installe en une commande, tourne sans
  GPU et peut fonctionner sans aucune API payante.

Le passage du Cloud au local a demandé :
- de remplacer Delta et Spark par PostgreSQL ;
- de remplacer le Vector Store managé par pgvector ;
- de remplacer les embeddings Azure par des modèles open source comparés sur le jeu de test.

Les algorithmes de matching et de clustering sont restés identiques.
