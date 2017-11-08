Meetup Modern Devops #1 Microservices Architectures @Criteo

Le 30 Octobre 2017 - Prise de notes par Kevin Nadin @kevinjhappy



Conférence 1 

@benoitc
Base d’un bon micro service

Est ce que mon service fait vraiment une seule chose ?
Est t’il autonome ? Si il plante il faut que je puisse le remplacer rapidement
Il doit aussi contenir ses propres données.
Chaque MS doit contenir ses propres données localement

Important, partager ses données entre les MS, de façon à ce qu’on puisse le faire cracher sans faire cracher les autres
Si il plante il n’a pas d’impacts

Cloud Storage :
Client qui attaque un webservice, et récupère des données, on cherche des solutions complexe pour cacher les données, et aussi pour être sur que la base puisse résister.
Faire un cache qui est performante est compliqué au dessus du réseau, on va vouloir s’assurer que les données sont à jour

Beaucoup de logique au niveau du cache.
Idée : construire ma solution de cache, et avoir un stockage qui soit local, et faire mes requêtes chez moi et ne pas me sync avec un service à distance

Même les systèmes les plus consistants sont capable de travailler si le réseau est en panne.
Je Query, j’update mes données localement, et je sync en asynchrone

Outil Barrel : sortie prod en 11/2017

BDD focus sur la simplicité, proche de graphQL ou falkor ou firebase, données automatiquement indexé, accès aux données à travers des chemins / requête
On stock des documents, local first, data is sync with other storage
Pas besoin de dupliquer toute les données,
Données en format JSON
Stock un document avec plusieurs locations le système d’inflexion découpe ces docs sous forme d’arbre, et lorsque l’on fait une requête on interroge le chemin spécifique de l’arbre.
Pas besoin de chercher à indexer
 avantages :
- Design de table change, migrer les données et indexation, ici pas besoin de s’en soucier, l’index continue d’indexer les chemins

Possible avec HTTP 1,2
Stockage, embarquer une logique

Comment c’est fait ?
Logique similaire au crut (horloge sémantique, vecteur clock)
Transactions possible comme une suite d’opération
Les index sont répliqués

Consistency

Pas de vecteur clock, mais un arbre de révision
A chaque fois q’un nouveau design est fait, y’a une branche, si conflit review faite, système de PR classique

Client Alice : a -> b 
Client Iko : a -> commit => récupère b
Client. Bob : a -> b => conflit =>faire comme branche b et c  en parallèle -> merge d

Bob se sync avec Alice et ico => ils ont tous le document d

Tous les peer fork le master, update sont offline
Peer pull and merge from the main server
Works well for back pressure (writes can be delayed)



Conférence 2 - Dockering chez Meetic

Sebastien Le Gall - lead back end at meetic

Docker in a world of events and micro service

Meetic avant, on avait des mobile web, webservice, crontab, desktop, back offices, tout en php 4 dans un seul serveur monolithique dans une grosse base Oracle

Meetic now : frontaux -> exposition layer avec symfony (envoi les données) couche micro service en DDD, un micro service par domaine, stock en MS auto suffisant
Ex : gérer la photo, gérer les profils, etc…
Backoffice sortie
Plusieurs BDD, mysql, nosql
Event bus -> consumers

Problématiques : 32 micro service, 41 consommateurs (un par MS + autres)
3 Gateway, on a choisi de passer sur docker 

Architecture des conteneurs

« One CI to rule all the micro-service »
Gérer les environnement de recette, tester un incrément technique, mais tester une user story fait intervenir plusieurs MS, et on doit pouvoir gérer les différentes version X 150 devs qui peuvent tester

Meetic before : Un gros serveur avec pleins de PC connecté en SSH dessus
Dev now : chaque environnement de dev se fait sur ses conteneurs docker en local, chaque dev a le code source en local
CI before : Jenkins qui tourne on déclenche un job mais avec plusieurs jobs qui se font les uns après les autres, Jenkins est surchargé, même avec plusieurs slaves.
CI now : toujours à base de Jenkins, JFrog Artifactory, tous les noeuds sont fait en docker, et docker compose passe les micro services en prod
Jenkins en docker aussi ce qui permet de reprendre le travail en cas de crash
On pousse des conteneurs qui vont contenir le code source

Testing environnement

On part sur une utilisation de Cube, on oublie les codes en SSH déposé zippé, il faut récupérer les images dockers qui contienne le code source
On part sur des namespace, pour configurer tous ces conteneurs
Container architecture, on prends le plus possible des conteneurs officiels,

CentOS -> docker Meetic/centOS -> meetic /php5.6 -> MEETIC / PHP-CLI 5.6 -> plusieurs conteneurs
Toutes les technos ont leur conteneurs
Conteneurs par micro service (un pour la photo, un pour la geoloc, un par consommateur,…) 
CI pour ces conteneurs
Feature -> branch -> build le conteneur-> buildable ? -> push sur master -> build -> pousse sur la registry

Dev : make install -> docker built
Make up -> docker compose up
./run_cli.sh -> démarre un Batch dans un conteneur php pour avoir de la ligne de commande

Volume sur la machine auth, y’a le code source les logs, la config, script elasticSearch, 
On peut travailler en mode bash dans ces conteneurs, permet d’exécuter du code en bash
Sur le code source et pouvoir l’exécuter

On a pleins de stack
Faire tourner un Kafka sur le pc c’est pas ouf
Trop lourd d’avoir un seul docker compose qui prend toute la stock

4 use case de dev

- Make up
- Make db-up
- Make Kafka-up
- Make db-kafka-up

CI à gérer
50 conteneurs avec héritage, consommateur 40, on ne peut pas faire un job Jenkins à chaque repo, faire des mono repo
Php cli-controller -> php project generator-> folder generator-> merge request handler ->generate some stuff
À la fin un job Jenkins par pull request, pour éviter de se geéner
Publie un artefact, -> code buildé avec ses dépendances, et on pousse une image docker correspondante

A CI environnement per micro service

Jenkinsfile define what pipeline script have to be executed
One docker compose per microserice
Some images are built on the go in order to be able to override some configuration value during tests

CI now : pousse une PR, trier un Jenkins, fait les tests, crée une image de docker avec le code
Consumer en scala
Micro service en php 
=> permet les docker file générique

Environnement de test

3 choses difficile, validation de cache, nommer des objets, tester les micro services
Intro de kubernetes : orchestrateur de conteneur, dans tel namespace je veux tel conteneur, et son rôle est de s’assurer qu’ils sont toujours open running, son rôle peut marcher pour les updates, il fait le travail de l’ouvrir, update la version, et le figer
Un namespace ou plus par user
Toutes les nuits cela lance des tests, on crée un namespace à la volée
On monte les conteneurs en prod et on surcharge que les conteneurs dont on a besoin

Introducing Kubergen : JSon en entrée, microservices avec versions, avec ce json on choisi les versions que l’on veux utiliser.

Conclusion

Optimiser est important, l’héritage aussi sur les conteneurs

Donner aux images une base, les laisser override la configuration

Ne pas traiter tous les use case de la même manière

Interaction avec Git : on travaille dans un volume qui est en local, je commit sur mon PC, qui a un volume partagé sur mon PC, je fais mes tests puis je commit et je push sur le cloud, qui lui va faire des tests unitaire lorsque cela va builder

Mononith -> microservice en 2ans et 1/2, beaucoup de formation, de turnover, toutes les machines n’étaient pas concrétisés.

Les micro services sont déployés sur les cent machines, pas encore une super scalabilité



Conférence 3

Instana presentation

Tools to monitor micro services activity
