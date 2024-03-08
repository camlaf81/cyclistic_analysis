# Exploratory Data Analysis

Maintenant que les données sont rassemblées dans une table et nettoyées, on peut commencer l'exploration. L'objectif est d'identifier les différences de comportement entre les membres (abonnés annuels) et les utilisateurs occasionnels.

## Aggrégation et exportation des données
Je décide d’agréger les données par type d’utilisateur, type de vélo, mois, jour de la semaine. Cela me permettra d’identifier d’éventuelles tendances au cours d’une semaine, au cours de l’année, et d’évaluer si l’utilisation par les abonnés annuels diffère de celle des utilisateurs occasionnels. Je complète les données en calculant la durée et la distance des trajets.
En partant de la table `cyclistic_clean_data.clean_data`, j’exécute donc la requête suivante :
```sql
SELECT
  user_type,
  rideable_type,
  EXTRACT(MONTH FROM start_time) AS month,
  EXTRACT(DAYOFWEEK FROM start_time) AS day,
  COUNT(EXTRACT(DAYOFWEEK FROM start_time)) AS n_of_rides,
  ROUND(AVG(
    CAST(SPLIT(duration, ':')[0] AS INT64) * 3600
    + CAST(SPLIT(duration, ':')[1] AS INT64) * 60
    + CAST(SPLIT(duration, ':')[2] AS INT64)
    ), 3) AS mean_duration_s,
  ROUND(AVG(
    ST_DISTANCE(
      ST_GEOGPOINT(start_lng, start_lat),
      ST_GEOGPOINT(end_lng, end_lat))
    ), 1) AS mean_distance_m
FROM
  cyclistic_clean_data.clean_data
GROUP BY
  user_type,
  rideable_type,
  month,
  day
ORDER BY
  user_type,
  rideable_type,
  month,
  day
```
J'exporte les résultats de cette requête en CSV pour les étudier dans Sheets.




## Généralités sur l'utilisation des vélos
5 716 768 trajets ont été effectués entre août 2022 et juillet 2023.

### <ul><li>Utilisateurs</ul></li>
On peut regarder la répartition des trajets selon le type d'utilisateurs :

<img src="img/EDA/users.png" width=40%>

Près de 40% des trajets sont effectués par des utilisateurs occasionnels. Il y a donc une marge d’évolution considérable pour fédérer de nouveaux abonnés annuels.

### <ul><li>Type de vélos</ul></li>

Pour chaque catégorie d’utilisateurs, on peut regarder l’utilisation des différents types de vélos.

<img src="img/EDA/bike_distribution__members.png" width=40%> <img src="img/EDA/bike_distribution__casual.png" width=40%>

Ce sont majoritairement les vélos électriques qui sont utilisés chez les utilisateurs occasionnels. Ces utilisateurs sont également les seuls à utiliser des vélos cargo (“docked bike”). *NB : Peut-être que ces vélos ne sont pas ouverts à la location dans le cadre de l’abonnement annuel, ce qui expliquerait l’absence totale de vélos cargo utilisés par les abonnés.*

Pour les abonnés annuels, l’utilisation est plus équilibrée entre vélos électriques et classiques, mais avec néanmoins une légère avance pour les vélos électriques.



## Tendances selon les jours de la semaine
### <ul><li>Nombre de trajets</ul></li>

Je rajoute dans la table deux colonnes pour afficher les noms de jours de la semaine et des mois de l’année, grâce à `VLOOKUP` :
- je crée une nouvelle feuille `days month for VLOOKUP`
- pour extraire les noms des mois : `= VLOOKUP(C2, 'days month for VLOOKUP'!C:D, 2)`
- pour extraire les noms des jours : `= VLOOKUP(D2, 'days month for VLOOKUP'!A:B, 2)`

Je crée ensuite un tableau croisé dynamique avec `user_type` et `rideable_type` en Lignes, `day_id` et `day` en Colonnes, et `n_of_rides` en Valeurs.

*NB :* `day_id` *et* `day` *sont nécessaires si on veut pouvoir afficher les jours de la semaine dans l’ordre (*`day_id` *étant ajouté en premier, c’est lui qui prime pour l’ordre, et* `day` *permet d’afficher le nom du jour correspondant).*

<img src="img/EDA/Number of rides depending on user type.png" width=70%>

<img src="img/EDA/Number of rides for members.png" width=70%>
<img src="img/EDA/Number of rides for casual users.png" width=70%>


