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
On utilise le fichier `positions-geographiques-des-stations-du-reseau-ratp.csv` qui contient notamment les noms des stations ainsi que l'identifiant qui va nous aider à céer les connexion entre les stations : `stop_id`.

Chaque ligne du document va correspondre à un noeud de notre graphe. Pour créer les noeuds, on utilise le script suivant :
On peut maintenant passer à l'ajout des stations :
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
Il n'y a pas sur le site de la RATP de fichiers permettant directement d'ajouter les liaisons entre les stations. Afin d'avoir quelque chose de similaire au tutoriel du métro de Londre, nous devons créer les fichiers nécessaires.

### Création des fichiers de liaison entre les stations
Le dossier `RATP_GTFS_LINES` contient l'ensemble des informations disponibles sur chaque ligne. C'est à partir de ces fichiers que nous allons construire un document CSV avec la station de départ, la station d'arrivée, la ligne concernée ainsi que le temps moyen du trajet.


