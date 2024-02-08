# DevOps - Compte rendu 

## TP 01 - Docker 🚀🚀🚀

### Partie 1 🛫✈️🛸

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

## Connexion à une base de donnée PostGres

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

❓ Why do we need a volume to be attached to our postgres container?

💡 Il faut un volume attaché au container afin de persister la donnée. Si on détruit le container et qu'il n'a pas de volume attaché, la base de donnée est réinitialisée à son prochain lancement.

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

❓ Why do we need a multistage build? And explain each step of this dockerfile.

💡 Un build multistage est un build qui utilise plusieurs fois l'instruction `FROM`. Chaque `FROM` constitue une étape du build. Cela nous permet d'utiliser différentes bases.

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

❓ Why do we need a reverse proxy?

💡 Pour ne pas exposer directement le port de notre container. On doit forcément passer par notre front.

## Link application

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

❓ Why is docker-compose so important?

💡 Car cela permet de lancer & de configurer plusieur docker de manière automatique.

## Publish

*TODO*



# DevOps - Compte rendu 

## TP 02 - Github Action 🚀🚀🚀


### Build and test your Application

❓ What are testcontainers?

💡 x

❓ Document your Github Actions configurations

💡 x


### Setup Quality Gate


###     Bonus: split pipelines (Optional)


## TP 03 - Ansible

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
❓ Document your inventory and base commands

💡 Inventaire:
- On fournit en variable le chemin vers notre clé ssh, et l'utilisateur.
- On fournit l'adresse de notre serveur

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

❓ Document your playbook

💡 Chaque task s'occupe de lancer un yum install avec les paramètres fournis.

Créer un rôle:

```
ansible-galaxy init roles/<NOM_DU_ROLE>
```

Chaque sous dossier du dossier roles est un rôle & on peut les executer de puis le playbook.yml

Roles :
- app
- database
- docker
- network
- proxy

Lancer les rôles depuis le playbook.yml:

```
  roles:
    - app
    - database
    - docker
    - network
    - proxy
```

