# PokéPrice FR

## Objectif du projet

PokéPrice FR est une plateforme de référence permettant de répondre à une question simple :

> À combien puis-je vendre ma carte Pokémon ?

Le projet vise à fournir un prix fiable basé uniquement sur des ventes réelles, avec une expérience utilisateur simple, intuitive et minimaliste.

Philosophie centrale :

Fiabilité > Spéculation  
Données réelles > Annonces actives  

---

# Vision du projet

## MVP (Phase 1)

- Cartes Pokémon FR uniquement
- Séries des 4 à 5 dernières années
- Une seule entrée par numéro de carte
- Segments :
  - RAW
  - PSA
  - PCA
- Calcul basé sur ventes complétées
- Score de confiance
- Courbe d’évolution
- Affichage des X dernières ventes

## Vision long terme

- Toutes les séries FR
- Distinction holo / reverse / variantes
- Plus de sociétés de gradation
- Comparaison sold price vs listing price
- Dashboard collection utilisateur
- Alertes de prix
- API publique
- Analyse statistique globale (évolution Pikachu, rareté, etc.)

---

# Sources de données & APIs

## Catalogue des cartes

### TCGdex API
Utilisée pour importer :
- Séries FR
- Cartes
- Numéros
- Raretés
- Images

TCGdex fournit une API REST et GraphQL multilingue incluant le français.

Rôle dans le projet :
- Remplir la base `sets`
- Remplir la base `cards`
- Mise à jour automatique lors des nouvelles sorties

---

## Prix & Ventes

### Objectif : ventes réelles uniquement

### Sources envisagées :

#### 1 eBay (ventes complétées)
- Récupération des ventes "sold"
- Extraction :
  - Prix
  - Date
  - Titre
  - Lien
  - Devise
- Normalisation en EUR

 Note :
Les APIs officielles eBay ont des limitations concernant les sold data.
Selon la configuration, une solution tierce ou alternative peut être nécessaire.

---

#### 2 Cardmarket (benchmark agrégé)

- Utilisation possible du Price Guide (trend / average)
- Sert de référence secondaire
- Ne remplace pas les ventes unitaires

---

## Philosophie de fiabilité

Les annonces actives ne sont PAS utilisées pour le prix de référence.
Elles peuvent être affichées séparément dans le futur comme indicateur de marché.

---

# Architecture générale

Le système est divisé en 6 modules :

## 1 Import Catalogue
- Récupération via TCGdex
- Mise à jour planifiée

## 2 Collecteur de ventes
- Job planifié (cron)
- Récupération nouvelles ventes
- Stockage brut dans `sale_records`

## 3 Normalisation
- Conversion devise
- Calcul prix total (objet + frais)
- Détection grade PSA/PCA
- Nettoyage titre
- Exclusion lots

## 4 Matching automatique
Association vente → carte

Méthodes :
- Détection numéro carte
- Détection série
- Fuzzy matching nom
- Détection PSA/PCA + grade

Chaque vente reçoit un :
`match_confidence ∈ [0 ; 1]`

## 5 Filtrage statistique
- Méthode IQR
- Exclusion anomalies

## 6 Moteur de calcul
- Médiane
- Moyenne
- P25 / P75
- Score de confiance

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
- name (PSA, PCA)

## sale_records
- id
- card_id
- source
- source_sale_id
- sold_at
- title
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
- segment (RAW, PSA_10, PCA_9...)
- window_days
- sales_count
- median_price
- mean_price
- p25_price
- p75_price
- confidence_score
- updated_at

---

# Moteur de calcul des prix

## 1 Filtrage

Exclusion automatique des mots-clés :
- lot
- bundle
- collection
- x10

Suppression des outliers via IQR :

dispersion = (P75 - P25) / median

---

## 2 Prix de référence

Sur 30 jours :

Prix principal = Médiane  
Prix secondaire = Moyenne  

---

# Score de confiance (0 à 100)

Le score indique la fiabilité du prix affiché.

## 1 Volume (0-40)
Basé sur le nombre de ventes :

volume_score = min(40, (sales_count / 10) * 40)

10 ventes = score max

---

## 2 Récence (0-20)

Basé sur la vente la plus récente :

< 3 jours → 20  
< 7 jours → 15  
< 14 jours → 10  
< 30 jours → 5  
> 30 jours → 0  

---

## 3 Stabilité (0-25)

Basé sur la dispersion :

< 10% → 25  
< 20% → 20  
< 40% → 15  
< 60% → 8  
> 60% → 0  

---

## 4 Matching (0-15)

matching_score = moyenne(match_confidence) * 15

---

## Formule finale

confidence_score =
    volume
  + récence
  + stabilité
  + matching

Score plafonné à 100.

---

# Courbe d’évolution

Deux modes :

- Points individuels (chaque vente)
- Médiane glissante

Permet de visualiser :
- Tendance
- Pics
- Stabilité du marché

---

# Système utilisateur (Phase 2)

- Création de compte
- Ajout cartes possédées
- Sélection grade
- Dashboard portefeuille
- Évolution automatique
- Alertes prix

---

#  Roadmap

## MVP
- Import séries récentes
- Collecteur ventes
- Matching automatique
- Calcul prix + confiance
- Pages : Accueil → Série → Carte

## V1
- Optimisation performance
- Historique long
- Amélioration matching
- Ajout nouvelles sources

## V2
- Comptes utilisateurs
- Dashboard collection
- Watchlist
- Alertes

## V3
- Toutes les séries FR
- Variantes holo/reverse
- Comparaison listing vs sold
- API publique
- Analyse statistique globale

---

#  Potentiel futur

- Analyse évolution par pokemon toutes séries confondues
- Évolution par rareté
- Indice global marché Pokémon FR
- Indicateur volatilité

---


# Conclusion

PokéPrice FR est conçu pour devenir une référence fiable du marché Pokémon FR en combinant :

- Ventes réelles
- Transparence
- Statistiques robustes
- Expérience utilisateur claire
- Vision long terme ambitieuse