### <ul><li>Durée moyenne des trajets</ul></li>
Pour que la présentation des durées soit lisible (ce qui est difficile lorsqu’elle est exprimée en secondes), je crée dans Sheets un nouvelle colonne `mean_duration` calculée à partir de la durée moyenne en secondes selon le calcul `mean_duration = mean_duration_s / (3600*24)` puis je formate la colonne en tant que Durée (car pour Sheets, une durée unitaire = 1 jour).

Je crée ensuite un tableau croisé dynamique avec `user_type` et `rideable_type` en Lignes, `day_id` et `day` en Colonnes, et `mean_duration` en Valeurs.

*NB : Comme précédemment,* `day_id` *et* `day` *sont nécessaires si on veut pouvoir afficher les jours de la semaine dans l’ordre (*`day_id` *étant ajouté en premier, c’est lui qui prime pour l’ordre, et* `day` *permet d’afficher le nom du jour correspondant).*

<img src="img/EDA/Daily average of ride duration depending on user type.png" width=70%>

<img src="img/EDA/Daily average of ride duration for members.png" width=70%>
<img src="img/EDA/Daily average of ride duration for casual users.png" width=70%>


### <ul><li>Résultat des observations nombre et durée des trajets selon les jours de la semaine</ul></li>
On voit de très grandes différences d’utilisation selon le type d’utilisateurs :

- Les abonnés annuels effectuent davantage de trajets que les utilisateurs occasionnels, l’écart se situant principalement sur les jours de semaine (L-V).
- Les jours de plus grande utilisation des vélos sont différents selon le type d’utilisateurs : en semaine (L-V) pour les abonnés annuels, en fin de semaine pour les utilisateurs occasionnels (V-D). On peut donc supposer que les abonnés annuels utilisent davantage les vélos pour les trajets domicile-travail, alors que les occasionnels ont une utilisation plus tournée vers les loisirs.
- Les utilisateurs occasionnels utilisent davantage les vélos électriques que les vélos classiques, quel que soit le jour de la semaine. Mais les vélos cargo sont davantage utilisés durant la fin de semaine (V-D).
- Les abonnés annuels effectuent des trajets beaucoup plus courts (11:43) que les utilisateurs occasionnels (27:00). Cette différence provient du fait que :
   - La location des vélos cargo (exclusivement par les utilisateurs occasionnels) est beaucoup plus longue (45:47) que celle des autres types de vélos.
   - Les vélos classiques sont utilisés plus longuement par les utilisateurs occasionnels (22:10) que par les abonnés annuels (12:36).
   - Pour les vélos électriques, la tendance est la même mais dans une moindre mesure (13:03 pour les occasionnels et 10:50 pour les abonnés).
- Les trajets durent un peu plus longtemps durant les week-ends (22:33 en moyenne) que pendant les jours de semaine (20:13 en moyenne), en accord avec un usage “loisirs” où les usagers sont moins pressés ou bien parcourent de plus longues distances.


### <ul><li>Distance moyenne parcourue par trajet</ul></li>

NB : La distance "parcourue" représente ici la distance séparant les stations de départ et d’arrivée de chaque location. Elle s’exprime en mètres et a été calculée en SQL par la formule : 
```sql
  ROUND(AVG(
    ST_DISTANCE(
      ST_GEOGPOINT(start_lng, start_lat),
      ST_GEOGPOINT(end_lng, end_lat))
    ), 1) AS mean_distance_m
```
Je crée un Tableau croisé dynamique pour synthétiser les données, avec `user_type`, `month_id` et `month` en Lignes, `day_id` et `day` en Colonnes, et `mean_distance_m` en Valeurs.

*NB : Comme pour `day` et `day_id`, il faut utiliser `month_id` pour ordonner correctement `month`.*

<img src="img/EDA/Average distance by ride depending on user type.png" width=70%>

La distance moyenne des trajets est très proche entre les utilisateurs occasionnels (1982.0 m) et les abonnés annuels (2006.6 m). De même, aucune différence nette n’est visible entre les deux types d’utilisateurs selon les jours de la semaine (à 60 m près ce qui apparaît négligeable).

On peut faire les mêmes observations selon les mois de l'année :

<img src="img/EDA/Average distance by ride for members.png" width=70%>
<img src="img/EDA/Average distance by ride for casual users.png" width=70%>

