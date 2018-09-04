# Neo4j - Projet réseaux d'intéraction

Le but de ce projet est de réaliser une base de données orientée graphe représentant le métro parisien. Pour effectuer ce travail, nous allons nous inspirer de ce qui a été fait avec [le métro de Londre](https://tbgraph.wordpress.com/2017/08/31/neo4j-london-tube-system-analysis/).

## Prérequis
- [Neo4j](https://neo4j.com/download/)
- Le plugin [Graph algorithms](https://github.com/neo4j-contrib/neo4j-graph-algorithms/)
- Le plugin [APOC](http://github.com/neo4j-contrib/neo4j-apoc-procedures/releases/3.4.0.1) (*Nécessaire pour la conversion en format date et la future upgrade temps-réel*)
- Les informations sur l'[ensemble des lignes](http://dataratp.download.opendatasoft.com/RATP_GTFS_LINES.zip) du réseau RATP

Le dossier RATP_GTFS_LINES contient un répertoire par ligne de transport. Nous allons ici présenter les commandes pour la ligne 1 du métro, pour obtenir le graphe de l'ensemble du réseau, il est nécessaire de répéter l'opération pour chaque ligne.

## Étape 1 - Ajout des stations
On utilise le fichier `stops.csv` qui contient notamment les noms des stations ainsi que l'identifiant qui va nous aider à céer les connexions entre-elles : `stop_id`.

Chaque ligne du document correspond à un quai de métro, il y a deux quais sur une voie, un pour chaque sens de circulation. Un noeud dans notre graphe est donc un nom et un ensemble d'identifiant de quai. Pour créer les noeuds, on utilise le script suivant :
```php
LOAD CSV WITH HEADERS FROM "file:///RATP_GTFS_METRO_1/stops.csv" as row
MERGE (s:Station{name:row.stop_name})
ON CREATE SET s.stop_M1_1 = toInteger(row.stop_id),
    s.name = row.stop_name,
    s.address = row.stop_desc,
    s.coordonnee = row.coord,
    s.latitude = row.stop_lat,
    s.longitude = row.stop_lon,
    s.code = row.code_INSEE,
    s.departement = row.departement
ON MATCH SET s.stop_M1_2 = toInteger(row.stop_id)
```
*Note* : Nous avons ajouté 25 stations, il serai interessant de pourvoir stocker cette information par exemple dans un noeud spécial `MATCH(s:Station) WHERE s.stop_M1_1 > 0 MERGE (n:Data) SET n.nbStationM1 = count(s)` (*Ne fonctionne pas*)

*Edit* : Possible avec APOC ? A tester =>
```php
MATCH (user:User)
WHERE user.Id = 1234
WITH user, size((user)-[:IS_FRIENDS_WITH]->(:Friend)) as numberOfFriends
CALL apoc.map.setKey(user, 'numberOfFriends', numberOfFriends) YIELD value AS userWithFriends
RETURN userWithFriends
```

La commande `MATCH (n) RETURN n` permet de visualiser l'état actuel du graphe :

![Graph with all the stations](https://github.com/CamilleSimon/neo4j-project/blob/master/graph.png)

## Étape 2 - Ajout des connexions entre les stations

Le fichier `stop_times.txt` contient l'ensemble des trajets effectués sur la ligne de métro.

Un déplacement entre deux stations corresponds au passage d'une ligne du fichier à la suivante.
Afin de simplifier le traitement, nous allons dans un premier temps enregistrer tous les arrêts à quai puis construire les déplacements entre deux quais.

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
MATCH (t2:Travel{trip_id:t1.trip_id}) WHERE t2.stop_sequence = t1.stop_sequence + 1
MATCH (s1:Station) WHERE s1.stop_M1_1 = t1.stop_id
MATCH (s2:Station) WHERE s2.stop_M1_1 = t2.stop_id
MERGE (s1)-[m:M1]->(s2) 
```
```php
MATCH (t1:Travel)
MATCH (t2:Travel{trip_id:t1.trip_id}) WHERE t2.stop_sequence = t1.stop_sequence + 1
MATCH (s1:Station) WHERE s1.stop_M1_1 = t1.stop_id
MATCH (s2:Station) WHERE s2.stop_M1_2 = t2.stop_id
MERGE (s1)-[m:M1]->(s2) 
```
```php
MATCH (t1:Travel)
MATCH (t2:Travel{trip_id:t1.trip_id}) WHERE t2.stop_sequence = t1.stop_sequence + 1
MATCH (s1:Station) WHERE s1.stop_M1_2 = t1.stop_id
MATCH (s2:Station) WHERE s2.stop_M1_1 = t2.stop_id
MERGE (s1)-[m:M1]->(s2) 
```
```php
MATCH (t1:Travel)
MATCH (t2:Travel{trip_id:t1.trip_id}) WHERE t2.stop_sequence = t1.stop_sequence + 1
MATCH (s1:Station) WHERE s1.stop_M1_2 = t1.stop_id
MATCH (s2:Station) WHERE s2.stop_M1_2 = t2.stop_id
MERGE (s1)-[m:M1]->(s2) 
```

*Remarque 1* : Impossible d'utiliser une version factorisée du code !
```php
MATCH (t1:Travel)
MATCH (t2:Travel{trip_id:t1.trip_id}) WHERE t2.stop_sequence = t1.stop_sequence + 1
MATCH (s1:Station) WHERE s1.stop_M1_1 = t1.stop_id OR s1.stop_M1_2 = t1.stop_id
MATCH (s2:Station) WHERE s2.stop_M1_1 = t2.stop_id OR s2.stop_M1_2 = t2.stop_id
MERGE (s1)-[m:M1]->(s2) 
```

[TODO : calcul de complexité en nombre d'opération]

*Remarque 2* : 
Il est possible de connaitre le temps moyen de parcours entre deux stations en ajoutant le code suivant au script de création des trajets : 
```php
ON CREATE SET m.nb = 1, m.time = toFloat(t2.arrival_time-t1.departure_time)
ON MATCH SET m.nb = m.nb + 1, m.time = m.time + t2.arrival_time-t1.departure_time
```
Afin d'avoir la moyenne en minute de temps de trajet, on ajoute l'attribut `avg_time` sur chaque arrête :
```php
MATCH ()-[m:M1]-() SET m.avg_time = m.time / m.nb RETURN m
```

### Graphe de la ligne 1 du métro
La commande `MATCH (n)-[:M1]-(m) RETURN n,m` nous permet de visualiser la ligne 1 du métro :

![Graph of M1](https://github.com/CamilleSimon/neo4j-project/blob/master/graph-metro1.png)

*Remarque* : 

Sur certain trajet, le métro ne fait pas toutes les stations, on a alors la création de "triangulaire". Doit-on tout représenter ou seulement le chemin le plus long en nombre d'arrête ?

Ex : A-->C, A-->B, B-->C. Eliminer A-->B et B-->C ?

Si l'on veut, dans l'avenir, calculer le temps de trajet entre deux stations, il est nécessaire de garder tous les trajets ainsi que leurs horaires de passages mais cela rend le graphe moins lisible. 

## Idées d'amélioration
- Ajouter les heures de départ et d'arrivée afin de prendre en compte les horaires pour les correspondances
- Que faire des triangulaires ? 

## Références
- L'exemple avec le métro londonnien : [https://tbgraph.wordpress.com/2017/08/31/neo4j-london-tube-system-analysis/](https://tbgraph.wordpress.com/2017/08/31/neo4j-london-tube-system-analysis/)
- Calcul d'itinéraire depuis les données RATP en scala : [http://www.dericbourg.net/2015/12/10/calcul-ditineraire-a-partir-des-donnees-ratp/](http://www.dericbourg.net/2015/12/10/calcul-ditineraire-a-partir-des-donnees-ratp/)
- Cours de M. Olivier [http://litis.univ-lehavre.fr/~dolivier/PagePerso/uploads/Enseignement/ReseauxInteraction/Neo4J.pdf](http://litis.univ-lehavre.fr/~dolivier/PagePerso/uploads/Enseignement/ReseauxInteraction/Neo4J.pdf)




