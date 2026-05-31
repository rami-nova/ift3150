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
    **Sujet**: Collecte de données météorologiques  
    **Étudiant**: Amir Hannache  
    **Superviseur**: Fabian Bastin  

## Description du projet

### Contexte

Hydro-Québec doit planifier sa production électrique à l'avance. La demande varie beaucoup selon la température, surtout au Québec, où le chauffage électrique représente une part importante de la consommation.

Pour répondre à cet enjeu, le laboratoire UncertainLab a développé un modèle de prévision de la demande électrique à court terme. Ce modèle repose sur une série de Fourier combinée à une régression OLS, disponible ici : [ElecDemandTuto](https://github.com/UncertainLab/ElecDemandTuto).

### Problématique

Le modèle prédit la demande électrique à partir de températures futures réelles. Comme cette information n'est pas connue à l'avance, le modèle est inutilisable dans un contexte opérationnel.

### Proposition et objectifs

Remplacer la source météo CSV par des prévisions météorologiques en temps réel obtenues via la librairie Herbie, qui donne accès à plusieurs modèles.

| Modèle | Résolution | Horizon |
|---------|-----------|----------|
| HRRR | 3 km | 0–48 h |
| GFS | ~28 km | 0–16 jours |
| HRDPS | 2,5 km | 0–48 h |

L'objectif final est de développer un pipeline capable de prédire la demande électrique en temps réel à partir de prévisions météorologiques opérationnelles.


### Méthodologie envisagée

- Intégrer les prévisions HRRR au pipeline existant sans modifier le modèle de prédiction.
- Ajouter des variables météorologiques supplémentaires (rayonnement solaire, pression atmosphérique, précipitations, etc.).
- Exploiter les données de plusieurs stations météorologiques.
- Évaluer et comparer différentes combinaisons de modèles.

### Validation et évaluation

Quantifier la perte de précision du modèle (RMSE, MAPE) lorsque les températures observées du CSV sont remplacées par des prévisions météorologiques Herbie, et évaluer l'impact de l'ajout de variables explicatives, de stations météorologiques multiples et de différents modèles.