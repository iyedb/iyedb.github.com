---
layout: post
title: "Récupération d'une archive de base de données PostgreSQL"
date: 2013-08-26 00:52:17
categories: PostgreSQL
---

Dans ce billet, nous allons récuperer une archive ou une sauvegarde ("un dump")
d'une base de données PostgreSQL.  
La base de données en question s'appelle "DVD rental". Il s'agit d'une base 
de données exemple utilisée par une chaine de boutiques fictive de location de 
DVD. Cette base de données est décrite 
[ici](http://www.postgresqltutorial.com/postgresql-sample-database/)
et l'archive PostgreSQL correspondante que nous allons récuperer est 
téléchargeable 
[ici][archive]

Les commandes qui suivent ont été exécutées sous OS X et un serveur 
PostgreSQL version 9.2.2. Si vous êtes sous OS X, l'installation d'un serveur
PostgreSQL est très simple: téléchargez l'application 
[Postgres.app](http://postgresapp.com/) et copiez l'application dans 
le répertoire "Applications" de votre Mac, exécutez l'application et le tour
est joué. Il vaut mieux aussi ajouter le chemin 
`/Applications/Postgres.app/Contents/MacOS/bin` à `PATH` afin d'avoir 
accès au commandes de PostgreSQL.
Si vous êtes sous Ubuntu, PostgreSQL est surement déjà installé.
Cependant, si jamais ce n'est pas le cas ou si la version n'est pas une 9.2.x,
visitez cette [page](http://www.postgresql.org/download/linux/ubuntu/) sur 
le site officiel de PostgreSQL et suivez les instructions.

Passons à l'essentiel:

+ Téléchargez le fichier de [l'archive][archive] et décompressez le avec `unzip`.
Vous obtiendrez le fichier "dvdrental.tar".

Assurez vous d'avoir un utilisateur avec le nom "postgres" déclaré dans PostgreSQL.
Si ce n'est pas le cas il faudra le créer sinon la restauration échouera 
(cette d'archive fait référence à l'utilisateur "postgres"):

+ Connectez-vous à PostgreSQL avec la commande `psql`
+ Dans la console `psql` tapez la commande suivante:  
`create user postgres superuser password 'postgres';`
+ Créez la base de données "dvdrental":  
`create database dvdrental;`
+ Quittez psql: `ctrl-d`
+ On doit  maintenant  pouvoir se connecter à la base "dvdrental" sur le 
serveur PostgreSQL en tant qu'utilisateur "postgres":  
`psql -U postgres -d dvdrental`
+ On peut maintenant lancer la commande de restauration de la base de données 
grâce à la commande de restauration de PostgreSQL `pg_restore`:  
`pg_restore -U postgres -d dvdrental dvdrental.tar`  
`pg_restore` se connecte au serveur PostgreSQL en tant qu'utilisateur "postgres"
et en spécifiant la base de données "dvdrental" pour y restaurer notre archive.

Et voilà! Notre archive est restaurée. On peut maintenant commencer à jouer 
avec:  
`psql -U postgres -d dvdrental`  
Dans la console psql, tapez:  
`select * from actor;` par exemple pour afficher la liste des acteurs.
On peut maintenant s'amuser à explorer et exploiter cette base de données en utilisant le langage SQL.
Le site [postgresqltutorial](http://www.postgresqltutorial.com/) contient justement 
d'excellents articles sur ce sujet.

Pour conclure, l'intérêt de cette base de données exemple est qu'elle 
représente un exemple réel de cas d'utilisation et surtout qu'elle est préremplie.
Elle peut donc servir
de terrain de jeux pour s'exercer à SQL mais aussi au développement d'applications 
web. D'ailleurs, dans un prochain billet, nous utiliserons le framework 
[Play](http://www.playframework.com) avec le langage [scala](http://www.scala-lang.org/) 
pour se connectez à cette base de données.
[archive]: http://www.postgresqltutorial.com/?wpdmact=process&did=MS5ob3RsaW5r


