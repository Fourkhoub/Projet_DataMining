# Projet Datamining — Sujet 8 : Le Dôme

Projet de fin de module Datamining (B3).

Le Dôme est une salle de spectacle fictive qui accueille près de 100 événements par saison. Deux problèmes reviennent à chaque comité : les files d'attente saturent certains créneaux pendant que d'autres entrées restent vides, et le partenaire restauration commande ses stocks sans savoir ce qui va se passer. La direction nous ouvre ses fichiers et nous pose neuf questions pour arrêter de piloter à l'intuition.

## Équipe

| Membre | Questions |
|---|---|
| Mehdi Tebib| Q1, Q2 |
| Lounes Djenad | Q3, Q4 |
| Moussa Tahtat | Q5, Q6 |
| Aimad Cheurfa | Q7, Q8 |
| Rayane Boukherrouf| Q9 et synthèse |

## Installation

1. Cloner le dépôt
2. Créer un environnement virtuel : `python -m venv venv`
3. L'activer :
   - Windows : `venv\Scripts\activate`
   - Mac / Linux : `source venv/bin/activate`
4. Installer les dépendances : `pip install -r requirements.txt`

## Organisation du dépôt

- `data/raw/` : fichiers fournis par Le Dôme (`Dome_evenements.csv`, `Dome_billets.csv`), jamais modifiés
- `data/processed/` : fichiers nettoyés, produits par `preprocessing.ipynb`
- `notebooks/` :
  - `eda_evenements.ipynb` : audit du fichier des événements
  - `eda_billets.ipynb` : audit du fichier des billets
  - `preprocessing.ipynb` : nettoyage et enregistrement des fichiers propres
  - `Question9.ipynb` : concentration en salle, entrées et comportements hors norme
- `slides/` : PDF de la soutenance

## Ordre d'exécution

1. `eda_evenements.ipynb` et `eda_billets.ipynb` (facultatif : ils constatent, ils ne modifient rien)
2. `preprocessing.ipynb` : **obligatoire avant toute analyse**, il produit les fichiers de `data/processed/`
3. Les notebooks des questions, qui partent uniquement des fichiers nettoyés

À la relecture des fichiers nettoyés, la date doit être reconvertie, car le format CSV ne conserve pas les types :

```python
pd.read_csv('../data/processed/evenements_clean.csv', parse_dates=['date'])
```

---

## 1. Démarche sur les données

Les fichiers sont livrés tels qu'ils sortent du système de l'entreprise. Nous avons suivi la démarche classique d'un projet de data mining (phases « compréhension des données » puis « préparation des données » du cycle CRISP-DM) :

1. **Audit** de chaque fichier, sans rien modifier : constat chiffré de chaque problème (doublons, valeurs manquantes, valeurs impossibles, unités, cohérence entre colonnes et entre fichiers).
2. **Nettoyage** dans un notebook unique, qui applique les décisions de l'audit et les vérifie par des contrôles automatiques (`assert`).

Chaque décision tient en une phrase et renvoie au constat qui la justifie. Le détail complet se trouve dans les tableaux récapitulatifs à la fin de chaque notebook d'audit.

### 1.1 Audit du fichier des événements (`eda_evenements.ipynb`)

98 lignes, 8 colonnes, conforme au sujet.

