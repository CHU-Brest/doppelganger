# Doppelgänger — Comptes Rendus d'Hospitalisation Synthétiques (FR)

## Description

Dataset de comptes rendus d'hospitalisation (CRH) fictifs en français, générés automatiquement à partir de statistiques PMSI nationales. Chaque CRH est rédigé dans un style médical classique CHU, avec données patient fictives (noms, dates, biologie, constantes vitales).

## Méthodologie

La génération repose sur un **tirage aléatoire pondéré avec remise**. Chaque élément clinique (diagnostic, acte, comorbidité) se voit attribuer un poids proportionnel à sa fréquence observée dans l'activité hospitalière nationale. Plus un diagnostic est fréquent en pratique réelle, plus il a de chances d'être tiré. Le tirage avec remise permet à un même élément d'apparaître dans plusieurs séjours, reproduisant ainsi la quasi-distribution réelle.

Le pipeline se déroule en 4 étapes :

1. **Tirage du séjour** : un diagnostic principal (DP) est tiré selon sa fréquence nationale, associé à un profil patient (âge, sexe) et un groupe homogène de malades (GHM).
2. **Tirage des actes et comorbidités** : des actes CCAM et des diagnostics associés (DAS) sont tirés de manière pondérée, avec un mécanisme de fallback progressif pour garantir la cohérence clinique.
3. **Tirage de la durée de séjour** : une durée est tirée par distribution triangulaire (P25, P50, P75) selon le GHM.
4. **Génération du CRH** : le scénario structuré (codes et libellés CIM-10, CCAM, GHM) est envoyé à un LLM qui rédige un compte rendu d'hospitalisation en français médical.

> **Note importante** : conformément aux règles de diffusion des données de santé, les combinaisons cliniques rares (< 11 patients au niveau national) sont exclues des distributions de pondération. Aucune combinaison rare ne peut donc être reproduite par le tirage.

## Colonnes

| Colonne | Description |
|---|---|
| `generation_id` | Identifiant unique du séjour (UUID v4), attribué dès le tirage. Permet d'apparier les CRH avec d'autres données issues du même pipeline. |
| `scenario` | Scénario clinique structuré (patient, GHM, DP, actes CCAM, DAS, durée) avec codes et libellés |
| `report` | Compte rendu d'hospitalisation fictif rédigé en français médical |
| `model` | Modèle LLM utilisé pour la génération du CRH |
| `timestamp` | Date et heure de génération de la ligne |

## Fichiers
 
| Fichier | Description |
|---|---|
| `medical_reports_1000_*.parquet` | 1000 comptes rendus d'hospitalisation fictifs en français au format Parquet, avec scénario source, texte du CRH, modèle utilisé et horodatage de génération. |

## Cas d'usage

- Entraînement et évaluation de modèles NLP sur du texte médical français
- Benchmarking de systèmes d'extraction d'information clinique (NER, codage automatique)
- Prototypage d'outils DIM sans données réelles de santé
- Recherche en traitement automatique du langage médical

## Avertissement

Ces données sont **entièrement fictives**. Aucun patient réel n'est représenté. Aucune donnée source individuelle n'est publiée — seuls les textes synthétiques générés par le LLM sont diffusés. Ce dataset ne doit pas être utilisé à des fins de diagnostic ou de décision médicale.