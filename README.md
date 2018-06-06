# Neo4j-project

Le but de ce projet est de réaliser un travail similaire à [celui-ci](https://tbgraph.wordpress.com/2017/08/31/neo4j-london-tube-system-analysis/) mais portant sur le métro parisien.

## Prérequis
- [Neo4j](https://neo4j.com/download/)
- Le plugin [Graph algorithms](https://github.com/neo4j-contrib/neo4j-graph-algorithms/)
- Le plugin [APOC](http://github.com/neo4j-contrib/neo4j-apoc-procedures/releases/3.4.0.1)
- Les fichiers RATP suivants :
  - Les informations sur l'[ensemble des lignes](http://dataratp.download.opendatasoft.com/RATP_GTFS_LINES.zip) du réseau
  - Les [positions des stations](https://data.ratp.fr/explore/dataset/positions-geographiques-des-stations-du-reseau-ratp/download/?format=csv&timezone=Europe/Berlin&use_labels_for_header=true)

## Étape 1 - Ajout des stations
On utilise le fichier `positions-geographiques-des-stations-du-reseau-ratp.csv` qui contient notamment les noms des stations ainsi que l'identifiant qui va nous aider à céer les connexion entre les elles : `stop_id`.

Chaque ligne du document va correspondre à un noeud dans notre graphe. Pour créer les noeuds, on utilise le script suivant :
```php
LOAD CSV WITH HEADERS FROM "file:///positions-geographiques-des-stations-du-reseau-ratp.csv" as row
CREATE (s:Station)
SET s.stop_id = toInteger(row.stop_id),
    s.name = row.stop_name,
    s.address = row.stop_desc,
    s.coordonnee = row.coord,
    s.latitude = row.stop_lat,
    s.longitude = row.stop_lon,
    s.code = row.code_INSEE,
    s.departement = row.departement
```

La commande `MATCH (n) RETURN n` permet de visualiser l'état actuel du graphe :


![Graph with all the stations](https://github.com/CamilleSimon/neo4j-project/blob/master/graph.png)

## Étape 2 - Ajout des connexions entre les stations
Les exemples suivants sont appliqués sur la ligne 1 du métro, pour créer le plan complet du métro de Paris, il faut répéter l'opération pour chacune des lignes.

Le dossier `RATP_GTFS_LINES` contient l'ensemble des informations disponibles sur chaque ligne. C'est à partir de ces fichiers que nous allons construire nos connexions entre les stations.

### Liaisons entre les quais des stations
Les `stop_id` sont les quais où s'arrête le métro. Il y au moins deux `stop_id` pour une même station, un pour chaque sens de circulation.
Commençons par créer les connexions entre les quais.
```php
MATCH(s1:Station)
MATCH(s2:Station) WHERE s1.name=s2.name AND s1<>s2
MERGE (s1)-[:MEMESTATION]->(s2)
MERGE (s2)-[:MEMESTATION]->(s1)
```

### Liaisons à pied entre les stations
Il existe des couloirs permettant de se déplacer d'une station à l'autre, cette information est dans le fichier `transfers.txt`.
```php
USING PERIODIC COMMIT 800
LOAD CSV WITH HEADERS FROM "file:///RATP_GTFS_METRO_1/transfers.csv" as row
MATCH (s1:Station{stop_id:toInteger(row.from_stop_id)})
MATCH (s2:Station{stop_id:toInteger(row.to_stop_id)})
MERGE (s1)-[:PIED{time:row.min_transfer_time}]->(s2)
MERGE (s1)<-[:PIED{time:row.min_transfer_time}]-(s2)
```

### Liaisons entre les stations d'une même ligne
Le fichier `stop_times.txt` est celui qui va nous interesser, chaque ligne du fichier est constituée de :
trip_id,arrival_time,departure_time,stop_id,stop_sequence,stop_headsign,shape_dist_traveled

Un déplacement entre deux stations corresponds au passage d'une ligne du fichier à la suivante.
Afin de simplifier le traitement, nous allons dand un premier temps enregistrer tous les arrêts à quai puis construire les déplacmeent entre deux quais.

Extraction des informations sur l'arrivée et le départ des trains à quai :
```php
USING PERIODIC COMMIT 800
LOAD CSV WITH HEADERS FROM "file:///RATP_GTFS_METRO_1/stop_times.csv" as row
CREATE (t:Travel)
SET t.trip_id = toInteger(row.trip_id),
              t.arrival_time = toInteger(apoc.date.parse(row.departure_time,'m','HH:mm:ss')),
              t.departure_time = toInteger(apoc.date.parse(row.departure_time,'m','HH:mm:ss')),
              t.stop_id = toInteger(row.stop_id),
              t.stop_sequence = toInteger(row.stop_sequence)
```

Nous pouvons maintenant construire le déplacement entre deux arrêts :
```php
MATCH (t1:Travel)
MATCH (t2:Travel) WHERE t1.trip_id = t2.trip_id AND t2.stop_sequence = t1.stop_sequence + 1
MATCH (s1:Station) WHERE s1.stop_id = t1.stop_id
MATCH (s2:Station) WHERE s2.stop_id = t2.stop_id
MERGE (s1)-[m:M1]->(s2) 
ON CREATE SET m.nb = 1, m.time = toFloat(t2.arrival_time-t1.departure_time)
ON MATCH SET m.nb = m.nb + 1, m.time = m.time + t2.arrival_time-t1.departure_time
```

Afin d'avoir la moyenne en minute de temps de trajet, on ajoute l'attribut `avg_time` sur chaque arrête :
```php
MATCH ()-[m:M1]-() SET m.avg_time = m.time / m.nb RETURN m
```

*Remarque* : 

Sur certain trajet, le métro ne fait pas toutes les stations. Doit-on tout représenter ou seulement le chemin le plus long ?

Ex : A-->C, A-->B, B-->C. Eliminer A-->B et B-->C ?
Si on veut dans l'avenir calculer le temps de trajet entre deux stations, il est nécessaire de garder tous les trajets ainsi que leurs horaires de passages.

//Une fois tous les déplacements entre les stations ajoutés au graphe, on peut supprimer les informations sur les arrivées et départs des trains :

*Suppression des connexions en cas de fausse manipulation*
```php
MATCH ()-[r]-()
DELETE r
```

## Pour aller plus loin
- Ajouter les heures de départ et d'arrivée afin de prendre en compte les temps des correspondances
- Fusionner les `stop_id` dans une liste. Chaque station aurai un nom unique et une liste de `stop_id`



