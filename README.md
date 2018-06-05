# Neo4j-project

Le but de ce projet est de réaliser un travail similaire à [celui-ci](https://tbgraph.wordpress.com/2017/08/31/neo4j-london-tube-system-analysis/) mais portant sur le métro parisien.

## Prérequis
- [Neo4j](https://neo4j.com/download/)
- Le plugin [Graph algorithms](https://github.com/neo4j-contrib/neo4j-graph-algorithms/)
- Les fichiers RATP suivants :
  - Les informations sur l'[ensemble des lignes](http://dataratp.download.opendatasoft.com/RATP_GTFS_LINES.zip) du réseau
  - Les [positions des stations](https://data.ratp.fr/explore/dataset/positions-geographiques-des-stations-du-reseau-ratp/download/?format=csv&timezone=Europe/Berlin&use_labels_for_header=true)

 ## Etape 1 - Ajout des stations
Pour s'assurer que chaque station est unique, on ajout la contrainte suivante :
```
    CREATE CONSTRAINT ON (s:Station) ASSERT s.id is unique;
```
Afin de facilité l'indexation des stations, l'id sera attribuer dans l'ordre alphabétique du nom des stations.
```
    CREATE INDEX ON :Station(name);
```


