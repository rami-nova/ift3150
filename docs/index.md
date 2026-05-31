---
title: Vue d'ensemble du projet
---

<style>
    @media screen and (min-width: 76em) {
        .md-sidebar--primary {
            display: none !important;
        }
    }
</style>

# Vue d'ensemble du projet

!!! info "Informations générales"
    **Session**: Été 2026  
    **Auteur(s)**: Amir Hannache (20060308)  
    **Thème(s)**: Prévision de demande électrique, données météo NWP, Hydro-Québec  
    **Superviseur(s)**: UncertainLab, Université de Montréal  
    **Collaborateur(s):** Hydro-Québec  

## Description du projet

### Contexte

Le laboratoire UncertainLab a développé un modèle de prévision de demande électrique court terme pour le Québec, disponible sur GitHub ([ElecDemandTuto](https://github.com/UncertainLab/ElecDemandTuto)). Ce modèle — basé sur une série de Fourier et une régression OLS — apprend les habitudes de consommation horaire à partir des données historiques d'Hydro-Québec (2019–2022) et de données météo de la station Montréal Intl A.

La demande électrique au Québec est fortement liée à la température : le chauffage électrique représente une part majeure de la consommation résidentielle. Prédire la demande à l'avance nécessite donc des **prévisions météo** fiables.

### Problématique

Le modèle existant utilise un fichier CSV statique d'observations passées (`montreal_weather.csv`). Pour simuler une prédiction, il consulte la **vraie température de l'heure cible** directement dans ce fichier — information qui n'existe pas encore en production réelle.

Ce fonctionnement en mode *backtesting* rend le modèle inutilisable en conditions réelles : il triche en accédant aux données futures. La question centrale est donc :

> **Comment rendre ce pipeline utilisable en production, et quel est le coût de ce passage d'observations parfaites à des prévisions réalistes ?**

### Proposition et objectifs

L'objectif est de remplacer la source météo CSV par des prévisions issues de la librairie Python **Herbie**, qui donne accès aux modèles de prévision numérique du temps (NWP) de la NOAA via AWS :

| Modèle | Résolution | Horizon |
|--------|-----------|---------|
| HRRR | 3 km | 0–48h |
| GFS | ~28 km | 0–16 jours |
| HRDPS | 2.5 km | 0–48h |

**Objectifs mesurables :**

1. Remplacer la météo CSV par des prévisions Herbie HRRR sans modifier le modèle de prédiction
2. Quantifier la dégradation de précision (RMSE, MAPE) CSV → Herbie sur au moins une journée de test
3. Analyser l'impact de l'horizon de prévision (fxx = 1h, 6h, 12h, 24h) sur la précision — sur 4 saisons
4. Enrichir le modèle avec des variables météo supplémentaires disponibles dans HRRR
5. Préparer le terrain pour la combinaison multi-modèles (HRRR + GFS + HRDPS)

### Méthodologie

Le projet suit une progression par étapes, chacune validée avant de passer à la suivante :

**Étape 1 — Compréhension du modèle de base** *(réalisée)*  
Lecture et analyse du repo ElecDemandTuto. Identification du problème de backtesting et des variables météo utilisées. Découverte d'un mislabel dans le code original : la colonne `rel_hum` est en réalité le dew point (°C), pas l'humidité relative (%).

**Étape 2 — Branchement de Herbie** *(réalisée)*  
Création de `fetch_weather_for_model()` dans `scripts/fetch_herbie_data.py` : interroge Herbie HRRR, convertit les unités (Kelvin → °C, humidité → dew point, wind chill), et retourne un dictionnaire compatible avec les colonnes du CSV original. Intégration dans `get_data.py` via `get_history_with_herbie()`.

**Étape 3 — Comparaison avant/après** *(réalisée)*  
Notebook `analyse_complete.ipynb` : comparaison CSV vs Herbie HRRR (fxx=6h) sur le 15 janvier 2021, avec tableau HTML, métriques RMSE/MAPE et visualisation des courbes et erreurs.

**Étape 4 — Analyse multi-horizons** *(réalisée)*  
Notebook `analyse_horizons.ipynb` : boucle sur 4 journées (hiver, printemps, été, automne 2021) et 4 horizons (fxx = 1h, 6h, 12h, 24h), avec matrice RMSE, heatmap et comparaison par saison.

**Étape 5 — Variables supplémentaires** *(réalisée)*  
Ajout de `solar_rad`, `pressure` et `precip` aux prévisions Herbie. Section dédiée dans les deux notebooks reproduisant le même protocole d'évaluation.

**Prochaines étapes :**

- Ajouter plusieurs stations météo (Québec, Sherbrooke, Saguenay...) pour une meilleure couverture spatiale
- Combiner HRRR + GFS + HRDPS (moyenne pondérée ou méta-modèle)

### Validation et Évaluation

**Métriques :**

- **RMSE** (Racine de l'erreur quadratique moyenne, en MW) — écart absolu entre prédiction et réalité
- **MAPE** (Erreur absolue moyenne en pourcentage) — écart relatif pour comparer les journées

**Protocole :**

- Entraînement : 2019–2020 (données CSV Hydro-Québec, inchangées)
- Test : journées de 2021, hors période d'entraînement
- Baseline : prédictions avec météo observée CSV (borne inférieure d'erreur atteignable)
- Comparaison : CSV vs Herbie HRRR à différents horizons fxx

**Résultats obtenus — 15 janvier 2021, fxx = 6h :**

| Version | RMSE | MAPE |
|---------|------|------|
| CSV (baseline) | 750 MW | 2.72 % |
| Herbie HRRR | 1 481 MW | 4.78 % |

L'écart de ~730 MW représente le coût de réalisme du pipeline : passer d'observations parfaites à des prévisions introduit une incertitude supplémentaire attendue. L'analyse multi-horizons confirme que des horizons courts (fxx=1h) donnent des résultats proches du CSV, alors que des horizons longs (fxx=24h) dégradent davantage la précision.

### Mise en commun

Ce projet produit des outils réutilisables par les autres équipes travaillant sur des sujets connexes :

- **Pipeline Herbie → modèle de demande** : la chaîne `fetch_herbie_data.py` → `get_data.py` → `fourier_series.py` est modulaire et adaptable à d'autres localisations ou d'autres modèles NWP
- **Protocole de comparaison CSV vs NWP** : les notebooks servent de template pour évaluer n'importe quelle source météo alternative
- **Bugs identifiés** : convention de longitude HRRR (0→360) et mislabel `rel_hum` dans le code original — utiles pour tout projet utilisant le même repo ElecDemandTuto

**Points à discuter lors des séances de mise en commun :**

- Quel modèle NWP choisir selon l'horizon de prévision ?
- Comment combiner HRRR + GFS + HRDPS de façon efficace ?
- Est-ce qu'une seule station météo (Montréal) est suffisante pour représenter la demande électrique de tout le Québec ?

---

## Équipe

| Membre | Matricule | Rôle |
|--------|-----------|------|
| Amir Hannache | 20060308 | Développement et expérimentation |

**Superviseur :** UncertainLab, Université de Montréal  
**Partenaire :** Hydro-Québec

---

## Échéancier

!!! info
    Le suivi complet est disponible dans la page [Suivi de projet](suivi.md).

| Activités | Début | Fin | Livrable | Statut |
|-----------|-------|-----|----------|--------|
| Compréhension du modèle de base | 4 mai | 15 mai | Notes d'analyse | ✅ Terminé |
| Branchement Herbie (fetch + intégration) | 15 mai | 22 mai | `fetch_herbie_data.py`, `get_data.py` | ✅ Terminé |
| Comparaison CSV vs Herbie | 22 mai | 31 mai | `analyse_complete.ipynb` | ✅ Terminé |
| Analyse multi-horizons (4 saisons × 4 fxx) | 24 mai | 31 mai | `analyse_horizons.ipynb` | ✅ Terminé |
| Variables météo supplémentaires | 28 mai | 31 mai | ÉTAPE 3 dans les deux notebooks | ✅ Terminé |
| Ajout de stations météo multiples | juin | juillet | Pipeline multi-stations | ⏳ À venir |
| Combinaison de modèles (HRRR + GFS + HRDPS) | juillet | août | Pipeline multi-modèles | ⏳ À venir |
| Présentation finale + Rapport | août | août | Présentation + Rapport PDF | ⏳ À venir |
