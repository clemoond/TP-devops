# TP DEVOPS

## TP1

### Question 1.1 : 
> *Documentez les éléments essentiels de votre conteneur de base de données : commandes et Dockerfile.*

Les commandes :
```
1. docker pull postgres
2. docker pull adminer
3. docker run -p "8090:8080" --network --name=adminer -d adminer
4. docker build . -t mydb
5. docker run -d --network app-network --name db mydb
```
- La commande 3 permet de lancer l'adminer.
- La commande 4 permet de construire l'image en la nommant mydb.
- La commande 5 permet de lancer la base de données.

Dans le Dockerfile, il faut rajouter les 2 lignes suivantes pour supprimer le conteneur db, reconstruire l'image et relancer le conteneur :
````
COPY CreateScheme.sql /docker-entrypoint-initdb.d
COPY InsertData.sql /docker-entrypoint-init-db.d
````

### Question 1.2 :
> *Pourquoi avons-nous besoin d'une construction en plusieurs étapes ? Et expliquez chaque étape de ce fichier docker.*

Nous avons besoin d’une construction en plusieurs étapes pour avoir une image Docker légère en ayant tous les composants nécessaires pour exécuter l’application. 
Le Dockerfile est composé de 2 grandes étapes :
-	La construction de l’application : en compilant avec Maven et Amazon Correto et en copiant pom.xml et le répertoire src dans le conteneur
-	L’exécution de l’application : en copiant le fichier .jar créé à l’étape d’avant et en l’exécutant

### Question 1.3 :
> *Documenter les commandes les plus importantes pour le docker-compose.*

On a créé un nouveau dossier http avec un Dockerfile composé des 2 lignes suivantes :
```
FROM httpd:2.4
COPY ./index.html /usr/local/apache2/htdocs/
```

Ensuite, on a construit l'image correspondante et lancé le contenur grâce aux commandes suivantes :
```
docker build -t myapache2 .
docker run -d --name my-front-app --network app-network -p 8080:80 my-apache2
docker cp my-front-app:/usr/local/apache2/conf/httpd.conf httpd.conf
```
Enfin, on a ajouté une 3ème ligne dans le Dockerfile :
```
COPY ./httpd.conf /usr/local/apache2/conf/httpd.conf
```
et on a reconstruit l'image et relancé le conteneur.

### Question 1.4 :
>*Documenter votre fichier docker-compose.*

Le fichier docker-compose ressemble à :
```
version: '3.7'

services:
    backend:
      container_name: my-running-app
      build: ./simple-api-student-main
      networks: 
        - app-network
      depends_on: 
        - database

    database:
      container_name: db
      build: ./Database
      networks: 
        - app-network

    httpd:
      container_name: my-front-app
      build: ./http
      ports: 
        - "8080:80"
      networks: 
        - app-network
      depends_on: 
        - backend

networks:
  app-network:
```
Ce fichier définit les 3 services suivants qui seront exécutés dans des conteneurs séparés :
* __backend__ : construction d'un conteneur avec ajout au réseau, et qui dépend de la database
* __database__ : construction d'un conteneur et ajout au réseau
* __httpd__ : construction d'un conteneur, ajout au réseau et qui dépend du backend

### Question 1.5 :
>*Documentez vos commandes de publication et vos images publiées dans Dockerhub.*

La 1ère commande est ```docker compose up``` qui permet de tout lancer d'un coup.
Ensuite, pour chaque image on ajoute un tag avec la commande suivante : ```docker tag tp-database clemenced69/tp-database:1.0``` et enfin la commande ```docker push clemenced69/tp-database:1.0``` permet de publier l'image dans Dockerhub.

------------------------------------------------------------------------
## TP2
### Question 2.1 :
>*Que sont les conteneurs de test ?*

Le conteneur de test est une bibliothèque Java Open Source qui permet de simplifier le processus de création et de gestion des conteneurs pendant l'exécution des tests.

### Question 2.2 :
>*Documentez vos configurations d'actions Github.*

La configuration se trouve dans le ```main.yml```:
```
name: CI devops 2023
on:
  push:
    branches:
      - main
      - develop
  pull_request:

jobs:
  test-backend:
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/checkout@v2.5.0

      - name: Set up JDK 17

        uses: actions/setup-java@v3
        with:
          java-version: 17
          distribution: 'adopt'

      - name: Build and test with Maven
        working-directory: simple-api-student-main
        run: mvn clean verify
```