En revanche, on constate qu’il existe une **saisonnalité** des distances parcourues : elles sont plus importantes au printemps et en été (> 2000 m) qu’en automne et en hiver (< 2000 m).


> NB : Pour pouvoir tracer les données annuelles par ordre chronologique, i.e. Aug 22 - Jul 23 (et non January-December qui ne correspond pas à l’enchaînement réel des données), je crée finalement une colonne `mmm yy` grâce à la formule :
>
> `=DATE(IF(C2<8,"2023","2022"),C2,1)`
>
> où `DATE` prend 3 arguments : année, mois, jour
>
>   `IF` conditionne l’année sur la base de la valeur de `month_id`
>
> *Pour rappel `IF(condition, value if True, value if False)`.*
>
>Enfin, je formate la colonne pour afficher le nom du mois sur trois lettres suivi de l’année sur deux chiffres.

Je modifie ensuite les paramètres du tableau croisé dynamique avec `user_type` et `mmm yy` en Lignes, `day_id` et `day` en Colonnes, et `mean_distance_m` en Valeurs.



## Tendances selon les mois de l'année
### <ul><li>En fonction du type d'utilisateur</ul></li>

<img src="img/EDA/Number of rides per month depending on user type.png" width=70%>
<img src="img/EDA/Average ride duration depending on user type.png" width=70%>
<img src="img/EDA/Average distance by ride per month depending on user type.png" width=70%>

On confirme bien que la saison modifie profondément l’utilisation des vélos :
- Le nombre de trajets mensuels chute considérablement pendant l’hiver (particulièrement rude à Chicago), d’un facteur 3.2 pour les membres et d’un facteur 9.0 pour les utilisateurs occasionnels.
- La durée des trajets est également raccourcie sur les mois froids, les utilisateurs occasionnels réduisant davantage la durée de leurs trajets que les abonnés annuels.
- La distance moyenne séparant les stations de départ et d’arrivée est également diminuée en hiver, mais sans distinction entre membres et utilisateurs occasionnels.

Est-ce que cet impact est différent selon le type de vélo emprunté ?


### <ul><li>En fonction du type de vélo</ul></li>
#### *Number of rides*
<img src="img/EDA/Number of rides per month for members.png" width=70%>
<img src="img/EDA/Number of rides per month for casual users.png" width=70%>

#### *Ride duration*
<img src="img/EDA/Average ride duration for members.png" width=70%>
<img src="img/EDA/Average ride duration for casual users.png" width=70%>

On ne constate pas de différence significative selon le type de vélo emprunté par rapport à la tendance générale


### <ul><li>Conclusion de cette étude selon les mois de l'année</ul></li>
Il est observé une saisonnalité de l’utilisation des vélos, qui est davantage creusée pendant les mois d’hiver pour les utilisateurs occasionnels. On peut donc supposer que :
- Le fait de détenir un abonnement annuel majore l’utilisation des vélos l’hiver (les membres sont peut-être sensibles à la rentabilisation de leur abonnement annuel, ou soucieux d’utiliser un moyen de déplacement écologique).
- L’utilisation du vélo pour les trajets domicile-travail (que l’on suspecte prédominante chez les abonnés annuels) est impactée dans une moindre mesure par la saison.



## Visualisation des données sur l'année, en fonction du jour de la semaine
On peut effectuer une cartographie des données grâce à une mise en forme conditionnelle des cellules des tableaux croisés dynamiques. On crée une mise en forme conditionnelle distincte pour chaque jeu de données (i.e. indicateur observé `n_of_rides` *vs* type d’utilisateur [`casual`, `member`]) :

*Average number of rides along the year, depending on the day of week*
<img src="img/EDA/heat_map.png">

On observe ainsi que durant les 3 mois d’hiver (décembre, janvier, février), les abonnés maintiennent une utilisation plus importante sur les jours de semaine (i.e. du lundi au vendredi) que durant les week-ends. A l’inverse, les utilisateurs occasionnels sont très peu actifs quels que soient les jours de la semaine.





## Analyse des trajets selon l’heure de la journée
L’objectif est de confirmer l’hypothèse que les abonnés annuels utilisent davantage les vélos pour leurs trajets domicile-travail.

Il faut donc extraire l’heure médiane (milieu entre l’heure de début et l’heure de fin) pour chaque trajet, puis regrouper les trajets par tranche horaire.

