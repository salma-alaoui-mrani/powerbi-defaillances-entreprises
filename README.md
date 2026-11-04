# Défaillances d'entreprises en France : tableau de bord Power BI

Tableau de bord Power BI sur les défaillances d'entreprises en France, par secteur d'activité et par région, de 2015 au 2e trimestre 2026. Les données sont celles de la Banque de France.

**Question métier :** dans quels secteurs et dans quelles régions les entreprises font-elles le plus défaillance, et le risque s'aggrave-t-il ? C'est une question centrale pour le **risque de crédit** : une entreprise défaillante ne rembourse plus ses prêts.

![Page 1 : vue d'ensemble](Vue_d'ensemble.png)
![Page 2 : secteurs et régions](Secteurs_et_régions.png)

---

## Principaux résultats

| Constat | Chiffre |
|---|---|
| Défaillances sur les 12 derniers mois, à fin juin 2026 | **68 093**, soit **+38 %** par rapport à 2019 (49 487) |
| Effet des aides publiques pendant la crise Covid | **-39 %** en 2020 |
| Rattrapage après la fin des aides | **+51 %** en 2022, **+36 %** en 2023 |
| Tendance récente | **+3,2 %** en 2025, puis **+6,2 %** au 1er semestre 2026 sur un an |
| Concentration sectorielle (12 derniers mois) | Construction et commerce : **40 %** des défaillances |
| Secteurs en accélération (1er semestre 2026 sur un an) | Agriculture **+15 %**, santé-social **+14 %**, finance-assurance **+12 %** ; construction quasi stable (**+1,3 %**) |
| Concentration régionale (12 derniers mois) | Île-de-France : **24 %** des défaillances ; plus forte hausse au 1er semestre 2026 : Auvergne-Rhône-Alpes (**+18 %**) |

---

## Données