Ce code permet de lancer les actions Github lorsqu'on push sur les branches ```main``` et ```develop```. 
Les actions en question sont :
- récupérer le code à partir du référentiel Github
- configurer la version 17 de Java
- construire et lancer l'application ```simple-api-student-main```


### Question 2.3 :
>*Documentez la configuration de votre portail qualité.*

La commande pour accéder au portail qualité sur Github est la suivante :
```
run: mvn -B verify sonar:sonar -Dsonar.projectKey=clemencedebadier_clemenced -Dsonar.organization=clemencedebadier -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }}  --file ./pom.xml
```

------------------------------------------------------------------------
## TP3

### Question 3.1 :
>*Documentez votre inventories et vos commandes de base.*

Dans le dossier ```inventories```, on a ajouté un fichier ```setup.yml``` composé de :
```
all:
  vars:
    ansible_user: centos
    ansible_ssh_private_key_file: "/home/clem_admin/id_rsa"
  children:
    prod:
      hosts: clemence.debadier.takima.cloud
```
La commande utilisée pour tester la connectivité est la suivante :
``` 
ansible all -i inventories/setup.yml -m ping
```
On peut remplacer ```-m ping``` par ```-m setup``` pour collecter les informations et par ```-m yum``` pour la gestion des paquets.

### Question 3.2 :
>*Documentez votre fichier ```playbook.yml```.*

Le fichier ```playbook.yml``` ressemble à :
```
- hosts: all
  gather_facts: false
  become: true
  tasks:
    - name: Install device-mapper-persistent-data
      yum: 
        name: device-mapper-persistent-data
        state: latest
    
    - name: Install lvm2
      yum: 
        name: lvm2
        state: latest
    
    - name: add repo docker
      command:
        cmd: sudo yum-config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
    
    - name: Install Docker
      yum: 
        name: docker-ce
        state: present
        
    - name: Make sure Docker is running
      service: name=docker state=started
      tags: docker     
```
Ce script permet d'automatiser le processus d'installation de Docker.  
Les deux premières tâches installent les modules ```device-mapper-persistent-data``` et ```lvm2``` en utilisant la dernière version disponible.  
La tâche suivante exécute la ligne de command qui permet d'ajouter le Docker à la configuration.  
Les deux dernières tâches permettent d'installer le Docker puis de s'assurer que le service Docker est bien en cours d'exécution.


### Question 3.3 :
>*Documentez la configuration de vos tâches docker_container.*

Dans cette partie, on a modifié le fichier ```playbook.yml``` afin d'ajouter des rôles pour chaque tâche (docker, network, database, app et proxy) :

```
- name: Deploy My Application
  hosts: all
  gather_facts: false
  become: yes
  roles:
    - docker
    - network
    - database
    - app
    - proxy
```

On a alors créé un dossier ```rôles``` avec des packages`pour chaque rôle.  
Pour le rôle ```docker```, le contenu du fichier permet d'installer le docker de manière automatique :
```
- name: Install device-mapper-persistent-data
  yum:
    name: device-mapper-persistent-data
    state: latest

- name: Install lvm2
  yum:
    name: lvm2
    state: latest

- name: add repo docker
  command:
    cmd: sudo yum-config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo

- name: Install Docker
  yum:
    name: docker-ce
    state: present

- name: Make sure Docker is running
  service: name=docker state=started
  tags: docker
```

Pour le rôle ```network```, le code suivant permet de créer le réseau utilisé par le Docker :
```
- name: Create Docker Network
  docker_network:
    name: app-network
    state: present
```

Pour le rôle ```database```, le code suivant permet de lancer la base de données, le conteneur Docker correspondant et de créer l'image correspondante :
```
- name: Run database
  docker_container:
    name: db
    image: clemenced69/database:1.0
    networks:
      - name: app-network
```

Pour le rôle ```app```, le code suivant permet de lancer le backend, le conteneur et l'image correspondante :
```
- name: Run backend
  docker_container:
    name: my-running-app
    image: clemenced69/simple-api-student-main:latest
    networks:
      - name: app-network
```

Pour le rôle ```proxy```, le code suivant permet de lancer le frontend sur le port 80, le conteneur et l'image correspondante :
```
- name: Run Proxy Container
  docker_container:
    name: my-front-app
    ports:
      - "80:80"
    image: clemenced69/http:1.0
    networks:
      - name: app-network
```

------------------------------------------------------------------------



