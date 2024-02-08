# DevOps - Compte rendu 

## TP 01 - Docker ✈️✈️✈️

- DockerFile 🐳:

``` yaml
FROM postgres:14.1-alpine

ENV POSTGRES_DB=db \
   POSTGRES_USER=usr \
   POSTGRES_PASSWORD=pwd

```

- Pour build 🔧 :
```shell
sudo docker build -t tp1/tp1 .
```

- pour lancer le run en détach : 👨‍🦽

```shell
sudo docker run --name tp1 tp1/tp1
```

❓ Why should we run the container with a flag -e to give the environment variables? 🇺🇸

💡 	 Il faut lancer le container avec le flag -e pour préciser les variables d'environment. Pour une database, il est obligatoire d'avoir un superuser avec un mot de passe. 🇫🇷

### Connexion à une base de donnée PostGres

- Ajouter un network 🕸️
```shell
sudo docker network create app-network
```
- Relancer le container tp1 avec le network ♻️

``` shell
sudo docker run --net=app-network -d --name=tp1 tp1/tp1
```

- Adminer 🕵️‍♂️

```shell
sudo docker run \
    -p "8090:8080" \
    --net=app-network \
    --name=adminer \
    -d \
    adminer
```

### Connexion depuis le navigateur sur le port 8090

- infos de logs 🧑‍💻

```shell
    System:	PostGreSQL
    Server: tp1
    Username: usr
    Password: pwd
    Database: db	
```
❗ Les identifiants de connexions sont soit fournies en ligne de commande avec le flag `-e` soit spécifié dans le dockerfile.

### Lancement d'une database avec un set de donnée:

Tous les scripts copiés dans `docker-entrypoint-initdb.d` seront executés. On rajoute donc au Dockerfile 📝: 

```shell
COPY CreateScheme.sql /docker-entrypoint-initdb.d/
COPY InsertData.sql /docker-entrypoint-initdb.d/
```
### Persistance des données 

Pour persister la donnée, il faut utiliser un volume. Pour ce faire il faut préciser avec le flag `-v`. Il prend 3 paramètres dont un optionnel:
- Nom du volume
- Chemin où le volume sera monté dans le container
- Liste d'option (optionnel) 

❓ Why do we need a volume to be attached to our postgres container? 🇺🇸

💡 Il faut un volume attaché au container afin de persister la donnée. Si on détruit le container et qu'il n'a pas de volume attaché, la base de donnée est réinitialisée à son prochain lancement.🇫🇷

## Backend API

### Java Application

DockerFile 🐳:

```
FROM openjdk:11
COPY ./Main.java /usr/src/
WORKDIR /usr/src/
RUN javac Main.java
CMD ["java", "Main"]
```
Build 🔧:

```
sudo docker build -t java/backend . 
```

Run 👨‍🦽 : 

```
sudo docker run --name backend java/backend    
```
### Springboot Application

❗ Il faut bien exposer les ports docker sinon l'application springboot ne sera pas accessible. 

DockerFile 🐳:
```
# Build
FROM maven:3.8.6-amazoncorretto-17 AS myapp-build
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY pom.xml .
COPY src ./src
RUN mvn package -DskipTests

# Run
FROM amazoncorretto:17
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar

ENTRYPOINT java -jar myapp.jar
```
Build 🔧:
```
sudo docker build -t api/simpleapi .
```

Run 👨‍🦽 : 
```
sudo docker run --name=simpleapi -p 8080:8080 api/simpleapi
```

❓ Why do we need a multistage build? And explain each step of this dockerfile. 🇺🇸

💡 Un build multistage est un build qui utilise plusieurs fois l'instruction `FROM`. Chaque `FROM` constitue une étape du build. Cela nous permet d'utiliser différentes bases.🇫🇷

Dans notre cas, il nous faut d'abord build notre application java en téléchargeant les dépendances maven. On le run ensuite avec le JDK d'amazon.

### Backend API 

