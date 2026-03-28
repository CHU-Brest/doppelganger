# Génération automatique de Comptes Rendus d'Hospitalisation fictifs
### Réunion interne — Équipe R&D

---

## Sommaire

1. Contexte & problématique
2. Architecture du pipeline
3. Étape 1 — Génération de séjours fictifs (`get_fictive`)
4. Étape 2 — Construction des scénarios (`get_scenario`)
5. Étape 3 — Génération LLM (`get_report`)
6. Prompt engineering
7. Multi-backend LLM
8. Exemple complet : scénario → CRH
9. Stack technique
10. Perspectives

---

## 1. Contexte & problématique

### Pourquoi générer des CRH fictifs ?

Les données hospitalières réelles sont soumises à des contraintes fortes :

- **RGPD / secret médical** → impossible de partager des CRH réels
- **Rareté des données annotées** → frein au fine-tuning et aux benchmarks NLP
- **Besoin croissant** de données synthétiques pour entraîner des modèles de codage PMSI automatique, d'extraction d'entités médicales, ou de résumé clinique

### Objectif du projet

> Générer des CRH **réalistes, cohérents et non traçables**, ancrés dans les référentiels PMSI officiels (GHM, CCAM, CIM-10), via un pipeline LLM piloté par des scénarios structurés.

---

## 2. Architecture du pipeline

```
Référentiels PMSI
(DP, CCAM, DAS, DMS, CIM-10, GHM)
         │
         ▼
┌─────────────────────┐
│   get_fictive()     │  ← Tirage pondéré statistique
│  Séjour fictif      │    GHM · DP · CCAM · DAS · DMS
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│   get_scenario()    │  ← Sérialisation texte structuré
│  Prompt structuré   │    prêt pour injection LLM
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│   get_report()      │  ← Appel LLM (Ollama / Claude / Mistral)
│  CRH généré         │    + persistance Parquet batch
└─────────────────────┘
         │
         ▼
   medical_reports_*.parquet
```

---

## 3. Étape 1 — Génération de séjours fictifs (`get_fictive`)

### Principe

Chaque séjour est construit par **tirage pondéré** sur les référentiels PMSI, qui reflètent les fréquences observées dans les données hospitalières nationales.

### Pipeline interne

| Étape | Mécanisme | Détail |
|-------|-----------|--------|
| **DP** | Tirage pondéré `P_DP` | Par GHM5, AGE, SEXE |
| **CCAM** | Tirage pondéré `P_CCAM` (si GHM chirurgical `C`/`K`) | Cascade de fallback : GHM5+DP → GHM5+catégorie → GHM5 |
| **DAS** | Tirage séquentiel avec exclusion catégorielle CIM-10 `[:2]` | Cascade 5 niveaux : GHM5+AGE+SEXE+DP → GHM5 |
| **DMS** | Distribution triangulaire `(P25, P50, P75)` | Correction epsilon si distribution dégénérée |

### Sortie

```
generation_id | AGE | SEXE | GHM5 | GHM5_CODE | DP | DP_CODE | CCAM (list) | DAS (list) | DMS
```

---

## 4. Étape 2 — Construction des scénarios (`get_scenario`)

### Principe

Transformation du `DataFrame` structuré en **prompt textuel** injecté dans le LLM.

### Format du scénario généré

```
Patient : Masculin, > 80 ans.
GHM : Angine de poitrine (05M06).
Diagnostic principal : Angine de poitrine instable (I200).
Actes CCAM : Aucun.
Diagnostics associés : Infarctus du myocarde, ancien (I252),
  Autres anémies par carence en fer (D508),
  Douleur thoracique, sans précision (R074),
  Fibrillation auriculaire paroxystique (I480),
  Présence de dispositifs électroniques cardiaques (Z950).
Durée de séjour : 0 jours.
```

### Points clés

- **Libellés humains** : les codes CIM-10 et CCAM sont résolus en libellés longs (`cim10_map`, `ccam_map`)
- **Format `"libellé (code)"`** : conserve la traçabilité du code source
- **DMS = 0 jours** : cas ambulatoire — le LLM doit gérer l'entrée/sortie le même jour

---

## 5. Étape 3 — Génération LLM (`get_report`)

### Appel LLM par ligne

```python
response = client.chat(
    model=model,
    messages=[
        {"role": "system", "content": system_prompt},
        {"role": "user",   "content": df_row["scenario"]},
    ],
)
```

### Persistance intermédiaire (tolérance aux pannes)

- Chaque CRH généré est **immédiatement écrit** dans un fichier `temp.parquet`
- Flush en fichier final `medical_reports_{batch_size}_{ts}.parquet` tous les `batch_size` séjours (défaut : 1000)
- Reprise possible en cas d'interruption

### Schéma de sortie

```
generation_id | scenario | report | model | timestamp
```

---

## 6. Prompt engineering

### Objectif du prompt système

Le LLM joue le rôle d'un **médecin hospitalier de CHU** qui rédige un CRH de routine — sans jamais exposer la structure PMSI sous-jacente.

### 9 règles absolues encodées dans le prompt

