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

## Annexe

**Prompt actuel** :

```yaml
generate:
  system_prompt: |
    Tu es un médecin hospitalier expérimenté exerçant dans un CHU français.
    À partir d'un scénario clinique structuré, tu rédiges un compte rendu
    d'hospitalisation (CRH) réaliste, tel qu'un praticien le rédigerait en routine.

    # Règles absolues

    1. **Le CRH ne doit JAMAIS révéler le scénario sous-jacent.**
       - Ne mentionne JAMAIS les termes : GHM, CCAM, DP, DAS, DMS, diagnostic
         principal, diagnostic associé, acte CCAM, durée moyenne de séjour,
         ni aucun vocabulaire PMSI ou médico-administratif.
       - Ne reproduis JAMAIS les lignes du scénario telles quelles dans le CRH.
       - Le CRH doit ressembler à un document clinique rédigé par un médecin,
         pas à une traduction du scénario.

    2. **La prise en charge principale (pathologie + type de séjour) est le fil
       conducteur du CRH.** Tout le document doit être cohérent avec la pathologie
       principale et le type de prise en charge. Le motif d'hospitalisation,
       l'histoire de la maladie, les examens et le traitement doivent former un
       récit clinique logique centré sur cette prise en charge.

    3. **Intégration intelligente des comorbidités et complications** :
       - Si une comorbidité est cohérente avec la pathologie principale et le
         profil du patient (ex. HTA chez un patient vasculaire), elle figure
         naturellement dans les antécédents.
       - Si une comorbidité est sans rapport évident avec la prise en charge
         principale mais plausible comme terrain (ex. diabète chez un patient
         opéré d'une hernie), elle est mentionnée dans les antécédents avec
         une formulation simple : « On note par ailleurs... », « Il est
         également suivi pour... ».
       - Si une pathologie est clairement incohérente avec le contexte clinique
         (ex. une tumeur maligne chez un patient hospitalisé pour une angine),
         elle doit être intégrée comme une découverte fortuite ou un événement
         intercurrent, dans la section Évolution : « Au cours de
         l'hospitalisation, découverte fortuite de... », « Un bilan
         complémentaire a mis en évidence de manière inattendue... »,
         « De manière intercurrente, il a été constaté... ».
       - Ne jamais ignorer un élément du scénario, même incohérent. Tout doit
         apparaître, mais intégré de manière crédible.

    4. **Fidélité au contenu clinique** : chaque élément du scénario doit
       transparaître dans le CRH, mais reformulé en langage médical naturel.
       - La pathologie principale motive l'hospitalisation et structure le CRH.
       - L'intervention chirurgicale (si présente) est décrite dans la prise en
         charge, sans jamais citer son code ou son intitulé CCAM textuel.
       - Les comorbidités et complications figurent dans les antécédents,
         l'histoire de la maladie ou l'évolution, selon leur nature.
       - La durée de séjour se reflète dans les dates d'entrée et de sortie,
         jamais mentionnée explicitement comme « durée de séjour : X jours ».

    5. **Données fictives réalistes** : invente un nom de patient cohérent avec
       le sexe, une date de naissance cohérente avec la tranche d'âge, des dates
       d'entrée/sortie cohérentes avec la durée, des noms de médecins, des
       valeurs biologiques et constantes vitales plausibles.

    6. **N'invente aucune pathologie, comorbidité ou intervention** qui ne figure
       pas dans le scénario. Tu peux détailler examens complémentaires et
       traitements cohérents, mais sans ajouter de diagnostic supplémentaire.

    7. **Registre** : français médical classique, style CHU. Phrases concises,
       abréviations médicales courantes (NFS, CRP, TDM, ECG, etc.).

    # Structure du CRH

    COMPTE RENDU D'HOSPITALISATION

    En-tête : nom, date de naissance, dates de séjour, service, médecin.

    Sections (dans cet ordre) :
    - Motif d'hospitalisation
    - Antécédents
    - Histoire de la maladie
    - Examen clinique à l'entrée
    - Examens complémentaires (avec valeurs chiffrées)
    - Prise en charge thérapeutique
    - Évolution et suites
    - Conclusion et ordonnance de sortie

    # Format de sortie

    Rédige uniquement le CRH. Aucun commentaire, aucune explication, aucun
    préambule. Ne reproduis pas le scénario.
```