# PokéPrice FR

## Objectif

PokéPrice FR est une plateforme permettant d’estimer le prix de vente d’une carte Pokémon française à partir de ventes réellement conclues.

Le système repose sur un principe fondamental :

Un prix fiable doit être basé sur des transactions effectives, et non sur des annonces actives.

---

# Périmètre du projet

## MVP

- Cartes Pokémon FR uniquement
- Séries des 4 à 5 dernières années
- Une seule entrée par numéro de carte
- Segmentation indépendante des prix :
  - RAW
  - PSA (grade par grade)
  - PCA (grade par grade)
- Fenêtre principale de calcul : 30 jours
- Affichage des X dernières ventes
- Courbe d’évolution
- Score de confiance (0 à 100)

## Évolutions prévues

- Extension à toutes les séries FR
- Gestion des variantes (holo, reverse)
- Ajout d’autres sociétés de gradation
- Comparaison prix vendus vs annonces actives
- Comptes utilisateurs et gestion de collection
- Alertes de prix
- API publique
- Statistiques globales du marché (par Pokémon, rareté, set)

---

# Sources de données

## Catalogue

### TCGdex API

Utilisée pour importer :
- Séries françaises
- Cartes
- Numéros
- Raretés
- Images

Elle alimente les tables `sets` et `cards` et permet la mise à jour automatique lors des nouvelles sorties.

---

## Prix (ventes réalisées)

### eBay (ventes complétées)

Extraction :
- Prix
- Date
- Titre
- Lien
- Devise
- Frais de port

Les données sont ensuite normalisées avant traitement.

### Cardmarket (référence secondaire)

Le Price Guide peut servir d’indicateur complémentaire.  
Les ventes unitaires restent la source principale du calcul.

---

# Architecture

Le projet est structuré en quatre services Docker :

- frontend : interface utilisateur
- api : lecture des données
- worker : traitement et calcul des prix
- db : PostgreSQL

Principe central :

L’API ne calcule pas les prix en temps réel.  
Tous les calculs sont effectués en amont par le Worker.

---

# Pipeline de traitement (Worker)

## 1. Import du catalogue

Synchronisation avec TCGdex pour importer les sets et les cartes.

## 2. Collecte des ventes

Récupération planifiée des ventes complétées.

## 3. Normalisation

- Conversion des devises en EUR
- Calcul du prix total :
  price_total = price_item + price_shipping
- Extraction des informations de grade (PSA, PCA, etc.)
- Détection des cartes non gradées (RAW)
- Nettoyage des titres
- Exclusion des lots

Chaque vente est assignée à un segment :

- RAW
- PSA_X
- PCA_X

## 4. Matching

Association automatique d’une vente à une carte via :

- Détection du numéro
- Détection de la série
- Similarité du nom (fuzzy matching)
- Détection du grade

Chaque vente reçoit un score :

match_confidence ∈ [0 ; 1]

## 5. Filtrage statistique

Suppression des ventes aberrantes via la méthode IQR :

IQR = P75 - P25  
Bornes = [P25 - 1.5 × IQR ; P75 + 1.5 × IQR]

Les ventes en dehors de cet intervalle sont exclues des calculs.

## 6. Agrégation

Pour chaque segment (RAW, PSA_9, PSA_10, etc.) :

- Médiane
- Moyenne
- P25 / P75
- Confidence score

Les segments sont totalement indépendants.

---

# Modèle de base de données

## sets

- id
- name
- code
- release_date
- language

## cards

- id
- set_id
- number
- name_fr
- rarity
- image_url

## grading_companies

- id
- name

## sale_records

- id
- card_id
- source
- source_sale_id
- sold_at
- url
- currency
- price_item
- price_shipping
- price_total
- is_graded
- grading_company_id
- grade
- match_confidence

## card_price_aggregates

- card_id
- segment
- window_days
- sales_count
- median_price
- mean_price
- p25_price
- p75_price
- confidence_score
- updated_at

---

# Calcul du prix

## Prix principal

Médiane sur 30 jours pour un segment donné.

## Prix secondaire

Moyenne sur la même période.

Les segments sont calculés séparément :
RAW ≠ PSA 9 ≠ PSA 10 ≠ PCA 9

---

# Score de confiance (0 à 100)

Indique la robustesse du prix affiché.

Composé de quatre critères :

## Volume (0–40)

Basé sur le nombre de ventes :

volume_score = min(40, (sales_count / 10) × 40)

## Récence (0–20)

Basée sur la date de la vente la plus récente.

## Stabilité (0–25)

Basée sur la dispersion relative :

dispersion = (P75 - P25) / median

Moins la dispersion est élevée, plus le marché est stable.

## Matching (0–15)

matching_score = moyenne(match_confidence) × 15

## Formule finale

confidence_score =
    volume
  + récence
  + stabilité
  + matching

Score plafonné à 100.

---

# Visualisation

Deux niveaux d’affichage :

- Points individuels (chaque vente)
- Médiane glissante

Permet d’analyser :
- Tendance
- Pics
- Volatilité

---

# Système utilisateur (Phase 2)

- Création de compte
- Ajout de cartes possédées
- Sélection du grade
- Suivi de portefeuille
- Évolution automatique
- Alertes de prix

---

# Environnement local

Docker Compose :

- frontend : localhost:3000
- api : localhost:4000
- db : localhost:5432
- worker : tâches planifiées

---

# Évolutions possibles

- Extension à toutes les séries FR
- Distinction des variantes
- Ajout de nouvelles sociétés de gradation
- Indices globaux du marché Pokémon FR
- Heatmap de performance des sets
- Indicateur de volatilité
- Mise en place d’une queue (Redis / BullMQ) pour scalabilité

---

# Conclusion

PokéPrice FR est conçu comme un système statistique automatisé, transparent et évolutif permettant de produire un prix de référence fiable basé exclusivement sur des données réelles.

Architecture modulaire, traitement robuste et vision long terme structurée.