| Problème | Constat | Décision | Justification |
|---|---|---|---|
| Doublons | 2 lignes identiques (EV331, EV340) | Supprimer les copies (98 → 96) | Même événement compté deux fois |
| Type de `billets_vendus` | Lu en décimal, toutes les valeurs sont entières | Convertir en entier | Format d'export (valeurs écrites avec « .0 »), aucune perte |
| Notoriété manquante | 2 valeurs (EV362, EV303) | Imputer par la médiane du genre | La médiane est robuste pour une colonne asymétrique |
| 0 billet vendu | 3 événements (EV374, EV303, EV355), tous isolés du reste de la distribution | Reportée, puis résolue par l'audit des billets (voir 1.2) | Il fallait vérifier s'il s'agissait d'annulations |
| Dépassement de capacité | 0 sur 98 | Aucune | Ventes cohérentes avec la jauge |
| Événements complets | 5 événements, dont 4 en configuration 6 500 places | Conserver, signaler | Effet plafond : leurs ventes mesurent la jauge, pas la demande (Q2, Q3) |
| Date / `jour_semaine` | 81 incohérences sur 98, soit le niveau du hasard (≈ 14 concordances attendues, 17 observées) | Retenir `jour_semaine` pour l'analyse par jour | Profil réaliste (samedi dominant, aucun lundi), alors que la date donne une répartition uniforme sur les 7 jours |
| Plage de dates | Du 01/09/2025 au 16/09/2026, aucune date illisible | Aucune | Une saison couverte |
| Prix | De 18 € à 137 €, aucune valeur isolée | Aucune | Valeurs plausibles en euros |
| Capacité | 3 configurations (6 500, 9 800, 15 800 places) | Traiter comme catégorielle | Ce sont des jauges, pas une variable continue |
| Genre, jour | Aucune valeur manquante ni écriture incohérente | Aucune | Colonnes propres |

**Le point le plus important : la date et le jour de la semaine se contredisent.** Seuls 17 événements sur 98 ont un `jour_semaine` qui correspond à leur date, soit à peine plus que le hasard. Calculée à partir de la date, la programmation serait parfaitement uniforme (13 à 15 événements chaque jour, lundi compris), ce qui est peu plausible pour une salle de spectacle. `jour_semaine` présente au contraire un profil réaliste. Une comparaison du taux de remplissage par jour (test de Kruskal-Wallis, en vérification complémentaire) ne permet de départager aucune des deux colonnes (p = 0,22 et p = 0,19). La décision repose donc sur la plausibilité, et ce choix est signalé comme une limite.

### 1.2 Audit du fichier des billets (`eda_billets.ipynb`)

820 lignes, 12 colonnes, conforme au sujet.