| # | Règle | Détail |
|---|-------|--------|
| 1 | **Opacité PMSI** | Interdiction absolue des termes GHM, CCAM, DP, DAS, DMS — aucun vocabulaire médico-administratif |
| 2 | **Fil conducteur** | La pathologie principale structure tout le document : motif, histoire, examens, traitement |
| 3 | **Placement structurel** | Terrain → Antécédents ; comorbidités → Antécédents ± sections selon pertinence ; complications → Évolution/Conclusion uniquement ; incohérences → Évolution comme événement intercurrent |
| 4 | **Exhaustivité reformulée** | Chaque élément du scénario apparaît, traduit en langage clinique naturel |
| 5 | **Données fictives réalistes** | Nom/DDN/âge épidémiologiquement plausibles ; valeurs biologiques cohérentes avec le tableau clinique |
| 6 | **Posologies réalistes** | Référentiels HAS/Vidal ; adaptation à l'âge, poids, fonction rénale |
| 7 | **Cohérence narrative** | Fil chronologique strict : admission → bilan → prise en charge → évolution → sortie, avec connecteurs logiques |
| 8 | **Pas d'invention** | Aucune pathologie absente du scénario ne peut être ajoutée |
| 9 | **Registre CHU** | Français médical classique, abréviations courantes (NFS, CRP, TDM, ECG…) |

### Évolution clé entre v1 et v2 du prompt

La règle 3 a été entièrement refondée : v1 distinguait 3 cas génériques (cohérente / neutre / incohérente) ; v2 spécifie des **sections cibles précises** pour chaque type d'élément clinique, éliminant les ambiguïtés de placement et améliorant la cohérence structurelle des CRH générés.

### Structure imposée du CRH

```
En-tête · Motif · Antécédents · Histoire de la maladie · Examen clinique
· Examens complémentaires · Prise en charge · Évolution pendant le séjour · Conclusion
```

---

## 7. Multi-backend LLM

### Trois backends interchangeables via un wrapper commun

```python
# Interface commune : client.chat(model, messages, **kwargs)

# Local (Ollama)
client = Client(config["ollama"]["host"])

# Cloud souverain (Claude / Anthropic)
client = OllamaCompatClient(api_key=..., verify="/path/to/ca.pem")

# Cloud (Mistral)
client = MistralCompatClient(api_key=..., verify=False)
```

### Gestion SSL d'entreprise

Les wrappers `OllamaCompatClient` et `MistralCompatClient` exposent un paramètre `verify` :

| Valeur | Comportement |
|--------|--------------|
| `True` | SSL standard |
| `False` | Désactivé (dev/debug) |
| `str` | Chemin vers bundle CA `.pem` (proxy d'entreprise) |

→ Compatible avec des environnements réseau contraints (INRIA, CHU, etc.)

---

## 8. Exemple complet : scénario → CRH

### Scénario d'entrée

```
Patient : Masculin, > 80 ans.
GHM : Angine de poitrine (05M06).
Diagnostic principal : Angine de poitrine instable (I200).
Actes CCAM : Aucun.
Diagnostics associés : Infarctus du myocarde ancien (I252),
  Anémie par carence en fer (D508), Douleur thoracique (R074),
  Fibrillation auriculaire paroxystique (I480),
  Présence de dispositifs électroniques cardiaques (Z950).
Durée de séjour : 0 jours.
```

### CRH généré (extrait)

> **M. Jean-Pierre Dupont**, né le 12/03/1940 — Service Cardiologie, CHU de Lyon
>
> **Motif :** Douleur thoracique aiguë, suspicion d'angine de poitrine instable.
>
> **Antécédents :** HTA, diabète de type 2 (metformine), hypercholestérolémie, IDM 2015, pacemaker bipolaire 2018, FA paroxystique sous apixaban, anémie ferriprive (Hb 10,8 g/dL).
>
> **Examens :** TROP-I 0,35 ng/mL ↑, ST-dépression V2-V5, TTE : FEVG 55 %, Hb 10,8 g/dL, ferritine 20 ng/mL.
>
> **Évolution :** Résolution de la douleur en 4h, ECG stable. Sortie le jour même.

### Ce que le CRH ne révèle pas

- Aucune mention de GHM, CCAM, DP, DAS, DMS
- Les codes CIM-10 sont absorbés dans le récit clinique
- Le Z950 (pacemaker) devient une mention anamnestique naturelle

---

## 9. Stack technique

| Composant | Outil | Rôle |
|-----------|-------|------|
| **Traitement données** | Polars (LazyFrame) | Manipulation référentiels PMSI |
| **Statistique** | NumPy | Tirages pondérés, loi triangulaire |
| **LLM local** | Ollama | Inférence on-premise |
| **LLM cloud** | Anthropic Claude, Mistral AI | Génération haute qualité |
| **HTTP** | httpx | Gestion SSL / proxies |
| **Sérialisation** | Parquet (Polars) | Persistance batch, reprise |
| **Config** | YAML | Paramétrage prompts & chemins |
| **Référentiels** | PMSI ATIH | DP, CCAM, DAS, DMS, CIM-10, GHM |

---

## 10. Perspectives

### Court terme

- **Évaluation qualité** : scoring automatique de la cohérence clinique (LLM-as-judge)
- **Filtrage par GHM** : le paramètre `ghm5_pattern` permet de cibler des spécialités (ex. oncologie `"17"`, cardiologie `"05"`)
- **Augmentation** : variation des scénarios à partir d'un même GHM pour la diversité de dataset

### Moyen terme

- **Fine-tuning** : utilisation des paires `(scénario, CRH)` pour entraîner un modèle de codage PMSI
- **Benchmark NLP médical** : création d'un dataset de référence annoté pour l'extraction d'entités cliniques
- **Déidentification inverse** : tester si un modèle peut reconstruire le scénario PMSI depuis le CRH (évaluation de l'opacité)

### Question ouverte

> Jusqu'où le LLM peut-il **halluciner de manière cliniquement cohérente** ? Comment borner et évaluer ce risque dans un contexte de données synthétiques médicales ?
