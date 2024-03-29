

**********************************************************************************************************************************

## Rappel pour retrouver les environnements éventuellement précédemment instanciés dans Gitpod : [ https://gitpod.io/workspaces ](https://gitpod.io/workspaces)

**********************************************************************************************************************************

## Pour lancer notre TP  (en ligne via un simple navigateur web et un compte Gitlab/Github basique)  :

[![Open in Gitpod](https://gitpod.io/button/open-in-gitpod.svg)](https://gitpod.io/#https://github.com/crystalloide/Flink-Kafka)

**********************************************************************************************************************************

# Démos : contexte : 

Pour aller plus loin et retrouver les conférences auxquelles Martijn Visser a participé et fait ces démonstrations :

* [Flink Forward talk 'Only SQL: Empower data analysts end-to-end with Flink SQL'](https://www.flink-forward.org/global-2021/conference-program#only-sql--empower-data-analysts-end-to-end-with-flink-sql) - [Recording](https://www.youtube.com/watch?v=KvaDe7QCwBQ) - [Demo version](https://github.com/MartijnVisser/flink-only-sql/releases/tag/v1.0) 
* [Uptime talk 'Only SQL'](https://uptime.aiven.io/session/325604) - [Demo version](https://github.com/MartijnVisser/flink-only-sql/releases/tag/v2.0)


## Docker

Nous utiliserons Docker Compose pour démarrer tous les services nécessaires à la démo. Cela lancera donc les services suivants :

* Apache Flink 1.15.2, accessible via http://localhost:8081
* Apache Flink SQL Client 1.15.2
* Apache Kafka (including Zookeeper) 7.2.1, accessible via broker:29092
* Confluent Schema Registry 7.2.1, accessible via http://localhost:8091 (or http://schema-registry:8091 via Docker networking) 
* Confluent REST Proxy 7.2.1, accessible via http://localhost:8082
* Elasticsearch 7.16.2, accessible via http://localhost:9200 (or http://elasticsearch:9200 via Docker networking)
* MySQL 8.0.30, accessible via JDBC at port 3306
* nginx 1.22.0 (stable): a powerfull HTTP server, accessible via http://localhost

![Démonstration Démo SQL Flink](only-sql-overview.png "Demo overview")

## Lancement de l'environnement de démo : 

Optionnel : si vous utilisez un environnement autre que Gitpod, on va d'abord cloner l'environnement : 

```bash

git clone https://github.com/crystalloide/Flink-Kafka

cd Flink-Kafka

```

### Récupération des images et lancements des services : ( option 1 : versions de 2022) 

```bash
docker compose up --build -d
```

### Récupération des images et lancements des services : ( option 2 : dernières versions disponibles à date) 

```bash
docker compose -f docker-compose-latest.yml up -d
```


### Vérification que tous les services sont bien lancés :

```bash
docker compose ps
```

### Lancement du client Flink SQL :

```bash
docker compose run sql-client
```

## Accès au site web de démonstration :

Avec le navigateur, aller sur l'URL http://localhost/flink/flink-docs-master/ 

qui est une copie du site documentaire de Flink et qui correspondra à notre démonstration :

Attention, dans notre cas sur Gitpod, ça sera quelque chose comme ça : (allez sur l'onglet des ports ouverts en bas à droite et cliquez sur l'URL xxx:80)
```bash
https://80-crystalloide-flinkkafka-g87zshy47e8.ws-eu110.gitpod.io/
```

## Analyse en temps réel des comportements des internautes sur le site : 

Chaque visite à l'une des pages web est envoyée vers le Topic Kafka nommé `pageview`. 

Pour ensuite explorer les informations, nous allons d'abord référencer ce topic Kafka en tant que table dans le catalog de Flink :

```sql
--Create table pageviews:
CREATE TABLE pageviews (
    `title` STRING,
    `url` STRING,
    `datetime` STRING,
    `cookies` STRING,
    `browser` STRING,
    `screensize` STRING,
    `ts` TIMESTAMP(3) METADATA FROM 'timestamp',
    `proc_time` AS PROCTIME(),
    WATERMARK FOR `ts` AS `ts` 
) WITH (
    'connector' = 'kafka',
    'topic' = 'pageview',
    'properties.bootstrap.servers' = 'broker:29092',
    'properties.group.id' = 'flink-only-sql',
    'scan.startup.mode' = 'latest-offset',
    'value.format' = 'avro-confluent',
    'value.avro-confluent.schema-registry.url' = 'http://schema-registry:8091'
);
```

Tout cookie appartenant au domaine `localhost` (où notre site Web est exécuté) est également envoyé au topic.
Dans notre exemple, nous nous intéressons tout particulièrement à un cookie appelé `identifier`. 
Nous allons donc enregistrer une vue, qui renvoie cette information en appliquant une expression régulière sur les données entrantes.

```sql
--Create view which already extracts the identifier from the cookies
CREATE TEMPORARY VIEW web_activities AS 
    SELECT 
        `title`,
        `url`,
        `datetime`,
        `cookies`,
         REGEXP_EXTRACT(cookies, '(^| )identifier=([^;]+)', 2) as `identifier`,
        `browser`,
        `screensize`,
        `proc_time`,
        `ts`
    FROM pageviews;
```

En exécutant désormais des requêtes sur la vue lors de la visite d'une page Web, les données apparaîtrons dans le client Flink SQL.

Il s'agit d'une source de données illimitée (en streaming), ce qui signifie que l'application ne s'arrêtera jamais.

```sql 
SELECT * from web_activities;
```

![Flink SQL Client Results](only-sql-results-01.png "Flink SQL Client Results - Unbounded datasource")


## Explorons le comportement historisé sur ce site Web :

Pour la démonstration, nous avons capturé des données historiques sur le comportement constaté sur le site Web. Ceci a été stocké dans la table MySQL `history`.

Pour accéder à ces données, il faut d'abord enregistrer cette table dans le catalogue Flink :

```sql
--Create table history:
CREATE TABLE history (
    `title` STRING,
    `url` STRING,
    `datetime` STRING,
    `cookies` STRING,
    `identifier` STRING,
    `browser` STRING,
    `screensize` STRING,
    `proc_time` STRING,
    `ts` TIMESTAMP(3),
    PRIMARY KEY (identifier) NOT ENFORCED
) WITH (
   'connector' = 'jdbc',
   'url' = 'jdbc:mysql://mysql:3306/sql-demo',
   'table-name' = 'history',
   'username' = 'flink-only-sql',
   'password' = 'demo-sql'
);
```

En exécutant maintenant une requête sur ces données, on retrouve bien les données historisées dans le client Flink SQL.

Il s'agit cette fois d'une source de données limitée (par batchs/lots), ce qui signifie que l'application se terminera après avoir traité toutes les données.


```sql 
SELECT * from history;
```

![Flink SQL Client Results](only-sql-results-02.png "Flink SQL Client Results - Bounded datasource")

## Déterminons les utilisateurs qui correspondent à un certain modèle de comportement : 

Nous utilisons ici la fonction `MATCH_RECOGNIZE` de Flink pour sélectionner tous les identifiants qui correspondent à un modèle spécifique.

Cette fonction est utilisable pour toutes sortes de fonctionnalités de traitement d'événements complexes.

Dans la configuration ci-dessous, nous sélectionnons tous les identifiants qui visitent :

1. http://localhost/flink/flink-docs-master/docs/try-flink/datastream/ followed by (both directly and indirectly)

2. http://localhost/flink/flink-docs-master/docs/try-flink/table_api/ followed by (both directly and indirectly)

3. http://localhost/flink/flink-docs-master/docs/try-flink/flink-operations-playground/

```sql
SELECT `identifier`
FROM web_activities
    MATCH_RECOGNIZE(
        PARTITION BY `identifier`
        ORDER BY `proc_time`
        MEASURES `url` AS url
        AFTER MATCH SKIP PAST LAST ROW
        PATTERN (A+ B+ C)
        DEFINE
            A AS A.url = 'http://localhost/flink/flink-docs-master/docs/try-flink/datastream/',
            B AS B.url = 'http://localhost/flink/flink-docs-master/docs/try-flink/table_api/',
            C AS C.url = 'http://localhost/flink/flink-docs-master/docs/try-flink/flink-operations-playground/'
);
```


## Agir sur les utilisateurs qui correspondent au modèle défini :

Après avoir créé la liste des « identifiants » qui répondent à notre modèle défini, on veut maintenant agir sur ces données.

Pour cela, nous envoyons la liste des « identifiants » vers Elasticsearch.

Le site Web vérifie s'il y a un résultat dans les résultats d'Elasticsearch et, si c'est le cas, il affiche la notification.

Pour envoyer les données à Elasticsearch, il faut d'abord créer une autre table, comme déjà fait précédemment, dans le catalogue de Flink.

On utilise le DDL suivant :


```sql
--Create a sink to display a notification
CREATE TABLE notifications (
    `identifier` STRING NOT NULL,
    `notification_id` STRING,
    `notification_text` STRING,
    `notification_link` STRING,
    PRIMARY KEY (identifier) NOT ENFORCED
) WITH (
    'connector' = 'elasticsearch-7',
    'hosts' = 'http://elasticsearch:9200',
    'index' = 'notifications'
);
```

Lorsque la table est créée, on reprend la requête SQL précédente (qui renvoie la liste des `identifier`)

et on envoie les résultats à la table créée précédemment.


```sql
INSERT INTO notifications (`identifier`, `notification_id`, `notification_text`)
    SELECT 
        T.identifier,
        'MyFirstNotification',
        'Are you trying to hack Flink?'
    FROM web_activities
    MATCH_RECOGNIZE(
        PARTITION BY `identifier`
        ORDER BY `proc_time`
        MEASURES `url` AS url
        AFTER MATCH SKIP PAST LAST ROW
        PATTERN (A+ B+ C)
        DEFINE
            A AS A.url = 'http://localhost/flink/flink-docs-master/docs/try-flink/datastream/',
            B AS B.url = 'http://localhost/flink/flink-docs-master/docs/try-flink/table_api/',
            C AS C.url = 'http://localhost/flink/flink-docs-master/docs/try-flink/flink-operations-playground/'
) AS T;
```

> :avertissement : La valeur par défaut du cookie `identifier` est `anonymous`. Aucune notification ne sera affichée si la valeur est `anonymous`.
>
> Pour modifier la valeur, on ouvre l'IDE  (outil de développement) via Cmd + Opt + J (sur Mac) ou Ctrl + Shift + J (sous Windows)
>
> Dans la console ouverte, vous devez ensuite taper `document.cookie="identifier=YourIdentifier"` pour modifier la valeur du cookie `identifier`.

Si vous modifiez la valeur de votre cookie `identifier` et que vous suivez le pattern défini, une notification sera affichée.

![Affichage d'une notification personnelle](only-sql-results-03.png "Displaying a personal notification")

## Jointure et enrichissement des données de streaming avec les données de batchs :

Un autre cas d'utilisation courant de SQL est que vous devez joindre des données provenant de plusieurs sources.

Dans l'exemple suivant, vous afficherez une notification à l'utilisateur du site Web qui a visité la page d'accueil plus de 3 fois en 10 secondes. 

Si l'`identifier` est `MartijnsMac`, la notification affichera un lien vers le compte Twitter de l'auteur. 

Le pseudo Twitter est récupéré à partir de la source externe.

Dans le cas où l'identifiant est différent, aucun lien ne sera inclus.

La première chose à faire est de créer une autre table afin de pouvoir se connecter aux données :


```sql
CREATE TABLE customer (
    `identifier` STRING,
    `fullname` STRING,
    `twitter_handle` STRING,
    PRIMARY KEY (identifier) NOT ENFORCED
) WITH (
   'connector' = 'jdbc',
   'url' = 'jdbc:mysql://mysql:3306/sql-demo',
   'table-name' = 'customer',
   'username' = 'flink-only-sql',
   'password' = 'demo-sql'
);
```

Nous allons utiliser une fonction "Window Table-Valued " pour déterminer quels identifiants ont visité la page d'accueil plus de 3 fois :

```sql
SELECT window_start, window_end, window_time, COUNT(`identifier`) AS `NumberOfVisits` FROM TABLE(
   TUMBLE(TABLE web_activities, DESCRIPTOR(ts), INTERVAL '10' SECONDS))
   WHERE `url` = 'http://localhost/flink/flink-docs-master/'
   GROUP BY window_start, window_end, window_time
   HAVING COUNT(`identifier`) > 3;
```

Le résultat de la fonction Window Table-Valued peut également être combiné dans une jointure JOIN. 

On va joindre les résultats précédents avec les données de la table `customer` précédemment enregistrée pour enrichir le résultat. 

Nous allons utiliser le DDL suivant pour cela :

```sql
SELECT w.identifier,
       COALESCE(c.fullname,'Anonymous') as `fullname`,
       COALESCE(c.twitter_handle,'https://www.google.com') as `twitter_handle`
FROM(
       SELECT `identifier`
       FROM TABLE(TUMBLE(TABLE `web_activities`, DESCRIPTOR(ts), INTERVAL '10' SECONDS))
       WHERE `url` = 'http://localhost/flink/flink-docs-master/'
       GROUP BY `identifier`
       HAVING COUNT(`identifier`) > 3 ) w
LEFT JOIN(
       SELECT *
       FROM customer ) c
ON w.identifier = c.identifier
GROUP BY w.identifier,
         c.fullname,
         c.twitter_handle;
```

Avec une simple modification du DDL ci-dessus, on peut utiliser le résultat pour afficher un aperçu exploitable de ces visiteurs :

```sql
INSERT INTO notifications (`identifier`, `notification_id`, `notification_text`, `notification_link`)
SELECT w.identifier,
       'MySecondNotification',
       CONCAT('Welcome ', COALESCE(c.fullname,'Anonymous')),
       COALESCE(c.twitter_handle,'https://www.google.com')
FROM(
       SELECT `identifier`
       FROM TABLE(TUMBLE(TABLE `web_activities`, DESCRIPTOR(ts), INTERVAL '10' SECONDS))
       WHERE `url` = 'http://localhost/flink/flink-docs-master/'
       GROUP BY `identifier`
       HAVING COUNT(`identifier`) > 3 ) w
LEFT JOIN(
       SELECT *
       FROM customer ) c
ON w.identifier = c.identifier
GROUP BY w.identifier,
         c.fullname,
         c.twitter_handle;
```

![Affichage d'une notification avec lien](only-sql-results-04.png "Displaying a personal notification on how to contribute")