Dans BigQuery, cela se traduit par la requête suivante avec une **CTE** (table d'expression commune) :
```sql
WITH tmp AS (
  SELECT
    ride_id,
    MAKE_INTERVAL(hour => CAST(SPLIT(duration,':')[0] AS INT64),
      minute => CAST(SPLIT(duration,':')[1] AS INT64),
      second => CAST(SPLIT(duration,':')[2] AS INT64))
      AS duration_interval
  FROM
    cyclistic_clean_data.clean_data
)

SELECT
  user_type,
  rideable_type,
  EXTRACT(MONTH FROM start_time + duration_interval / 2) AS month,
  EXTRACT(DAYOFWEEK FROM start_time + duration_interval / 2) AS day,
  EXTRACT(HOUR FROM start_time + duration_interval / 2) AS hour,
  COUNT(clean_data.ride_id) AS n_of_rides,
  ROUND(AVG(UNIX_SECONDS(start_time + duration_interval) - UNIX_SECONDS(start_time)),3) AS mean_duration_s,
  ROUND(AVG(
    ST_DISTANCE(
      ST_GEOGPOINT(start_lng, start_lat),
      ST_GEOGPOINT(end_lng, end_lat))
    ), 1) AS mean_distance_m
FROM
  cyclistic_clean_data.clean_data as clean_data
JOIN
  tmp
ON
  tmp.ride_id = clean_data.ride_id
GROUP BY
  user_type,
  rideable_type,
  month,
  day,
  hour
ORDER BY
  user_type,
  rideable_type,
  month,
  day,
  hour
```

J’enregistre les résultats de la requête dans un fichier Sheets.

Je renomme les mois et jours en `month_id` et `day_id` puis crée les colonnes `mmm yy = DATE(IF(C2<8,"2023","2022"),C2,1))` et `day = VLOOKUP(D2,'days for VLOOKUP'!$A$2:$B$8,2))`.

J’explore maintenant les données grâce à un tableau croisé dynamique avec en Valeurs `n_of_rides` et en Colonnes `hour`.

Je m’aperçois qu’une séparation des jours à minuit n’est pas la plus pertinente pour une visualisation des données car on voit bien que l’utilisation du samedi par exemple se prolonge jusque tard dans la nuit. En visualisant toutes les données, on voit que 4h du matin serait une meilleure heure pour séparer les jours.

Par conséquent, je fais les modifications suivantes :
- création d’une colonne `day_id (corr) = IF(E2<4, IF(D2=1, 7, D2-1), D2)`
- création d’une colonne `day (corr) = VLOOKUP(H2,'days for VLOOKUP'!$A$2:$B$8, 2)`
- création d’une colonne `hour_string = TEXT(TIME(E2, 0, 0), "hh:mm")` pour l’affichage des valeurs sur l’axe *x*
- création d’une colonne `hour_index = IF(E2>=4, E2, E2+24)` pour que l’axe *x* soit dans l’ordre chronologique i.e. 24h de 04:00 à 03:00
- dans le tableau croisé dynamique, je mets en Colonnes `hour_index` (pour l’ordre chronologique) et `hour_string` (pour l’affichage sur l’axe *x*).

Je peux maintenant commencer l’analyse des données.


### <ul><li>Profil horaire pour les utilisateurs en fonction des jours de la semaine</ul></li>
NB : Il s’agit d’une somme du nombre de trajets sur l’année entière.

<img src="img/EDA/Hourly number of rides for members.png" width=70%>
<img src="img/EDA/Hourly number of rides for casual users.png" width=70%>

En regardant la somme des trajets sur l’année, on confirme très nettement l’usage des **membres** pour les trajets domicile-travail (pics à 8h et 16-17h) du lundi au vendredi, ainsi que pour des trajets loisirs les vendredis soirs (à partir de 22h le profil est différent des autres jours de la semaine) et durant les week-ends.

Pour les **utilisateurs occasionnels**, on voit également un profil avec des pics autour de 8h et 17-18h, mais dans une moindre mesure (le pic du matin est très faible). Il y a en revanche davantage d’utilisation durant les week-ends (vendredis soirs, samedis et dimanches).


### <ul><li>Evaluation de l'impact des saisons</ul></li>
On considère les mois où l’utilisation est la plus faible : décembre, janvier et février (hiver) et les mois où elle est la plus élevée : juin, juillet, août (été). Les données sont donc sommées sur ces mois.

#### *Hiver*
<img src="img/EDA/Hourly number of rides during Winter for members.png" width=70%>
<img src="img/EDA/Hourly number of rides during Winter for casual users.png" width=70%>

Durant l’hiver, les utilisateurs occasionnels réduisent fortement l’utilisation pour les loisirs.

On remarque même une situation encore plus marquée sur le mois de décembre : 

<img src="img/EDA/Hourly number of rides in December for members.png" width=70%>
<img src="img/EDA/Hourly number of rides in December for casual users.png" width=70%>


#### *Eté*
NB : Je ne considère pas le mois d’août pour faire la somme car il s’agit d’août 2022, il n’y aurait donc pas de continuité temporelle entre les données avec les mois de juin et juillet 2023.

<img src="img/EDA/Hourly number of rides in Summer for members.png" width=70%>
<img src="img/EDA/Hourly number of rides in Summer for casual users.png" width=70%>

On observe les mêmes différences selon le type d’utilisateurs. L’utilisation pour les loisirs est plus importante qu’en hiver.


### <ul><li>Distance moyenne parcourue</ul></li>
Pour finir, on regarde si l’on voit des profils horaires significatifs en ce qui concerne les distances parcourues par trajet (i.e. distances séparant les stations de départ et d’arrivée) :

<img src="img/EDA/Hourly average ride distance for members.png" width=70%>
<img src="img/EDA/Hourly average ride distance for casual users.png" width=70%>

En semaine, pour les abonnés annuels, on constate que les trajets effectués en milieu de journée (9h-16h) sont plus courts (ca. 1700-2000 m) que ceux effectués durant les pics d’utilisation liés aux trajets domicile-travail (ca. 2000-2200 m).

Durant les week-ends, les trajets sont plus longs en journée, en comparaison avec les jours de semaine, quel que soit le type d’usager.

Il serait intéressant de regarder la localisation des stations de départ et d’arrivée dans des cas précis :
- en semaine à 8h, 12h, 17h
- le samedi à 15h

et essayer de corréler ces données avec les données géographiques et économiques : localisation de zones de bureaux, de zones commerciales/de restauration, de zones résidentielles et prendre en compte les plateformes multimodales (train, métro).





## Prérequis pour l’analyse géographique des trajets
La difficulté réside en ce que les coordonnées géographiques des stations ne sont pas toutes données avec la même précision. Je vais chercher les données des stations sur le portail de données ouvertes de la ville de Chicago
https://data.cityofchicago.org/Transportation/Divvy-Bicycle-Stations/bbyy-e7gq/data

Il existe d’après ces données officielles 1419 stations distinctes (noms distincts et ID distincts).

Dans le jeu de données nettoyées, le nombre de stations distinctes est supérieur à ce qui apparaît dans la table des stations !

```sql
WITH clean_data AS
  (
  SELECT
    *
  FROM
    cyclistic_merge_data.full_data
  WHERE
    end_lat > 0
    AND end_lng < 0
    AND ended_at > started_at
    AND ended_at - started_at <= MAKE_INTERVAL(0, 0, 1, 1, 0, 0) # 1 day + 1 hour
  )

SELECT
  COUNT(DISTINCT(clean_data.start_station_name)) AS start_stations,
  COUNT(DISTINCT(clean_data.end_station_name)) AS end_stations
FROM
  clean_data
```
![request result](img/EDA/stations_from_data.png)


*NB : dans la requête, je suis reparti de la table* `cyclistic_merge_data.full_data` *car dans la table* `cyclistic_clean_data.clean_data` *j’ai supprimé les informations de noms ou ID de stations.*

Attention, il y a également de très nombreux enregistrements avec des valeurs _null_ pour les stations de départ ou d’arrivée (`*_station_name` ou `*_station_id`) :

```sql
WITH clean_data AS
  (
  SELECT
    *
  FROM
    cyclistic_merge_data.full_data
  WHERE
    end_lat > 0
    AND end_lng < 0
    AND ended_at > started_at
    AND ended_at - started_at <= MAKE_INTERVAL(0, 0, 1, 1, 0, 0) # 1 day + 1 hour
  )

SELECT
  COUNTIF(clean_data.start_station_name IS NULL) AS null_start_name,
  COUNTIF(clean_data.start_station_id IS NULL) AS null_start_id,
  COUNTIF(clean_data.end_station_name IS NULL) AS null_end_name,
  COUNTIF(clean_data.end_station_id IS NULL) AS null_end_id,
FROM
  clean_data
```
![request result](img/EDA/stations_null.png)


Cela représente une fraction très importante des données nettoyées : **1 376 546 entrées** avec une valeur _null_ dans `start_station_name` ou `end_station_name`, soit **24% des données** !

C’est particulièrement surprenant car dans l’étape de nettoyage des données, j’ai supprimé les données pour lesquelles les coordonnées géographiques sont nulles ou égales à zéro (et cela ne représentait que 6112 enregistrements). Il y a donc un très grand nombre d’enregistrements pour lesquels des coordonnées géographiques sont renseignées mais pas les noms de station. 

> _Dans une situation réelle, il faudrait trouver la raison pour laquelle on a ces enregistrements : s’agit-il d’un bug du système qui ne renseigne pas les stations correctement ? est-ce que cela correspond à des vélos qui seraient retrouvés en-dehors des stations ??_


### <ul><li>Identification des stations de départ et arrivée</ul></li>

J’essaie sur les enregistrements restants d’identifier correctement les stations de départ et d’arrivée. Problème : les identifiants des stations dans la table des trajets ne correspondent pas aux identifiants officiels des stations, il est donc impossible de les utiliser. J’essaye donc de me baser sur les noms des stations et j’explore les données pour voir quels sont les problèmes dans les noms de stations par rapport aux noms officiels. Si je ne fais pas cette étape, je ne pourrai pas arriver à une visualisation de la géographie des trajets car il me sera impossible de faire une agrégation par station.

Je crée une table `stations.stations_summary` avec les noms de stations tels qu’ils existent en tant que `start_station_name` des données originales et la correspondance `ID` issue de la liste officielle des stations du portail open data de la ville de Chicago. Pour créer cette table, j’ai inspecté minutieusement les noms des stations et corrigé ce qui pouvait l’être. Lorsque j’importe la table dans BigQuery, je rends les champs `station_name` et `station_ID` requis ce qui supprime d’office les éventuelles valeurs nulles.

En complément, j’importe la table de la liste officielle des stations trouvée sur le portail open data de la ville de Chicago : table `stations.stations_list`.

J’essaie de faire un `JOIN` sur les **stations de départ** pour vérification :

```sql
WITH stations_coord AS
  (SELECT
    list.ID,
    list.Station_Name,
    list.Latitude,
    list.Longitude
  FROM
    stations.stations_list AS list
  JOIN
    stations.stations_summary AS summary
  ON
    list.ID = summary.station_ID
  )

SELECT
  merge_data.ride_id,
  merge_data.start_station_name,
  merge_data.start_lat,
  merge_data.start_lng,
  stations_coord.Station_Name,
  stations_coord.Latitude,
  stations_coord.Longitude
FROM
  cyclistic_merge_data.full_data AS merge_data
JOIN
  stations_coord
ON
  stations_coord.Station_Name = merge_data.start_station_name
WHERE
  end_lat > 0
  AND end_lng < 0
  AND ended_at > started_at
  AND ended_at - started_at <= MAKE_INTERVAL(0, 0, 1, 1, 0, 0) # 1 day + 1 hour
```

→ 4,892,713 résultats



La même chose sur les **stations d’arrivée** :

```sql
WITH stations_coord AS
  (SELECT
    list.ID,
    list.Station_Name,
    list.Latitude,
    list.Longitude
  FROM
    stations.stations_list AS list
  JOIN
    stations.stations_summary AS summary
  ON
    list.ID = summary.station_ID
  )

SELECT
  merge_data.ride_id,
  merge_data.end_station_name,
  merge_data.end_lat,
  merge_data.end_lng,
  stations_coord.Station_Name,
  stations_coord.Latitude,
  stations_coord.Longitude
FROM
  cyclistic_merge_data.full_data AS merge_data
JOIN
  stations_coord
ON
  stations_coord.Station_Name = merge_data.end_station_name
WHERE
  end_lat > 0
  AND end_lng < 0
  AND ended_at > started_at
  AND ended_at - started_at <= MAKE_INTERVAL(0, 0, 1, 1, 0, 0) # 1 day + 1 hour
```

→ 4,843,419 résultats


Il me faut à présent décider de comment extraire les données pour répondre à la question de l’analyse géographique des trajets.

Il faut conserver des données originales les noms des stations de départ et d’arrivée mais pas les coordonnées géographiques, que je récupérerai à l’aide de la table `stations.stations_list`.
Je pourrai donc faire une agrégation sur la base des noms de stations.


Pour simplifier les requêtes à venir, je crée une nouvelle table `stations.stations_valid` avec la correspondance entre les noms originaux et les noms officiels, et avec les coordonnées géographiques des stations :

```sql
SELECT
  summary.station_name AS original_name,
  list.Station_Name AS official_name,
  list.Latitude AS latitude,
  list.Longitude AS longitude
FROM
  stations.stations_summary AS summary
JOIN
  stations.stations_list AS list
ON
  summary.station_ID = list.ID
```

La table créée contient 1587 entrées : 1587 valeurs distinctes dans le champ `original_name` et 1369 dans le champ `official_name`.


### <ul><li>Aggrégation des données correspondant aux situations choisies</ul></li>

J’ai normalement tous les éléments en main pour effectuer l’analyse géographique des trajets dans les cas de figure considérés : en semaine à 8h, 12h, 17h et le samedi à 15h.

Je peux extraire les données agrégées pour ces 4 situations via la requête suivante :

```sql
WITH tmp AS (
  SELECT
    clean.ride_id,
    MAKE_INTERVAL(hour => CAST(SPLIT(duration,':')[0] AS INT64),
      minute => CAST(SPLIT(duration,':')[1] AS INT64),
      second => CAST(SPLIT(duration,':')[2] AS INT64))
      AS duration_interval,
    st_start.official_name AS start_station,
    st_end.official_name AS end_station
  FROM
    cyclistic_clean_data.clean_data AS clean
  JOIN cyclistic_merge_data.full_data AS merged ON clean.ride_id = merged.ride_id
  JOIN stations.stations_valid AS st_start ON st_start.original_name = merged.start_station_name
  JOIN stations.stations_valid AS st_end ON st_end.original_name = merged.end_station_name
)

SELECT
  user_type,
  rideable_type,
  EXTRACT(MONTH FROM start_time + duration_interval / 2) AS month,
  EXTRACT(DAYOFWEEK FROM start_time + duration_interval / 2) AS day,
  EXTRACT(HOUR FROM start_time + duration_interval / 2) AS hour,
  COUNT(clean_data.ride_id) AS n_of_rides,
  ROUND(AVG(UNIX_SECONDS(start_time + duration_interval) - UNIX_SECONDS(start_time)),3) AS mean_duration_s,
  ROUND(AVG(
    ST_DISTANCE(
      ST_GEOGPOINT(start_lng, start_lat),
      ST_GEOGPOINT(end_lng, end_lat))
    ), 1) AS mean_distance_m,
  start_station,
  end_station
FROM
  cyclistic_clean_data.clean_data as clean_data
JOIN
  tmp
ON
  tmp.ride_id = clean_data.ride_id
WHERE
  (EXTRACT(HOUR FROM start_time + duration_interval / 2) IN (8,12,17)
  AND EXTRACT(DAYOFWEEK FROM start_time + duration_interval / 2) IN (2,3,4,5,6))
  OR
  (EXTRACT(HOUR FROM start_time + duration_interval / 2) = 15
  AND EXTRACT(DAYOFWEEK FROM start_time + duration_interval / 2) = 7)
GROUP BY
  user_type,
  rideable_type,
  month,
  day,
  hour,
  start_station,
  end_station
ORDER BY
  user_type,
  rideable_type,
  month,
  day,
  hour,
  start_station,
  end_station
```

Cette requête retourne 648 798 entrées.




Pour commencer l’exploration des données, je scinde les données pour chaque situation. J’enregistre au préalable le résultat comprenant toutes les situations dans une nouvelle table `cyclistic_specific_data.all_situations`.




#### _Samedis à 15h :_

```sql
SELECT
  *
FROM
  cyclistic_specific_data.all_situations
WHERE
  hour = 15
```

41 665 entrées (pour un total de 52 564 trajets).




#### _Jours de semaine à 8h :_

```sql
SELECT
  *
FROM
  cyclistic_specific_data.all_situations
WHERE
  hour = 8
```

170 510 entrées (pour 207 929 trajets).




#### _Jours de semaine à 12h :_

```sql
SELECT
  *
FROM
  cyclistic_specific_data.all_situations
WHERE
  hour = 12
```

132 674 entrées (total de 154 684 trajets).




#### _Jours de semaine à 17h :_

```sql
SELECT
  *
FROM
  cyclistic_specific_data.all_situations
WHERE
  hour = 17
```

303 949 entrées (total 359 690 trajets)