| Problème | Constat | Décision | Justification |
|---|---|---|---|
| Doublons | 120 lignes identiques sur 820 (≈ 15 %), chacune en double | Supprimer les copies (820 → 700) | Ne pas fausser les statistiques ni surestimer les modèles (une copie en entraînement, l'autre en test) |
| Variables catégorielles | `zone` (4 modalités), `deja_venu` et `revient_saison_suivante` (oui / non) propres | Aucune | Colonnes propres |
| Information postérieure à l'achat | `revient_saison_suivante` n'est connue qu'après la saison | Exclue des variables explicatives de Q5 | Éviter une fuite de données |
| Âge manquant | 19 valeurs (14 billets uniques), réparties dans toutes les zones | Imputer par la médiane, avec un indicateur `age_inconnu` | Conserver les billets sans masquer qu'ils sont imputés |
| Âge de 12 ans | 48 billets à exactement 12 ans contre 9 à 13 ans ; ces « 12 ans » consomment plus au bar que les adultes (9,74 € contre 7,00 €) | Traité comme inconnu, puis imputé | Valeur plancher ou par défaut, pas un âge réel |
| Consommation manquante | 16 valeurs (14 billets uniques), profil proche de celui des consommateurs | Pas d'imputation ; billets exclus de Q5 | Un `NaN` n'est pas un 0, et c'est la cible de Q5 |
| Consommation à 0 € | 83 billets | Conserver | Valeur réelle : le spectateur n'a rien consommé |
| Prix à 0 € | 4 lignes (3 billets uniques), isolées du prix suivant (12 €) | Conserver, indicateur `invitation` | Invitations probables, à exclure des analyses de prix |
| Prix de 12 € | 28 billets empilés sur 12 €, distribution coupée à gauche | Conserver, indicateur `prix_plancher` | Plancher appliqué par le système : la vraie valeur est ≤ 12 € |
| Arrivées négatives | 33 lignes (30 billets uniques), retards de 1 à 20 minutes | Conserver | Retardataires plausibles, utiles pour Q9 |
| Délai d'achat | De 0 à 187 jours, sans rupture | Aucune | Valeurs plausibles |
| Consommation en loge | De 18 à 36 € (moyenne 28 €) contre 5 à 6 € ailleurs | Conserver | Segment réel, prédicteur clé pour Q5 et Q6 |
| Position dans la salle | Anneaux concentriques sans recouvrement (fosse, loge, gradin bas, gradin haut) | Aucune | Positions cohérentes avec les zones et avec des mètres |
| Lien avec les événements | 0 billet orphelin ; 94 événements sur 96 représentés | Aucune | Clé de liaison fiable |
| Événements à 0 billet vendu | EV374, EV303 et EV355 ont 12, 7 et 6 billets dans l'échantillon (moyenne : 8,7) | `billets_vendus` passé à inconnu ; ces événements sont exclus de Q1 à Q4 | Des spectateurs étaient présents : ce ne sont pas des annulations |
| Prix payé contre prix moyen | Rapport moyen 1,00 (de 0,59 à 1,49), sans facteur rond | Aucune | Les deux fichiers concordent |
| Représentativité | L'échantillon représente 0,15 % des ventes et n'est pas proportionnel aux ventes (corrélation 0,07) | Signaler ; pondération possible pour Q7 | Chaque événement pèse à peu près autant |

**Les deux découvertes principales de cet audit :**
- **Deux colonnes sont plafonnées par le bas** : l'âge s'empile sur 12 ans et le prix sur 12 €. Ce n'est visible qu'en regardant combien de billets tombent exactement sur la valeur minimale, et pas dans les statistiques descriptives.
- **La loge forme un segment à part** : tous les billets consommant plus de 20 € au bar sont en loge.

### 1.3 Nettoyage (`preprocessing.ipynb`)

Le notebook lit les fichiers bruts, applique les décisions dans un ordre précis (les doublons d'abord, pour que tous les calculs portent sur des lignes uniques ; les âges de 12 ans passés à inconnu avant l'imputation, pour ne pas fausser la médiane), vérifie le résultat, puis enregistre les fichiers propres.

**Événements** (`evenements_clean.csv`, 96 lignes, 9 colonnes) :
- suppression des 2 doublons ;
- ventes des 3 événements à 0 billet passées à inconnu ;
- `billets_vendus` converti en entier (`Int64`, qui accepte les valeurs manquantes) ;
- 2 notoriétés imputées par la médiane de leur genre ;
- date convertie ;
- ajout du **taux de remplissage** (billets vendus / capacité), qui va de 23,5 % à 100 % (moyenne 58 %, médiane 56 %).

**Billets** (`billets_clean.csv`, 700 lignes, 15 colonnes) :
- suppression des 120 doublons ;
- 36 âges à 12 ans passés à inconnu, soit 50 âges inconnus au total ;
- indicateur `age_inconnu`, puis imputation par la médiane (35 ans) ;
- indicateurs `prix_plancher` (26 billets) et `invitation` (3 billets) ;
- consommation au bar non modifiée (14 valeurs inconnues, 72 billets à 0 €) ;
- `deja_venu` et `revient_saison_suivante` converties en booléens.

**Vérifications** : une série de contrôles `assert` fait échouer le notebook si le résultat n'est pas celui attendu (nombre de lignes, identifiants uniques, absence d'âge à 12 ans, type numérique de l'âge, consommation non imputée, taux de remplissage ≤ 100 %, billets rattachés à un événement existant).

---

## 2. Les neuf questions

### Q1 — Qu'est-ce qui fait vraiment le remplissage ?

Notebook : `Question1.ipynb`

**Ce que nous avons compris de la question.** La direction veut savoir quels facteurs expliquent le taux de remplissage (billets vendus / capacité) : le prix, la notoriété de l'artiste, le genre, le jour de programmation ou la salle. « Vraiment » signifie séparer l'effet propre de chaque facteur au lieu de lire des corrélations simples : un artiste très connu fait payer plus cher *et* remplit davantage, le prix ne peut donc pas être jugé isolément. L'analyse porte sur 93 événements (96 moins les 3 événements aux ventes inconnues), dont 5 complets.

**Méthode.**
1. **Comprendre la cible** : distribution du taux de remplissage et statistiques descriptives.
2. **Regarder chaque facteur seul** : nuages de points pour le prix et la notoriété ; moyennes et boîtes à moustaches par genre, par jour et par capacité.
3. **Régression linéaire multiple** du taux de remplissage sur le prix moyen, la notoriété, le genre, la capacité (catégorielle) et le jour de la semaine. Les coefficients, leurs intervalles de confiance à 95 % et leurs p-values se lisent « toutes choses égales par ailleurs ». Les modalités de référence sont fixées explicitement : le genre et le jour les plus fréquents, et 6 500 places pour la capacité.
4. **Mesurer l'importance relative** de chaque facteur : on le retire du modèle et on regarde la perte de R² ajusté. On calcule aussi l'effet d'un écart-type du prix et de la notoriété, pour les comparer entre eux.
5. **Valider le modèle** par validation croisée à 5 plis, en comparant la régression linéaire complète, la même régression sans le prix, et un **KNN** (5 voisins). Un découpage classique 80 % / 20 % complète le contrôle.
6. **Tester la robustesse** : le modèle est réestimé sans les 5 événements complets, et on compare les coefficients du prix et de la notoriété.

**Pourquoi ces choix.**

| Choix | Justification |
|---|---|
| Régression linéaire multiple | Les coefficients se lisent directement et isolent l'effet de chaque facteur, ce qu'une corrélation simple ne permet pas |
| Capacité traitée comme catégorielle | Ce sont 3 jauges fixes (6 500, 9 800, 15 800 places), pas une variable continue |
| `jour_semaine` plutôt que `date` | Les deux colonnes se contredisent dans 81 cas sur 98 (voir 1.1) ; `jour_semaine` a le profil le plus plausible |
| Modalités de référence fixées | Sans cela, l'ordre alphabétique place `15800` en référence pour la capacité et les coefficients deviennent difficiles à lire |
| Validation croisée | Avec 93 événements, un seul découpage apprentissage / test est trop dépendant du tirage ; le R² calculé sur les données d'estimation est optimiste |
| KNN comme point de comparaison | Si le KNN, qui ne suppose aucune forme de relation, ne prédit pas mieux que la régression, l'hypothèse de linéarité est suffisante |
| Régression sans le prix | Si la qualité de prédiction ne baisse pas, le prix n'apporte rien au remplissage une fois les autres facteurs connus |
| K-means et DBSCAN non utilisés | Ce sont des méthodes non supervisées (groupes, anomalies) : elles ne répondent pas à une question d'explication d'une variable cible |

**Vérifications.**
- Le notebook affiche le nombre d'événements analysés (93 attendus) et de complets (5 attendus), et s'arrête si la capacité de référence est absente.
- Le graphique des résidus contrôle qu'ils sont répartis autour de 0 sans forme nette (entonnoir, courbure).
- La validation croisée se compare à une référence naïve : un modèle qui ne bat pas la prédiction « toujours la moyenne » n'explique rien.
- Les 5 événements complets sont plafonnés à 100 % : on vérifie que les conclusions tiennent sans eux.

**Résultats.** Les valeurs sont à reporter depuis les sorties du notebook après exécution :

| Élément | Valeur |
|---|---|
| Facteurs significatifs à 5 % dans le modèle | … |
| Facteur qui apporte le plus de R² ajusté | … |
| R² (validation croisée) : régression complète / sans prix / KNN | … |
| Coefficient du prix avec / sans les événements complets | … |

**Conclusion.** Le remplissage ne se réduit pas à un seul facteur. Nous retenons comme facteurs réellement associés au remplissage ceux qui remplissent les trois critères à la fois : significatifs dans le modèle, importants dans le classement par R² ajusté, stables sans les événements complets. Si retirer le prix ne dégrade pas la prédiction, le prix ne doit pas être présenté comme le levier principal du remplissage.

**Recommandations.**
1. **Agir en priorité sur les facteurs qui ressortent dans les trois contrôles** (modèle, importance, robustesse), plutôt que sur le prix seul.
2. **Présenter comme une piste, et non comme un levier établi,** tout facteur qui n'est significatif que dans un seul contrôle.
3. **Ne pas lire le taux de remplissage des soirées complètes comme une demande maximale** : le taux plafonne à 100 % alors que la demande peut le dépasser.

**Limites.**
- L'analyse est **observationnelle** : une association ne prouve pas qu'agir sur un facteur provoquerait la variation observée.
- Les **événements complets sont plafonnés à 100 %** : une partie de la demande réelle est masquée. Le test sans ces événements mesure l'effet de ce plafond sans le supprimer.
- L'échantillon est **petit** (93 événements pour une quinzaine de paramètres) : les coefficients par modalité sont peu précis, et les moyennes par genre ou par jour reposent sur peu d'événements.
- Le **jour de la semaine** est retenu sur un critère de plausibilité (incohérent avec la date dans 81 cas sur 98).
- Le **prix est choisi par les organisateurs** selon l'artiste et la salle : son coefficient n'est pas un effet causal.

### Q2 — Augmenter le prix de 10 % changerait-il les ventes ?

Notebook : `Question2.ipynb`

**Ce que nous avons compris de la question.** La direction veut savoir si une hausse de 10 % du prix moyen serait associée à une variation du nombre de billets vendus, et si le chiffre d'affaires y gagnerait malgré une éventuelle baisse des ventes. Il s'agit d'une **simulation** : on prédit les ventes de chaque événement au prix actuel, puis au prix augmenté de 10 %, toutes les autres caractéristiques restant identiques. L'analyse porte sur les mêmes 93 événements que Q1.

**Méthode.**
1. **Visualiser la relation** entre le prix moyen et les billets vendus (nuage de points).
2. **Régression linéaire** des ventes sur le prix, la notoriété, le genre, le jour et la capacité (catégorielle). Les variables numériques sont standardisées et les catégorielles encodées en colonnes 0 / 1. Le modèle est évalué sur un découpage 80 % / 20 % (MAE et R²).
3. **KNN** (régression, 5 voisins) avec la même préparation des variables et le même découpage, pour comparer les deux méthodes sur les mêmes événements de test.
4. **Simuler la hausse** : `prix_moyen_eur` est multiplié par 1,10 pour chaque événement, et les deux modèles prédisent les ventes avant et après. La variation est calculée par événement, en billets et en pourcentage.
5. **Bilan global** : ventes moyennes prédites avant et après, pour les deux modèles.
6. **Effet sur le chiffre d'affaires**, approché par prix moyen × billets prédits, avant et après la hausse.

**Pourquoi ces choix.**

| Choix | Justification |
|---|---|
| Deux modèles de nature différente | La régression linéaire suppose un effet constant du prix ; le KNN estime les ventes à partir d'événements similaires, sans supposer de forme. Si les deux convergent, la conclusion est plus solide |
| Standardisation des variables numériques | Indispensable au KNN, qui repose sur des distances : sans elle, la variable à plus grande échelle écraserait les autres |
| Capacité encodée comme catégorie | Trois jauges fixes (6 500, 9 800, 15 800 places) |
| Ventes en nombre de billets | La question porte sur les ventes et sur le chiffre d'affaires, qui s'obtient en multipliant les billets par le prix |
| Modification du seul prix | Toutes les autres caractéristiques de l'événement restent identiques pour isoler l'effet de la hausse |
| K-means et DBSCAN non utilisés | Méthodes non supervisées : elles ne prédisent pas de ventes |

**Vérifications.**
- Les performances des deux modèles sont comparées sur les mêmes événements de test (MAE et R²) : un modèle qui prédit mal ne peut pas servir à simuler.
- Les deux méthodes sont confrontées : si elles donnent des variations de sens opposé, la conclusion n'est pas établie.
- Le chiffre d'affaires est calculé avec le prix correspondant à chaque scénario (prix actuel avant, prix majoré après).

**Résultats.** Les valeurs sont à reporter depuis les sorties du notebook après exécution :

| Modèle | MAE (test) | R² (test) | Variation des ventes | Variation du chiffre d'affaires |
|---|---|---|---|---|
| Régression linéaire | … | … | … % | … % |
| KNN (k = 5) | … | … | … % | … % |

**Comment lire le résultat.** Une hausse de 10 % augmente le chiffre d'affaires tant que les ventes baissent de **moins de 9,1 %** (seuil égal à 1 − 1/1,10). L'élasticité-prix approchée est la variation des ventes en pourcentage divisée par 10.

**Conclusion.** La décision s'appuie sur les deux modèles et sur leurs performances : une variation est jugée crédible si les deux méthodes vont dans le même sens et si le modèle prédit mieux que le hasard. Si les ventes baissent de moins de 9,1 %, la hausse de 10 % reste économiquement intéressante malgré la baisse des quantités ; au-delà, le chiffre d'affaires diminue aussi.

**Recommandations.**
1. **Ne pas appliquer +10 % à toute la programmation** sur la seule base de cette simulation.
2. **Tester la hausse sur quelques événements non complets**, en gardant des événements comparables au prix actuel, puis comparer les ventes et le chiffre d'affaires.
3. **Si les deux modèles donnent une baisse des ventes, aborder la hausse avec prudence** ; si le chiffre d'affaires augmente malgré cette baisse, confirmer le résultat avec une expérimentation ou des données historiques comparables avant de généraliser.

**Limites.**
- La simulation repose sur des **données observationnelles** : elle mesure une association entre prix et ventes, pas un effet causal. Les organisateurs fixent les prix selon l'artiste et la salle, ce qui peut fausser l'effet apparent du prix.
- Les **5 événements complets sont plafonnés par la capacité** : leurs ventes mesurent la jauge et non la demande. Une hausse de prix peut avoir sur la demande réelle un effet qui n'apparaît pas dans les ventes observées.
- Les prédictions ne sont **pas bornées** entre 0 et la capacité : un modèle linéaire peut prédire plus de billets que la salle n'en contient.
- Le **KNN prédit une valeur constante par morceaux** : une hausse de 10 % ne change sa prédiction que si elle modifie les 5 événements voisins, ce qui peut donner un effet nul ou irrégulier.
- Le jeu de test ne compte qu'**une vingtaine d'événements** : le MAE et le R² varient fortement d'un tirage à l'autre.
- La simulation donne une **estimation sans intervalle d'incertitude**.
- Le chiffre d'affaires est **approché** par le prix moyen multiplié par les billets prédits : le prix réellement payé varie selon la place (rapport moyen de 1,00 avec le prix moyen, de 0,59 à 1,49 selon les billets).

### Q3 — Combien de billets annoncer au producteur trois mois avant ?
*À compléter.*

### Q4 — L'estimation tient-elle pour tous les genres ?
*À compléter.*

### Q5 — Le spectateur consommera-t-il au bar ?
*À compléter.*

### Q6 — Quels 3 000 spectateurs cibler par soirée ?
*À compléter.*

### Q7 — Combien de types de public, et à quoi ressemble chacun ?
*À compléter.*

### Q8 — Quel levier pour chaque type de public ?
*À compléter.*

### Q9 — Où se concentrent les spectateurs, quelles entrées saturent, quels comportements sortent de l'ordinaire ?

Notebook : `Question9.ipynb`

**Ce que nous avons compris de la question.** La question en contient trois : décrire la répartition des spectateurs dans la salle, identifier les entrées et les créneaux qui saturent, et repérer les comportements d'achat ou d'arrivée atypiques. La direction en attend un dispositif d'accueil concret.

#### Q9a — Où se concentrent les spectateurs ?

**Méthode.** Compter les billets par zone ne suffit pas : une zone peut en compter beaucoup simplement parce qu'elle est grande. Nous avons calculé une **densité relative** : la part des spectateurs d'une zone divisée par la part de la surface qu'elle occupe (1 = répartition uniforme). Les surfaces des zones sont estimées à partir des positions des billets, chaque zone formant un anneau autour du centre. Une carte de densité (`hexbin`) complète l'analyse.

**Pourquoi des parts plutôt que des spectateurs par m².** L'échantillon ne compte que 7 à 9 billets par événement : un nombre de spectateurs par m² n'aurait pas de sens, alors que la répartition des spectateurs entre zones, elle, est représentative.

| Zone | Part des spectateurs | Part de la surface | Densité relative |
|---|---|---|---|
| Fosse | 28,3 % | 9,9 % | **2,86** |
| Gradin bas | 34,7 % | 26,5 % | 1,31 |
| Loge | 6,4 % | 5,1 % | 1,25 |
| Gradin haut | 30,6 % | 58,5 % | **0,52** |

**Conclusion.** La fosse est près de 3 fois plus dense que la moyenne de la salle, avec un point chaud au centre ; le gradin haut est deux fois moins dense. Le gradin bas compte le plus de billets, mais c'est la fosse qui concentre le public.

#### Q9b — Quelles entrées saturent ?

**Méthode.**
1. **Définir les entrées.** Les données ne précisent ni le nombre ni l'emplacement des entrées. Hypothèse : la salle étant circulaire, elle compte 4 entrées (Nord, Est, Sud, Ouest), et chaque spectateur entre par celle du secteur de sa place. Le secteur est calculé à partir de l'angle du spectateur autour du centre. Une carte vérifie que le découpage forme bien 4 quartiers.
2. **Mesurer les arrivées** par créneau de 10 minutes et par entrée (tableau croisé, heatmap).
3. **Passer à une vraie soirée** : la part des spectateurs d'une entrée qui arrive pendant son créneau de pointe est rapportée à une soirée type (médiane : 5 639 spectateurs) puis aux trois configurations complètes, pour obtenir un flux en personnes par minute.
4. **En déduire un nombre de portiques**, avec l'hypothèse qu'un portique fait passer 20 personnes par minute.

**Résultats.**
- En volume, les 4 entrées reçoivent chacune environ 25 % des spectateurs : aucune n'est surchargée sur la soirée.
- L'essentiel des arrivées a lieu entre 20 et 60 minutes avant le début ; les retardataires restent marginaux (2 à 6 %).
- Le pic est décalé selon l'entrée : Nord 40-50 minutes avant le début, Ouest et Sud 30-40 minutes, Est 20-30 minutes.
- Au pic, une soirée médiane représente 27 à 31 personnes par minute et par entrée ; une soirée complète en 15 800 places, environ 85 personnes par minute à l'Ouest.

| Configuration | Portiques par entrée au pic |
|---|---|
| Jusqu'à 6 500 spectateurs | 2 |
| Complet 9 800 places | 3 |
| Complet 15 800 places | 5 à l'Ouest et au Sud, 4 au Nord et à l'Est |

**Conclusion.** La saturation est une question d'horaire, pas d'entrée.

#### Q9c — Quels comportements sortent de l'ordinaire ?

**Méthode.**
1. **Variable par variable**, avec la règle de l'écart interquartile (celle des boxplots), qui ne suppose pas de loi normale, ce qui convient au délai d'achat, très asymétrique.
2. **Sur plusieurs variables à la fois**, avec deux méthodes vues en cours, appliquées au délai d'achat, à l'heure d'arrivée et au prix payé, standardisés pour peser autant dans les distances :
   - **KNN** : chaque billet reçoit pour score la distance à son 10e plus proche voisin ; les 2 % les plus isolés sont retenus ;
   - **DBSCAN** (`min_samples = 6`, `eps = 1,0`) : les billets qui ne se rattachent à aucun groupe dense sont classés en bruit. `eps` est lu au coude du graphe des distances au 5e plus proche voisin.

**Pourquoi ces choix.** DBSCAN est la méthode du cours qui isole naturellement les points atypiques, sans imposer de nombre de groupes. Les variables de position ont été exclues : avec elles, les spectateurs du gradin haut, zone peu dense, ressortaient comme atypiques à cause de leur place et non de leur comportement.

**Vérifications.**
- Les deux méthodes isolent le même profil.
- **Test de robustesse** : DBSCAN relancé avec `eps` à 0,8, 1,0 et 1,2.

| eps | Groupes | Anomalies | Achat (jours avant) | Arrivée (min avant) | Prix (€) |
|---|---|---|---|---|---|
| 0,8 | 2 | 22 | 91,5 | 52,8 | 86,7 |
| 1,0 | 1 | 13 | 113,2 | 54,8 | 83,9 |
| 1,2 | 1 | 8 | 148,2 | 41,5 | 71,6 |
| *Reste du public* | | | *40,5* | *38,9* | *69,5* |

**Résultats.**
- Selon la règle de l'écart interquartile, 25 billets ont été achetés anormalement tôt (plus de 112 jours avant) et 3 spectateurs sont arrivés anormalement en avance (plus de 106 minutes) ; aucun retard n'est hors norme.
- DBSCAN ne trouve qu'un seul groupe : le public n'a pas plusieurs façons d'acheter et d'arriver nettement séparées.
- Environ 2 % des spectateurs ressortent comme atypiques. Leur trait robuste est l'**achat très anticipé** : environ 110 jours avant l'événement contre 41 en moyenne, et jusqu'à 5 mois pour les cas les plus extrêmes. Ils ont tendance à payer plus cher et à arriver plus tôt, mais ces deux traits ne se retrouvent pas chez les cas les plus extrêmes.
- Ce ne sont ni des invitations ni des prix plancher.

**Conclusion.** Aucun comportement ne relève de l'erreur ou de la fraude ; le seul profil atypique est celui des spectateurs qui réservent plusieurs mois à l'avance.

#### Recommandations de la question 9

1. **Dimensionner l'accueil selon la configuration de la soirée** : 2 portiques par entrée jusqu'à 6 500 spectateurs, 3 pour un complet en 9 800 places, 4 à 5 pour un complet en 15 800 places.
2. **Décaler l'ouverture des portiques supplémentaires** selon le pic de chaque entrée (Nord en premier, Est en dernier), pour que le même personnel passe d'une entrée à l'autre.
3. **Renforcer le placement et la sécurité en fosse**, où la densité est la plus forte, en particulier au centre.
4. **Cibler les spectateurs qui réservent très tôt** avec des préventes, avant-premières ou abonnements. Aucune mesure de contrôle particulière n'est nécessaire.

#### Limites de la question 9

- Les entrées sont déduites de la position des spectateurs : leur nombre et leur emplacement réels sont à confirmer avec l'exploitation.
- Le débit d'un portique (20 personnes par minute) est une hypothèse à remplacer par la valeur réelle.
- L'échantillon compte environ 175 billets par entrée : la forme générale du pic d'arrivées est solide, le détail entrée par entrée est une tendance.
- La densité est calculée sur une surface au sol, alors que les gradins sont en pente ; les surfaces des zones sont estimées à partir des positions observées.
- Les paramètres de détection sont des choix ; le test de robustesse montre que seul l'achat très anticipé est un trait constant des anomalies.

---

## 3. Synthèse : le plan de saison

*À compléter : prévision de remplissage (Q1 à Q4), dispositif d'accueil par créneau (Q9), offres ciblées (Q6, Q8) et gain attendu au bar (Q6).*

## 4. Limites générales

- Le fichier des billets est un **échantillon** (700 billets uniques, 0,15 % des ventes), tiré à peu près au même nombre de billets par événement, et non proportionnellement aux ventes.
- Le jour de la semaine retenu (`jour_semaine`) est incohérent avec la date dans 81 cas sur 98 ; il a été choisi sur un critère de plausibilité.
- Plusieurs valeurs sont plafonnées par le système (âge à 12 ans, prix à 12 €) : la vraie valeur de ces billets est inconnue.