- **Source :** Banque de France, portail Webstat, jeu de données **DIREN** (Observatoire des entreprises).
- **Série :** nombre de défaillances d'unités légales, cumul trimestriel, données brutes, toutes tailles d'entreprise.
- **Périmètre :** 13 régions de France métropolitaine × 11 secteurs d'activité = **143 séries**, du T1 2015 au T2 2026, soit **6 578 lignes**.
- **Export :** format long (une observation par ligne), CSV. [Lien vers la sélection Webstat](https://webstat.banque-france.fr/fr/catalogue/diren/?publication_status=up_to_date,upcoming,discontinued&sort=-greatest_updated_at,title_fr&DIREN,freq=Q&DIREN,diren_sectact=AZ,BE,FZ,G,H,I,JZ,KZ,LZ,MN,PS&DIREN,ref_area=R11,R24,R27,R28,R32,R44,R52,R53,R75,R76,R84,R93,R94)

**Choix de la série.** La Banque de France publie aussi un « cumul sur 12 mois ». Je ne l'ai pas utilisé comme donnée source : additionner des cumuls glissants compterait chaque défaillance plusieurs fois. J'ai pris le total de chaque trimestre et je calcule le cumul 12 mois en DAX.

---

## Démarche

### 1. Préparation des données (Power Query)
- Sélection des 4 colonnes utiles : date de début de trimestre, code région, code secteur, nombre de défaillances.
- Filtre sur la période 2015 → 2026.
- Renommage clair des colonnes et des tables.

### 2. Modèle en étoile

```
                Dim_Date
                    │ 1
                    │
                    ▼ *
Dim_Region ──1──▶ Faits_Defaillances ◀──1── Dim_Secteur
```

- **Faits_Defaillances** : Date, Code_region, Code_secteur, Nb_defaillances.
- **Dim_Region** : code Insee de la région (R11…) → nom de la région.
- **Dim_Secteur** : code NAF (FZ…) → libellé officiel et libellé court pour les graphiques.
- **Dim_Date** : calendrier DAX (`CALENDAR`) marqué comme table de dates, avec l'année, le trimestre et une étiquette Année_Trimestre.
- Relations un-à-plusieurs. Les clés de la table de faits sont masquées, et la colonne brute est remplacée par une mesure.

### 3. Mesures DAX

```dax
Total Defaillances = SUM( Faits_Defaillances[Nb_defaillances] )
```

```dax
Defaillances N-1 =
VAR DerniereDate =
    CALCULATE( MAX( Faits_Defaillances[Date] ), ALL( Faits_Defaillances ) )
VAR DatesAvecDonnees =
    FILTER( VALUES( Dim_Date[Date] ), Dim_Date[Date] <= DerniereDate )
RETURN
    CALCULATE( [Total Defaillances], SAMEPERIODLASTYEAR( DatesAvecDonnees ) )
```

```dax
Evolution N-1 % =
DIVIDE( [Total Defaillances] - [Defaillances N-1], [Defaillances N-1] )
```

```dax
Cumul 12 mois =
VAR DerniereDate =
    CALCULATE( MAX( Faits_Defaillances[Date] ), ALL( Faits_Defaillances ) )
VAR PremiereDate =
    CALCULATE( MIN( Faits_Defaillances[Date] ), ALL( Faits_Defaillances ) )
VAR FinPeriode =
    MIN( MAX( Dim_Date[Date] ), DerniereDate )
RETURN
    IF(
        MIN( Dim_Date[Date] ) > DerniereDate
            || FinPeriode < EDATE( PremiereDate, 9 ),
        BLANK(),
        CALCULATE(
            [Total Defaillances],
            DATESINPERIOD( Dim_Date[Date], FinPeriode, -12, MONTH )
        )
    )
```

```dax
Part du total % =
DIVIDE(
    [Total Defaillances],
    CALCULATE( [Total Defaillances], ALL( Dim_Secteur ), ALL( Dim_Region ) )
)
```

```dax
Moyenne par trimestre =
AVERAGEX( VALUES( Dim_Date[Annee_Trimestre] ), [Total Defaillances] )
```

**Trois biais corrigés dans les mesures :**
1. **Année incomplète.** Les données s'arrêtent en juin 2026. Sans correction, le 1er semestre 2026 serait comparé à toute l'année 2025, ce qui afficherait une fausse baisse de 45 %. La mesure N-1 ne compare que les périodes disponibles : 6 mois contre 6 mois.
2. **Cumul incomplet en début d'historique.** Les trois premiers trimestres de 2015 n'ont pas 12 mois d'historique. Le cumul est laissé vide plutôt que d'afficher une fausse hausse.
3. **Périodes futures.** Aucune valeur n'est affichée après la dernière donnée disponible.

### 4. Contrôles qualité
Une page masquée du rapport (« Contrôles ») sert à recouper les mesures :
- volume : 143 séries × 46 trimestres = **6 578 lignes**. Ce contrôle a permis de détecter un trimestre manquant, à cause d'un filtre « après » au lieu de « postérieur ou égal » ;
- total de la période : **588 871** défaillances, recalculé hors Power BI ;
- cohérence avec la Banque de France : environ 66 000 défaillances en 2025 sur le périmètre retenu, contre environ 68 600 pour la France entière. L'écart correspond aux DOM et aux entreprises dont le secteur est inconnu, exclus ici.

### 5. Restitution (2 pages)
- **Vue d'ensemble :** 4 indicateurs clés (total, évolution sur un an, cumul 12 mois, moyenne par trimestre), courbe du cumul 12 mois de 2015 à 2026, part de chaque secteur.
- **Secteurs et régions :** évolution sur un an par secteur, avec mise en forme conditionnelle au-delà de +10 %, volumes par région, et carte de chaleur région × secteur.
- Filtres par année, région et secteur, avec filtrage croisé. La courbe historique ignore le filtre d'année, grâce aux interactions modifiées.
- Thème Power BI et fonds de page personnalisés (dossier `template/`).

---

## Limites
- **Des volumes, pas des taux.** L'Île-de-France compte le plus de défaillances, en partie parce qu'elle compte le plus d'entreprises. Le vrai indicateur de risque serait un taux de défaillance (défaillances ÷ entreprises actives). C'est la prochaine étape.
- **Petits effectifs.** Certaines cases de la carte de chaleur reposent sur quelques dizaines de défaillances : les évolutions y sont très volatiles.
- **Périmètre :** France métropolitaine uniquement ; DOM et secteurs inconnus exclus.
- **Données révisées.** La Banque de France révise ses chiffres chaque mois. Les derniers trimestres peuvent encore évoluer.

---

## Ouvrir le projet
1. Installer **Power BI Desktop** (Windows).
2. Ouvrir `defaillances_entreprises.pbix`.
3. Si le chemin du fichier a changé : **Accueil → Transformer les données → Paramètres de la source de données**, puis pointer vers `data/Webstat_Export_fr_DIREN.csv`.

```
projet-defaillances/
├── README.md
├── defaillances_entreprises.pbix
├── Webstat_Export_fr_DIREN.csv
├── page1_vue_ensemble.png
└── page2_secteurs_regions.png
```

---

**Outils :** Power BI Desktop · Power Query · DAX · modélisation en étoile

**Auteure :** Salma Alaoui Mrani, Master 2 Big Data, Analyse et Business Intelligence, Université Sorbonne Paris Nord

*Données : Banque de France, Webstat (jeu DIREN). Réutilisées avec citation de la source.*
