# Sync

Sync est un module développé en SQL au dessus de Postgres 9.5 minimum pour faciliter la synchronisation de bases de données terrains (multiples) avec une base serveur. Les bases de données partagent le même schéma. 
Il ne s'agit pas de réplication de données comme fait SLONY (http://www.slony.info/), mais de fusion de données multi-sources (ici les bases de données terrain). 
Chaque utilisateur doit se connecter avec un login différent suivant la base terrain : par exemple, toto utilise l'utilisateur 'user1' lorsqu'il utilise la base de données 'terrain1', et l'utilisateur 'user2' lorsqu'il utilise la base de données 'terrain2'.

## Comment ça marche

Sync historise dans une table **sauv_data** via un trigger toutes les modifications faites sur les bases terrain. 

Il garde la modifications et les métadonnées sur cette modification : 
- le login utilisateur (integrateur) qui sera un traceur *et* de la machine *et* de l'utilisateur postgresql (![warning](./warning30x30.png) ce qui peut être source de bugs)
- le timestamp (ts) avec time zone
- le nom du schema (schema_bd)
- le nom de la table (tbl)
- l'action (update, delete, insert) : ACTION1
- nom de la PK sur la table modifiée: pk
- il encapsule le tuple qui a été modifié en json : sauv
- il renseigne un attribut replay : FALSE avant analyse des éventuels conflits, TRUE ensuite
- il renseigne un attribut no_replay :   
	- reste à 1 si c'est des mises à jours intervenants sur un objet édité plusieurs fois par le même utilisateur sur la même base terrain. On ne veut traiter que les dernières mises à jour d'un utilisateur.
    - passe à NULL si c'est une mise à jour (la dernière) qui doit être intégrée.
	- passe à 2 quand il y a un conflit, c'est à dire update multi-utilisateur sur le même objet (repéré par le triplet <schema_bd, tbl, pk>.

## En pratique

Initialiser l'environnement : créer la table sauv_data, les triggers et les vues afférentes. 

1. Exécuter function_sauv_data.sql sur votre base de données terrain
2. Exécuter  sync_server.sql qui créent 3 vues et 3 fonctions :  
    - **sync.replay** contient les lignes qui seront finalement intégrées dans la base de données serveur
    - **sync.no_replay** contient les lignes qui ne seront pas jouées du fait de l'édition multiple d'une même entité par le même utilisateur (contrôle sur integrateur).
    - **sync.conflict** contient les lignes qui ne seront pas jouées du fait de l'édition multiple d'une même entité par différents utilisateurs (contrôle sur integrateur) et qui présentent donc un conflit. La vue conflit peut être éditée par l'utilisateur pour résoudre les conflits en passant supprime_data à une valeur true pour toutes les valeurs qu'on ne souhaite pas garder pour la fusion. 
3. Exécuter les function sync.no_replay() et sync.replay() 

En production sur votre serveur : 

1. Supprimer les entrées de la table sauv_data (plus tard, ce sera fait automatiquement)
``` 
delete from sauv_data;
```

2. Récupérer les mises à jour des bases terrain par un lien (db_link) vers ces bases de données : elles sont copiées dans sauv_data.
Exemple pour deux bases terrain 'terrain1' et 'terrain2' ayant chacune un utilisateur différent 'user1' et 'user2'
``` 
-- Première BDD terrain1
SELECT
dblink_connect('linkterrain1','host=127.0.0.1 port=5432
 user=user1
 password=postgres
 dbname=terrain1');
-- "OK"

INSERT INTO sauv_data 
SELECT * from dblink('linkterrain1', 'select * from sauv_data;') as t(integrateur text, ts timestamp with time zone, schema_bd text, tbl text, action1 text, sauv json, pk text);
-- Query returned successfully: one row affected, 11 msec execution time.

SELECT dblink_disconnect('linkterrain1');


-- Deuxième BDD terrain2

SELECT
dblink_connect('linkterrain2','host=127.0.0.1 port=5432
 user=user2
 password=postgres
 dbname=terrain2');
-- "OK"

INSERT INTO sauv_data 
SELECT * from dblink('linkterrain2', 'select * from sauv_data;') as t(integrateur text, ts timestamp with time zone, schema_bd text, tbl text, action1 text, sauv json, pk text);
-- Query returned successfully: one row affected, 11 msec execution time.

SELECT dblink_disconnect('linkterrain2');

```


3. Jouer no_replay() 
``` 
select no_replay();
```
4. Vérifier et résoudre à la main les conflits : ouvrir la vue conflict et modifier no_replay à 2 par exemple pour les valeurs à ne pas garder (![warning](./warning30x30.png)  c'est bien cela ?)
5. Jouer replay()
``` 
select replay();
```



##Licence

Logiciel diffusé sous licence open-source AGPL