Fichiers téléchargés depuis le [git](https://github.com/takima-training/simple-api-student) mis à disposition.


Le Dockerfile est identique à celui précédemment.

Il faut cependant bien lancer le container dans le réseau crée précédemment où tourne notre base de donnée PostGreSQL.

Run 👨‍🦽 : 

```
sudo docker run --name=simpleapi-student \
--net=app-network \
-p 8081:8080 \
java/simple-api-student
```

Configuration de l'appli avant le build pour la connexion à la db: 

Application.yaml: 
```
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          lob:
            non_contextual_creation: true
    generate-ddl: false
    open-in-view: true
  datasource:
    url: jdbc:postgresql://tp1:5432/db
    username: usr
    password: pwd
    driver-class-name: org.postgresql.Driver
management:
 server:
   add-application-context-header: false
 endpoints:
   web:
     exposure:
       include: health,info,env,metrics,beans,configprops
```
On a accès grace au mapping des calls sur le port 8081 à la database. ✅

## HTTP Server

Configuration basique:
On crée un nouveau dossier `public-html` où l'on stockera un index.html classique.

DockerFile 🐳:
```
FROM httpd:2.4
COPY ./public-html/ /usr/local/apache2/htdocs/
```
Build 🔧:

`sudo docker build -t http/http .`

Run 👨‍🦽 :

`sudo docker run -dit --name my-running-app -p 8080:80 http/http`

### Configuration 
Afficher la configuration globale apache:

`sudo docker exec my-running-app cat /usr/local/apache2/conf/httpd.conf`

### Reverse proxy

On créer un fichier de confif à partir de celui de base :

```
sudo docker exec my-running-app cat /usr/local/apache2/conf/httpd.conf >> httpd.conf
```

On modifie le fichier de conf apache depuis le Dockerfile en rajoutant:
- Le nom du serveur :
`ServerName localhost`
- La config du proxy : 
```
<VirtualHost *:80>
ProxyPreserveHost On
ProxyPass / http://simpleapi-student:8080/
ProxyPassReverse / http://simpleapi-student:8080/
</VirtualHost>
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so
```

Dockerfile :

```
FROM httpd:2.4
COPY ./public-html/ /usr/local/apache2/htdocs/
COPY ./httpd.conf /usr/local/apache2/conf/httpd.conf
```


On rebuild et on relance:

```
sudo docker run --net=app-network --name my-running-app -p 8080:80 http/http
```

❓ Why do we need a reverse proxy? 🇺🇸

💡 Pour ne pas exposer directement le port de notre container. On doit forcément passer par notre front. 🇫🇷

### Link application

Docker compose :

```
version: '3.7'

services:
  db:
    build: database
    container_name: postgres-db
    environment:
      - POSTGRES_DB=db
      - POSTGRES_USER=usr
      - POSTGRES_PASSWORD=pwd
    networks:
      - app-network

  springboot-app:
    build: simple-api-student
    container_name : springboot-app-container
    environment:
      - URL=db:5432
      - POSTGRES_DB=db
      - POSTGRES_USER=usr
      - POSTGRES_PASSWORD=pwd
    networks:
      - app-network
    depends_on:
      - db
      
  my-apache-server:
    build: my-apache-server
    container_name: my-apache-server-container
    ports:
      - "8081:80"
    networks:
      - app-network
    depends_on:
      - springboot-app
networks:
  app-network:
    driver: bridge
```
On met à jour application.yaml de notre springboot pour prendre en charge les variables d'environnements: 
```
applications.yaml
  datasource:
    url: jdbc:postgresql://${URL}/${POSTGRES_DB}
    username: ${POSTGRES_USER} 
    password: ${POSTGRES_PASSWORD}
```
De même pour le serveur apache, on met à jour le fichier de conf httpd.conf:
```
ProxyPass / http://springboot-app:8080/
```
ProxyPassReverse / http://springboot-app:8080/

❓ Why is docker-compose so important? 🇺🇸

💡 Car cela permet de lancer & de configurer plusieur docker de manière automatique. 🇫🇷

## Publish

Réaliser principalement dans le TP 02 avec le déploiement continue.

❓ Document your publication commands and published images in dockerhub. 🇺🇸

💡 On tag les images pour les identifiés et on les versionne. Cela nous permettra par la suite de récupérer des versions biens spécifiques de nos applications lors de déploiement. 🇫🇷

## TP 02 - Github Action 🚀🚀🚀

### Setup Github Actions

GitHub Action est utilisé tout au long de ce TP. Il permet de lancer des **`jobs`**, des tâches à effectué lors d'un déploiement. On peut choisir à quel moment déclencher ces actions.

### First steps into the CI World

Il faut créer un workflow, un dossier .github & un sous dossier workflows, contenant les routines qui vont être executés. Cette routine est contenue dans le fichier `main .yml`. 

### Build and test your Application

```bash
mvn clean verify
```

Permet de lancer tous les tests présent dans le projet précisé dans le pom.xml.
Le flag `clean` permet de rebuild entièrement l'application, même si aucun morceau de code n'a été touché.

❓ What are testcontainers? 🇺🇸

💡 Les `testcontainers` permettent de réaliser des tests unitaires sans avoir à lancer toute l'application. Leur avantage principale est qu'ils sont portables et léger (car conteneurisé). 🇫🇷

```yml
name: CI devops 2023
on:
  #to begin you want to launch this job in main and develop
  push:
    branches: [master] 
  pull_request:

jobs:
  test-backend: 
    runs-on: ubuntu-22.04
    steps:
     #checkout your github code using actions/checkout@v2.5.0
      - uses: actions/checkout@v2.5.0

     #do the same with another action (actions/setup-java@v3) that enable to setup jdk 17
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
            java-version: '17'
            distribution: 'temurin'
     #finally build your app with the latest command
      - name: Build and test with Maven
        working-directory: simple-api-student/
        run: mvn clean verify
```

❓ Document your Github Actions configurations:

```yml
name: CI devops 2023
on:
  #to begin you want to launch this job in main and develop
  push:
    branches: [master] 
  pull_request:
```

- 💡 On précise à quel moment lancer les `jobs`. Ici, lors d'un push sur la branche master sur GitHub. On ne lance rien lors d'une pull request. 🇫🇷
``` yml
jobs:
  test-backend: 
    runs-on: ubuntu-22.04
    steps:
     #checkout your github code using actions/checkout@v2.5.0
      - uses: actions/checkout@v2.5.0

     #do the same with another action (actions/setup-java@v3) that enable to setup jdk 17
      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
            java-version: '17'
            distribution: 'temurin'
     #finally build your app with the latest command
      - name: Build and test with Maven
        working-directory: simple-api-student/
        run: mvn clean verify
```
- 💡 On a ici un seul job, `test-backend`. On précise son environnements, et les étapes qu'il devra suivre. On n'en a ici qu'une seule, composée de trois sous partie. Dans la première on récupère le dernier commit du repo. Dans la deuxième, on set up notre java en indiquant sa version & sa distribution. Enfin, on lance nos tests unitaire avec le `mvn clean verify`. On précise cependant avant l'emplacement du dossier car le repo contient plusieurs autres projets. 🇫🇷

### First steps into the CD World

Le but de cette partie est de permettre une livraison continue du code. À chaque push, on lance nos batterie de test & on build des images dockers qui sont ensuite publié sur dockerhub.

❓ Secured Variables, why? 🇺🇸

💡 Afin de ne pas rendre public des données sensibles qui pourraient rendre vulnérables nos projets (clé ssh, identifiant github, identifiant database etc... ). 🇫🇷

On déclare donc plusieurs variables secrète qu'on pourra accéder dans notre `main.yml` public sans qu'elles ne soient écritent en clair.

Pour build & push sur nos repo dockerhub, on rajoute un `job` à notre `main.yml`, **build-and-push-docker-image** :

```yml
build-and-push-docker-image:
    needs: test-backend
    # run only when code is compiling and tests are passing
    runs-on: ubuntu-22.04

    # steps to perform in job
    steps:
      - name: Checkout code
        uses: actions/checkout@v2.5.0

      - name: Login to DockerHub
        run: docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build image and push backend
        uses: docker/build-push-action@v3
        with:
          # relative path to the place where source code with Dockerfile is located
          context: ./simple-api-student
          # Note: tags has to be all lower-case
          tags:  ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-simple-api:latest
          push: ${{ github.ref == 'refs/heads/master' }}
      # DO the same for database
      - name: Build image and push database
        uses: docker/build-push-action@v3
        with:
          context: ./database
          tags:  ${{secrets.DOCKERHUB_USERNAME}}/tp-database:latest
          push: ${{ github.ref == 'refs/heads/master' }}
      # DO the same for httpd
      - name: Build image and push httpd
        uses: docker/build-push-action@v3
        with:
          context: ./my-apache-server
          tags:  ${{secrets.DOCKERHUB_USERNAME}}/my-apache-server:latest
          push: ${{ github.ref == 'refs/heads/master' }}
```

❓Why did we put needs: build-and-test-backend on this job?  🇺🇸

💡 Il y a un ordre à respecter car pour build notre image docker, il faut d'abord que notre application est été compilée au préalable. 🇫🇷

❓ For what purpose do we need to push docker images? 🇺🇸

💡 En pushant les images dockers, elles devient récupérables depuis n'importe qu'elle docker. Cela peut s'avérer très utile pour pouvoir déployer facilement & rapidement. 🇫🇷

### Setup Quality Gate

**Sonar** : Plateforme de monitoring de code à des fins d'analyse.

Après avoir configuré sonar en le liant au repo GitHub, on ajoute l'analyse dans notre fichier `main.yml` : 

```yml
      - name: Build and test with Maven
        working-directory: simple-api-student/
        run: mvn -B verify sonar:sonar -Dsonar.projectKey=Thomas0507_devOps-TP01 -Dsonar.organization=thomas0507 -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }}
```

Il a fallu rajouter les token sonar à nos secrets GitHub.

❓Document your quality gate configuration. 🇺🇸

💡 On utilise `sonar way`, le quality gate par défaut. Le code passe le quality test si :

- Aucun nouveau bug
- Aucune nouvelle vulnérabilité
- New code has limited technical debt
- Tous les nouveaux `security hotspots` sont couverts
- Assez de test pour couvrir le nouveau code (**> 80 %**)
- Peu de duplication dans le nouveau code (**< 3%**) 🇫🇷

###     Bonus: split pipelines (Optional)

Non réalisé :c

## TP 03 - Ansible 🛸🛸🛸

On installe ansible & on lui crée un inventaire à la racine:

```
mkdir ansible
cd ansible
mkdir inventories
cd inventories
touch setup.yml
```

Contenue du setup.yml:
```
all:
 vars:
   ansible_user: centos
   ansible_ssh_private_key_file: /home/pc/.ssh/id_rsa
 children:
   prod:
     hosts: thomas.marin.takima.cloud
```

On peut maintenant ping:

```
sudo ansible all -i ansible/inventories/setup.yml -m ping
```

Réponse:

```
thomas.marin.takima.cloud | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}
```
Vérifier la conf:

```
ansible all -i ansible/inventories/setup.yml -m setup -a  "filter=ansible_distribution*"
```
Réponse:
```
thomas.marin.takima.cloud | SUCCESS => {
    "ansible_facts": {
        "ansible_distribution": "CentOS",
        "ansible_distribution_file_parsed": true,
        "ansible_distribution_file_path": "/etc/redhat-release",
        "ansible_distribution_file_variety": "RedHat",
        "ansible_distribution_major_version": "7",
        "ansible_distribution_release": "Core",
        "ansible_distribution_version": "7.9",
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false
}
```

Si on avait httpd installé sur le serveur ansible, on le désinstalle:

```
ansible all -i ansible/inventories/setup.yml -m yum -a "name=httpd state=absent" --become
```
❓ Document your inventory and base commands 🇺🇸

💡 Inventaire:
- On fournit en variable le chemin vers notre clé ssh, et l'utilisateur.
- On fournit l'adresse de notre serveur 🇫🇷

### Playbooks

Création d'un fichier playbook.yml:

```
touch ansible/playbook.yml
```

Ping:

```
ansible-playbook -i ansible/inventories/setup.yml ansible/playbook.yml
```


### Advanced Playbooks

```
# Install Docker
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

    - name: Install python3
      yum:
        name: python3
        state: present

    - name: Install docker with Python 3
      pip:
        name: docker
        executable: pip3
      vars:
        ansible_python_interpreter: /usr/bin/python3

    - name: Make sure Docker is running
      service: name=docker state=started
      tags: docker
```

### Using Roles 

Créer un rôle:

```
ansible-galaxy init roles/<NOM_DU_ROLE>
```

Roles :
- app
- database
- docker
- network
- proxy

Chaque sous dossier du dossier roles est un rôle & on peut les executer de puis le playbook.yml

Lancer les rôles depuis le `playbook.yml`:

```
  roles:
    - app
    - database
    - docker
    - network
    - proxy
```



Notre `playbook.yml` avec les rôles :


```yml
# Install Docker
- hosts: all
  gather_facts: false
  become: true
  vars_files:
    - group_vars/var.yml
  roles:
    - docker # install docker
    - network # créer un réseau pour les containers
    - database # lance la db
    - app # lance le backend
    - proxy # lance le reverse proxy
```


❓ Document your playbook 🇺🇸

💡 On appel chaque rôle afin qu'il execute les tâches qu'il contient. 🇫🇷

### Deploy your App

Après création des rôles spécifiés, chacun doit exécuté une tâche. Leur ordre est important, il faut donc faire attention. Dans la continuité de nos secrets github, il faut rajouter des variables d'environements pour les **urls**, les **noms des containers**, le **réseau**,et les **tokens**.

Pour cela, on crée un fichier `group_vars\var.yml` :

```
#db
POSTGRES_CONTAINER: postgres-db
POSTGRES_DB: db
POSTGRES_USER: usr
POSTGRES_PASSWORD: pwd

# docker network
DOCKER_NETWORK: app-network

#backend
API_CONTAINER: springboot-app-container
```
On le chiffre ensuite avec `ansible-vault`:

```bash
ansible-vault encrypt ansible/group_vars/var.yml
```

On nous demande de saisir un mot de passe qui nous sera demander lors du déploiement.

Pour déchiffrer:

```yml
ansible-vault decrypt ansible/group_vars/var.yml
```

Les différentes `tasks` executés par les rôles :

- docker :

```yml
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

- name: Install python3
  yum:
    name: python3
    state: present

- name: Install docker with Python 3
  pip:
    name: docker
    executable: pip3
  vars:
    ansible_python_interpreter: /usr/bin/python3

- name: Make sure Docker is running
  service: name=docker state=started
  tags: docker
```
- network :
```yml
- name: Create docker network
  community.docker.docker_network:
    name: "{{ DOCKER_NETWORK }}" 
  vars:
    ansible_python_interpreter: /usr/bin/python3
```
- database :
```yml
- name: Run database
  vars:
    ansible_python_interpreter: /usr/bin/python3
  docker_container:
    name: "{{ POSTGRES_CONTAINER }}"
    image: thomas0507/tp-database:latest
    networks:
      - name: "{{ DOCKER_NETWORK }}"
    env:
      POSTGRES_DB: "{{ POSTGRES_DB }}"
      POSTGRES_USER: "{{ POSTGRES_USER }}"
      POSTGRES_PASSWORD: "{{ POSTGRES_PASSWORD }}"
    volumes:
      - myvolume:/var/lib/postgresql/data
    recreate:
      true
    pull:
      true
```
- app :
```yml
- name: Run Backend API
  vars:
    ansible_python_interpreter: /usr/bin/python3
  docker_container:
    name: "{{ API_CONTAINER }}" 
    image: thomas0507/tp-devops-simple-api:latest
    networks:
      - name: "{{ DOCKER_NETWORK }}" 
    env:
      URL: "{{ POSTGRES_CONTAINER }}:5432" 
      POSTGRES_DB: "{{ POSTGRES_DB }}" 
      POSTGRES_USER: "{{ POSTGRES_USER }}" 
      POSTGRES_PASSWORD: "{{ POSTGRES_PASSWORD }}"
    recreate: true
    pull: true
```
- proxy :
```yml
- name: Run HTTPD
  vars:
    ansible_python_interpreter: /usr/bin/python3
  docker_container:
    name: httpd
    image: thomas0507/my-apache-server:latest
    networks:
      - name: "{{ DOCKER_NETWORK }}" 
    ports:
      - 8080:80
    recreate: true
    pull: true
```

❗ On notera que pour certaines tâches, il a fallu précier l'interpréteur python d'ansible en version 3.


❓ Document your docker_container tasks configuration. 🇺🇸

- 💡 docker : On Installe toutes les dépendances & docker à l'aide de yum.
- 💡 network : On crée un réseau docker avec le nom contenue dans le `var.yml` et en précisant la version de python.
- 💡 database : On préciser la version de python. On récupère l'image docker de notre base de donnée depuis le dockerhub,  et on lui précise son réseau. On passe en variable d'environnement les log superuser. On précise l'emplacement de son volume afin de maintenir une persistance des données. On précise que l'image sera toujours rebuild & pull depuis le repo docker.
- 💡app : Idem qu'au dessus, on récupère l'image docker de dockerhub et on l'instancie avec ses variables d'environnements.
- 💡De même, sauf qu'ici on précise en plus son port exposé. 🇫🇷

❗Étant donné qu'on a chiffré notre `var.yml`, il faut absolument entré le mot de passe de la `vault` lors du déploiement :

```bash
sudo ansible-playbook --ask-vault-pass -i ansible/inventories/setup.yml ansible/playbook.yml
```

### Front and More

⏳Ran out of time ⏳

