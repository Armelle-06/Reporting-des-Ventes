# 📊 Reporting Opérationnel — Microsoft Power Platform

> Projet de reporting en temps réel où les commerciaux saisissent leurs ventes sur mobile et les managers visualisent les performances instantanément sur Power BI — sans export, sans fichier Excel, sans saisie double.


## 🎯 Contexte & Problématique

Dans de nombreuses entreprises, le suivi des performances commerciales repose encore sur des fichiers Excel échangés par email, avec des risques d'erreurs, de doublons et de retards.

Ce projet répond à un besoin concret :

- **Côté terrain** : les commerciaux saisissent leurs transactions en temps réel depuis une application mobile
- **Côté management** : les managers consultent un tableau de bord Power BI mis à jour instantanément
- **Côté sécurité** : chaque commercial ne peut saisir que ses propres ventes — l'identité est injectée automatiquement

---
### Prérequis pour reproduire ce projet
- Compte Microsoft 365 (gratuit ou professionnel)
- Accès Power Platform (version gratuit sur [make.powerapps.com](https://make.powerapps.com))
- Power BI Desktop ([téléchargement gratuit](https://powerbi.microsoft.com/fr-fr/desktop/))

### Étapes
```
1. Créer les tables dans Dataverse (dimEmployé + ventesPrincipales)
2. Établir la relation 1-à-plusieurs entre les deux tables
3. Connecter Power BI à Dataverse en mode DirectQuery
4. Nettoyer les données dans Power Query (colonnes, types, renommage)
5. Créer les mesures DAX (Total Des Ventes, Rang_Employé, User Filter)
6. Construire les visuels du rapport
7. Créer l'app Power Apps et l'intégrer dans le rapport Power BI
8. Configurer le bouton Envoyer avec Patch() + PowerBIIntegration.Refresh()
9. Publier le rapport sur Power BI Service
```

## 🏗️ Architecture du projet

```
Dataverse (base centrale)
    │
    ├── Table : dimEmployé
    │       ID employé, Nom du commercial, Photo employé
    │
    └── Table : ventesPrincipales
            Numero VendeurID, Montant des ventes, Date de transaction
                │
                ▼
        Relation 1-à-plusieurs
        (un employé → plusieurs transactions)
                │
         ┌──────┴──────┐
         ▼             ▼
     Power BI      Power Apps
  (DirectQuery)   (formulaire mobile)
         │             │
         └──────┬──────┘
                ▼
         PowerBIIntegration
     (pont entre les deux outils)
```

## 🛠️ Stack technique

| Outil et Rôle |
| **Dataverse** | Base de données centrale — tables, relations, sécurité |
| **Power BI** | Visualisation des performances en temps réel (DirectQuery) |
| **Power Apps** | Formulaire de saisie mobile pour les commerciaux |
| **PowerBIIntegration** | Connexion entre Power BI et Power Apps |
| **DAX** | Mesures de classement, sécurité et calculs dynamiques |
| **Power Automate** | Automatisation des flux et du processus de consultation |

---

## 📁 Structure des tables Dataverse

### Table `dimEmployé`
| Colonne | Type | Description |
|---|---|---|
| ID employé | Entier | Identifiant unique |
| Nom du commercial | Texte | Nom affiché dans l'app et le rapport |
| Photo employé | Image | Affichée dynamiquement dans Power Apps |
| Rang Employé | Calculé (DAX) | Classement selon le total des ventes |
| User Filter | Calculé (DAX) | Sécurité — identifie l'utilisateur connecté |

### Table `ventesPrincipales`
| Colonne | Type | Description |
|---|---|---|
| Numero VendeurID | Entier | Clé étrangère vers dimEmployé |
| Montant des ventes | Décimal | Montant de la transaction |
| Date de transaction | Date/Heure | Horodatage automatique |

---

## 📐 Mesures DAX

### 1. Total des ventes

```dax
Total Des Ventes =
SUM(ventesPrincipales[Montant des ventes])
```

> Agrège toutes les transactions. Mesure de base utilisée dans tous les visuels et les autres mesures.

---

### 2. Classement dynamique des commerciaux (RANKX)

```dax
Rang_Employé =
VAR RangE1 =
    ADDCOLUMNS(
        ALLSELECTED('dimEmployé'[Nom du commercial]),
        "@Rang",
        RANKX(
            ALLSELECTED('dimEmployé'[Nom du commercial]),
            [Total Des Ventes]
        )
    )
VAR Rang_Employé1 =
    CALCULATE(
        MAX('dimEmployé'[Nom du commercial]),
        FILTER(RangE1, [@Rang] = 1)
    )
RETURN Rang_Employé1
```

> **RANKX** parcourt tous les commerciaux, compare leurs totaux de ventes et attribue un rang à chacun (1er, 2ème, 3ème...). Peu importe le filtre appliqué, la mesure recalcule le classement en temps réel. C'est ce qui alimente le podium visuel en haut du rapport.

---

### 3. Filtre de sécurité utilisateur

```dax
User Filter = USERPRINCIPALNAME()
```

> Identifie l'adresse email de l'utilisateur connecté au rapport. Couplée à la logique Power Apps, elle garantit qu'un commercial ne peut consulter et saisir que ses propres données — jamais celles d'un collègue.

---

## 📱 Formulaire Power Apps — Logique du bouton Envoyer

Quand un commercial clique sur **Envoyer**, 3 actions s'exécutent en séquence :

```powerapps
PowerBIIntegration.Refresh();;
Notify("Success"; NotificationType.Success; 10000);;
Patch(
    VentesPrincipales;
    Defaults(VentesPrincipales);
    {
        'Numero VendeurID': First([@PowerBIIntegration].Data).'Numero Vendeur ID';
        'Montant des ventes': Value(valeurVentes.Text)
    }
);;
PowerBIIntegration.Refresh()
```

| Fonction | Rôle |
|---|---|
| `Patch()` | Écrit la transaction directement dans la table Dataverse |
| `Notify()` | Affiche une confirmation visuelle à l'utilisateur (10 secondes) |
| `PowerBIIntegration.Refresh()` | Rafraîchit le rapport Power BI instantanément après la soumission |
| `First([@PowerBIIntegration].Data)` | Récupère le commercial sélectionné dans Power BI — l'ID est injecté automatiquement, jamais saisi manuellement |
| `Value(valeurVentes.Text)` | Convertit le texte saisi en nombre pour Dataverse |

---

## 🔗 PowerBIIntegration — Le pont entre Power BI et Power Apps

`PowerBIIntegration` est un objet spécial automatiquement disponible dans toute app Power Apps intégrée dans un rapport Power BI.

**Son rôle :** transmettre à l'app les données du visuel Power BI sur lequel l'utilisateur a cliqué.

Quand un manager (ou commercial) clique sur **Nicolas Bernard** dans le tableau Power BI, `PowerBIIntegration` capture la ligne sélectionnée et la rend disponible dans Power Apps :

```powerapps
// Récupérer l'ID du commercial sélectionné dans Power BI
First([@PowerBIIntegration].Data).'Numero Vendeur ID'
```

Sans `PowerBIIntegration` → l'app et le rapport sont isolés.
Avec `PowerBIIntegration` → la sélection dans Power BI pilote le comportement de l'app.

---

## 🔄 Flux de données complet

```
1. Commercial ouvre Power BI sur mobile
2. Clique sur son nom dans le tableau des performances
3. Son nom et sa photo s'affichent dans Power Apps (via PowerBIIntegration)
4. Saisit le montant de sa vente
5. Clique sur Envoyer
        │
        ├── Patch() → écrit dans Dataverse
        ├── Notify() → confirmation visuelle
        └── PowerBIIntegration.Refresh() → rapport mis à jour

6. Le manager voit les nouvelles données en temps réel
```

---

## 🔐 Sécurité des données

- **Row-Level Security** : chaque commercial ne voit que ses propres transactions
- **ID injecté automatiquement** : `PowerBIIntegration` transmet l'ID vendeur depuis Power BI — impossible de soumettre une vente au nom d'un autre
- **Validation Power Apps** : `Value()` assure que seuls des nombres valides sont soumis

---

## 📊 Visuels du rapport Power BI

| Visuel | Description |
|---|---|
| Podium (Top 3) | Photos des 3 meilleurs commerciaux, classés par `Rang_Employé` |
| Tableau de performances | Nom du commercial, Somme des ventes, Rang — filtrable |
| Formulaire Power Apps | Intégré dans le rapport — saisie contextuelle selon la sélection |
| Cards KPI | Métriques clés (total ventes, nb transactions...) |

---



## 👩‍💻 Auteure

**Armelle MATCHEU** — Analytics Engineer
Projet documenté en public sur https://www.linkedin.com/in/armelle-DataAnalyst dans le cadre d'un apprentissage Microsoft Power Platform.

## 📄 Licence

Ce projet est partagé à des fins éducatives et de portfolio. Libre d'utilisation avec mention de la source.
