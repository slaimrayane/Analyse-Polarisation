# Projet — Fondements de l'Aide à la Décision
## Mesures de Polarisation sur des Profils de Vote

---

## 📋 Table des matières

1. [Présentation générale](#présentation-générale)
2. [Structure du projet](#structure-du-projet)
3. [Prérequis et installation](#prérequis-et-installation)
4. [Partie 1 — Génération de profils de vote](#partie-1--génération-de-profils-de-vote)
5. [Partie 2 — Mesure de polarisation φ²](#partie-2--mesure-de-polarisation-φ²)
6. [Partie 3 — Distances et mesures φ_dH / φ_dS](#partie-3--distances-et-mesures-φ_dh--φ_ds)
7. [Résultats et interprétation](#résultats-et-interprétation)
8. [Auteurs](#auteurs)

---

## Présentation générale

Ce projet implémente et analyse des **mesures de polarisation** sur des profils de vote dans le cadre des **fondements mathématiques de l'aide à la décision collective**.

On s'intéresse à deux types de bulletins de vote :

| Type | Notation | Description |
|------|----------|-------------|
| **Vote par approbation** | `A^n` | Vecteur binaire `{0,1}^m` — chaque votante approuve ou non chaque candidate |
| **Vote par ordres totaux** | `L^n` | Permutation de `{0,...,m-1}` — classement complet des candidates |

L'objectif est de quantifier à quel point un profil de vote est **polarisé**, c'est-à-dire divisé en deux camps opposés.

---

## Structure du projet

```
projet/
├── Projet_Fondements.ipynb          # Notebook principal (tout le code)
├── Projet_FMPAD_2526_2.pdf          # Énoncé du projet
├── phi2_evolution.png               # Figure Q6 : évolution de φ²
├── phi_dH_dS_evolution.png          # Figure Q15 : évolution de φ_dH et φ_dS (côte à côte)
├── phi_dH_dS_comparison.png         # Figure Q15 : comparaison sur un seul graphe
└── README.md                        # Ce fichier
```

### Organisation du notebook

```
Notebook
├── Partie 1 — Votantes, Candidates et Bulletins de Vote
│   ├── Q1 : Génération aléatoire de profil dans A^n (approbation)
│   └── Q2 : Génération aléatoire de profil dans L^n (ordres totaux)
│
├── Partie 2 — Axiomes sur la Polarisation
│   ├── Q3 : Calcul des d_ckcl (preuve + code)
│   ├── Q4 : Axiomes vérifiés par φ² (preuve écrite)
│   ├── Q5 : Implémentation de φ²
│   └── Q6 : Tracé de φ² en fonction de la polarisation
│
└── Partie 3 — Distances et Mesures de Polarisation
    ├── Q7  : Preuves que dH et dS sont des distances (preuve écrite)
    ├── Q8  : Implémentation de dH et dS
    ├── Q9  : Axiomes vérifiés par φ_dH (preuve écrite)
    ├── Q10 : Calcul efficace de u*1 pour approbation (preuve)
    ├── Q11 : Calcul efficace de u*1 pour ordres totaux (preuve)
    ├── Q12 : Implémentation de u*1
    ├── Q13 : Estimation de u*2 via k-means (k=2)
    ├── Q14 : Implémentation de φ_dH et φ_dS
    └── Q15 : Tracé de φ_dH et φ_dS en fonction de la polarisation
```

---

## Prérequis et installation

### Environnement recommandé

- **Python** ≥ 3.8
- **Google Colab** ou **Jupyter Notebook**

### Dépendances

```bash
pip install numpy matplotlib scipy
```

| Bibliothèque | Version testée | Usage |
|--------------|----------------|-------|
| `numpy` | ≥ 1.21 | Calculs numériques, linspace, moyennes |
| `matplotlib` | ≥ 3.4 | Tracés et visualisations |
| `scipy` | ≥ 1.7 | `linear_sum_assignment` pour le problème d'affectation (u*1 ordres) |
| `random` | stdlib | Génération aléatoire des profils et k-means |
| `itertools` | stdlib | `combinations` pour les paires de candidates |

### Lancement

```bash
jupyter notebook Projet_Fondements.ipynb
```

Ou ouvrir directement sur [Google Colab](https://colab.research.google.com/).

---

## Partie 1 — Génération de profils de vote

### Modèle de polarisation

La génération repose sur un **modèle à deux clusters** contrôlé par un paramètre `polarization ∈ [0, 1]` :

1. On tire un bulletin de référence `a` (ou classement `r`) aléatoirement.
2. On construit son **opposé** `ā` (ou `r̄`).
3. On crée deux groupes de votantes :
   - **Cluster 1** (taille `n - k`) : votantes proches de `a` (ou `r`)
   - **Cluster 2** (taille `k`) : votantes proches de `ā` (ou `r̄`)
   - `k = floor(n/2 × polarization)`
4. Un bruit individuel est ajouté sur chaque bulletin.

| `polarization` | Profil généré | Description |
|----------------|---------------|-------------|
| `0.0` | `p_a` | Consensus total, tout le monde vote pareil |
| `0.5` | Intermédiaire | Légère division |
| `1.0` | `p_{a, ā}` | Polarisation maximale, deux camps égaux et opposés |

### API principale

```python
# Vote par approbation
profile = generate_approval_profile(n, m, polarization, noise=0.05, approval_prob=0.5)
# n, m : entiers pairs
# polarization ∈ [0, 1]
# noise : probabilité d'inversion d'une case (bruit individuel)

# Vote par ordres totaux
profile = generate_ranking_profile(n, m, polarization, noise=0.05)
# noise converti en nombre d'échanges aléatoires
```

### Représentation des données

```python
# Bulletin d'approbation : liste binaire
[1, 0, 1, 0, 1, 1]  # approuve c1, c3, c5, c6

# Classement : permutation de {0, ..., m-1}
[2, 0, 3, 1]  # c3 > c1 > c4 > c2  (indices 0-based)
```

---

## Partie 2 — Mesure de polarisation φ²

### Définition

Pour un profil `p` de `n` votantes sur `m` candidates :

```
φ²(p) = Σ_{(ck,cl) ∈ C²} (n - d_ckcl(p)) / (n × C(m,2))
```

où `d_ckcl(p) = |n_ck>cl - n_cl>ck|` est la différence absolue entre le nombre de votantes préférant `ck` à `cl` et celles préférant `cl` à `ck`.

### Valeurs extrêmes

| Profil | φ²(p) | Signification |
|--------|--------|---------------|
| `p_a` (consensus) | **0** | Tout le monde préfère les mêmes candidates |
| `p_{a, ā}` (bi-polarisé) | **1** | Les préférences se compensent exactement |

### Axiomes vérifiés

| Axiome | Énoncé | Vérification |
|--------|--------|--------------|
| **Régularité** | `φ²(p) ∈ [0, 1]` | Somme de termes `(n-d)/n ∈ [0,1]`, normalisée par `C(m,2)` |
| **Neutralité** | `φ²(p^π) = φ²(p)` | Permuter les candidates permute les paires sans changer leur ensemble |
| **Anonymité** | `φ²(σp) = φ²(p)` | Permuter les votantes ne change pas les comptages `n_ckcl` |
| **Invariance par réplication** | `φ²(kp) = φ²(p)` | `d` et `n` sont multipliés par `k`, le rapport est invariant |

### Fonctions clés

```python
# Calcul des d_ckcl
d_dict = calculer_d_approbation(profil, m)   # approbation
d_dict = calculer_d_ordres(profil, m)         # ordres totaux

# Calcul de φ²
val = phi2_approbation(profil, m)
val = phi2_ordres(profil, m)
```

---

## Partie 3 — Distances et mesures φ_dH / φ_dS

### Les deux distances

#### Distance de Hamming `dH` (approbation)

```
dH(a, a') = Σ_{i=1}^{m} 1_{a[i] ≠ a'[i]}
```
Nombre de positions où les deux bulletins diffèrent.

#### Distance de Spearman `dS` (ordres totaux)

```
dS(≺, ≺') = Σ_{i=1}^{m} |r_≺(i) - r_≺'(i)|
```
Somme des différences absolues de rangs pour chaque candidate.

Les deux sont bien des **distances métriques** (positivité, séparation, symétrie, inégalité triangulaire — prouvées dans le notebook).

### Définition de φ_dH et φ_dS

```
φ_dH(p) = (2 / (n × m))    × (u*1(p) - ũ*2(p))
φ_dS(p) = (4 / (n × m²))   × (u*1(p) - ũ*2(p))
```

| Terme | Signification |
|-------|---------------|
| `u*1(p)` | Coût optimal pour **1 centre** (consensus unique) |
| `ũ*2(p)` | Coût estimé pour **2 centres** (k-means, k=2) |

### Calcul efficace de u*1

#### Pour l'approbation (Q10)

Les coordonnées sont **indépendantes** : le problème se décompose coordinate par coordinate.

```
u*1 = Σ_j min(s_j, n - s_j)
```

où `s_j = Σ_i a_i[j]` est le nombre de votantes approuvant la candidate `j`.

Le **bulletin optimal** est le vote majoritaire : `a*[j] = 1` si `s_j > n/2`, sinon `0`.

#### Pour les ordres totaux (Q11)

Les rangs forment une permutation → les coordonnées sont **liées**. On formule un **problème d'affectation** :

```
C[c][k] = Σ_{≺' ∈ p} |k - r_≺'(c)|    (coût d'assigner le rang k à la candidate c)
```

Résolu par l'**algorithme hongrois** (`scipy.optimize.linear_sum_assignment`).

### Estimation de ũ*2 (Q13) — K-means adapté

L'algorithme k-means (k=2) est adapté aux distances `dH` et `dS` :

```
Répéter jusqu'à convergence :
  1. Affectation  : chaque votante → centre le plus proche (dH ou dS)
  2. Mise à jour  : chaque centre → consensus du cluster (= u*1 du sous-profil)
Retourner la somme des distances intra-cluster
```

On relance `n_runs` fois (défaut : 10) pour éviter les minima locaux.

### Normalisation vérifiée

| Mesure | Profil min (p_a) | Profil max (p_{a,ā}) |
|--------|------------------|----------------------|
| φ_dH | **0** | **1** (`u*1 = nm/2`, `u*2 = 0`) |
| φ_dS | **0** | **1** (`u*1 = nm²/4`, `u*2 = 0`) |

### Fonctions clés

```python
# Distances
dh = hamming(a, b)          # int
ds = spearman(r1, r2)       # int

# u*1 optimal
u1 = u_star_1_approval(profile)   # int
u1 = u_star_1_ranking(profile)    # int

# ũ*2 estimé par k-means
u2 = kmeans_approval(profile, n_runs=10)   # int
u2 = kmeans_ranking(profile, n_runs=10)    # int

# Mesures finales
val = phi_dH(profile, n_runs=10)   # float ∈ [0, 1]
val = phi_dS(profile, n_runs=10)   # float ∈ [0, 1]
```

---

## Résultats et interprétation

### Comportement des mesures

Toutes les mesures (`φ²`, `φ_dH`, `φ_dS`) sont **croissantes avec le niveau de polarisation** :

```
polarization = 0.0  →  profil très polarisé  →  mesure ≈ 1
polarization = 1.0  →  profil peu polarisé   →  mesure ≈ 0
```

> ⚠️ Attention : dans ce projet, **`polarization = 0` correspond à la polarisation maximale** (deux camps égaux et opposés) et **`polarization = 1` correspond au consensus** (tout le monde vote pareil). Les graphes sont donc décroissants de gauche à droite.

### Comparaison des trois mesures

| Mesure | Type de vote | Normalisation | Avantages |
|--------|-------------|---------------|-----------|
| **φ²** | Approbation / Ordres | C(m,2) paires | Axiomatique, exacte |
| **φ_dH** | Approbation | `nm/2` | Basée sur distance, interprétable |
| **φ_dS** | Ordres totaux | `nm²/4` | Tient compte des positions relatives |

### Figures générées

| Fichier | Contenu |
|---------|---------|
| `phi2_evolution.png` | φ² pour approbation et ordres en fonction de la polarisation |
| `phi_dH_dS_evolution.png` | φ_dH et φ_dS côte à côte avec bande d'incertitude |
| `phi_dH_dS_comparison.png` | φ_dH et φ_dS superposés sur un même axe |

---



## Auteurs
Maria-Louiza Lahlouhi
Lydia Nedir
Rayane Slaim

Projet réalisé dans le cadre du cours **Fondements Mathématiques de l'Aide à la Décision (FMPAD)** — 2025/2026.
