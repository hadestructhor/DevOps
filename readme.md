# TP01

## Database

### Point de départ
Voici le Dockerfile au départ:
```docker
FROM postgres:11.6-alpine
ENV POSTGRES_DB=db \
POSTGRES_USER=usr \
POSTGRES_PASSWORD=pwd
```

Voici la commande pour build l'image de la db: 
```cmd
docker build -t hadestructhor/db .
```

Voici la commande pour lancer le container sur le port 5432:
```cmd
docker run -p 5432:5432 --name db hadestructhor/db
```

### Why should we run the container with a flag -e to give the environment variables ?
C'est une best practice qui permets de n'avoir qu'un Dockerfile par application et de pouvoir lancer plusieurs instances avec des variables d'environnements lors du lancement du container au lieu de les avoir de manière fixe dans l'image.
Cela permet également de ne pas marquer en dure les mots de passe de prod si l'on fait du versionnage.

### Lancer la bd avec des variables d'environnement
Le Dockerfile a été modifié comme tel:
```docker
FROM postgres:11.6-alpine
```

Un network a été créée pour permettre à d'autres conteneurs d'acceder à la db:
```cmd
docker network create tp-1-network-db
```

Un adminer a été pull puis run en fond en lançant les deux commandes suivantes:
```cmd
docker pull adminer
docker run -d -p 8080:8080 --name adminer --network=tp-1-network-db adminer
```

La commande pour lancer l'image avec les variables d'environnements est la suivante:
```cmd
docker run -p 5432:5432 --name db --network=tp-1-network-db -e POSTGRES_DB="db" -e POSTGRES_USER="usr" -e POSTGRES_PASSWORD="pwd" hadestructhor/db
```

Adminer et la bd postgres ont été lancé et mis dans le même network.

### persistance des données
Un dossier data a été créée dans le même dossier contenant tout ce qui à un lien avec la bd.
Pour persister les données, la commande docker run a été modifié comme tel:
```cmd
docker run -d --name db --network=tp-1-network-db -e POSTGRES_DB="db" -e POSTGRES_USER="usr" -e POSTGRES_PASSWORD="pwd" -v /c/Users/LENOVO/Desktop/docker/TP0DevOps1/db/data:/var/lib/postgresql/data hadestructhor/db
```

## Backend

### Basics

Voici le Dockerfile au départ:
```docker
FROM openjdk:11
COPY src /src
RUN javac /src/Main.java
CMD java /src/Main.java
```

Voici la commande pour build l'image:
```cmd
docker build -t hadestructhor/backend .
```

Voici la commande pour lancer un conteneur de l'image:
```cmd
docker run --name backend hadestructhor/backend
```

### Multistage build

Voici le dockerfile:
```docker
# Build
FROM maven:3.6.3-jdk-11 AS myapp-build
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY pom.xml .
COPY src ./src
RUN mvn dependency:go-offline 
RUN mvn package -DskipTests
# Run
FROM openjdk:11-jre
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar
ENTRYPOINT java -jar myapp.jar
```

Voici la commande pour le lancer:
```cmd
docker run --name db -d -p 8080:8080 hadestructhor/db
```

### Backend API

Voici le dockerfile pour build le backend:
```docker
# Build
FROM maven:3.6.3-jdk-11 AS myapp-build
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY pom.xml .
COPY src ./src
RUN mvn dependency:go-offline 
RUN mvn package -DskipTests
# Run
FROM openjdk:11-jre
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar
ENTRYPOINT java -jar myapp.jar
```

Voici le contenu de la configuration de l'application:
```properties
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
    url: jdbc:postgresql://db:5432/db
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

Voici la commande pour lancer le backend dans le même network que la bd:
```cmd
docker run --name backend -d -p 8080:8080 --network=tp-1-network-db hadestructhor/backend
```

## Server

### Basics
J'ai pull une image de https comme ceci:
```cmd
docker pull httpd
```

J'ai copié la configuration de base de httpd comme suit: 
```cmd
docker run --rm httpd cat /usr/local/apache2/conf/httpd.conf > httpd.conf
```

Pour le dockerfile correspondant, le voici:
```docker
FROM httpd:2.4
COPY ./public/ /usr/local/apache2/htdocs/
COPY httpd.conf /usr/local/apache2/conf/httpd.conf
```

Pour lancer le build de l'image:
```cmd
docker build -t hadestructhor/server .
```

Pour lancer un container:
```cmd
docker run -d --name server -p 80:80 hadestructhor/server
```

### Reverse Proxy

Voici la config rajouté à httpd.conf:
```xml
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so

<VirtualHost *:80>
    ProxyPreserveHost On
    ProxyPass / http://db:8080/
    ProxyPassReverse / http://db:8080/
</VirtualHost>
```

Voici la commande pour run le container après avoir rebuild l'image:
```cmd
docker run -d --name server --network=tp-1-network-db -p 80:80 hadestructhor/server
```

### Docker-compose

J'ai créée les deux networks suivants pour séparer les accès à la base de données:
```cmd
docker network create backend
docker network create frontend
```

Voici le docker-compose:
```properties
version: '3.7'
services:
  backend:
    container_name: backend
    build:
      ./backend/simple-api2
    networks:
      - backend
      - frontend
    depends_on:
      - db
  db:
    container_name: db
    volumes:
      - /c/Users/LENOVO/Desktop/docker/DevOps/db/data:/var/lib/postgresql/data
    environment:
      - POSTGRES_DB=db
      - POSTGRES_USER=usr
      - POSTGRES_PASSWORD=pwd
    build:
      ./db
    networks:
      - backend
  httpd:
    container_name: server
    build:
      ./server
    ports:
      - 80:80
    networks:
      - frontend
    depends_on:
      - backend

networks:
  backend:
  frontend:
```

Dans cette configuration :
* L'API peut accèder à la base de données
* Le server web peut accèder seulement à l'API et non à la base de données

Pour build les images puis lancer tous les services:
```cmd
docker-compose up --build
```

# TP03

## Intro
### Inventories
```yaml
all:
  vars:
    ansible_user: centos
    ansible_ssh_private_key_file: ~/.ssh/id_rsa_devops
  children:
    prod:
      hosts: angelo.al-yacoub.takima.cloud
```

commande:
`ansible all -i inventories/setup.yml -m ping`

### Facts
commande:
`ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"`
resultat: 
```cmd
angelo.al-yacoub.takima.cloud | SUCCESS => {
    "ansible_facts": {
        "ansible_distribution": "CentOS",
        "ansible_distribution_file_parsed": true,
        "ansible_distribution_file_path": "/etc/centos-release",
        "ansible_distribution_file_variety": "CentOS",
        "ansible_distribution_major_version": "8",
        "ansible_distribution_release": "Stream",
        "ansible_distribution_version": "8",
        "discovered_interpreter_python": "/usr/libexec/platform-python"
    },
    "changed": false
}
```
----------------------------
commande:
`ansible all -i inventories/setup.yml -m yum -a "name=httpd state=absent" --become`
resultat:
```output
angelo.al-yacoub.takima.cloud | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/libexec/platform-python"
    },
    "changed": true,
    "msg": "",
    "rc": 0,
    "results": [
        "Removed: mod_http2-1.15.7-3.module_el8.4.0+778+c970deab.x86_64",
        "Removed: httpd-2.4.37-43.module_el8.5.0+1022+b541f3b1.x86_64"
    ]
}
```

## Playbook
### First playbook
```yaml
- hosts: all
  gather_facts: false
  become: yes
  tasks:
    - name: Test connection
      ping:
```

commande:
`ansible-playbook -i inventories/setup.yml playbook.yml`

resultat:
```
PLAY [all] *****************************************************************************************************************************************

TASK [Test connection] *****************************************************************************************************************************
ok: [angelo.al-yacoub.takima.cloud]

PLAY RECAP *****************************************************************************************************************************************
angelo.al-yacoub.takima.cloud : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

### Advanced playbook
install_docker.yml:
```yml
- hosts: all
  gather_facts: false
  become: yes

  # Install Docker
  tasks:
    - name: Clean packages
      command:
        cmd: dnf clean -y packages
    - name: Install device-mapper-persistent-data
      dnf:
        name: device-mapper-persistent-data
        state: latest
    - name: Install lvm2
      dnf:
        name: lvm2
        state: latest
    - name: add repo docker
      command:
        cmd: sudo dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
    - name: Install Docker
      dnf:
        name: docker-ce
        state: present
    - name: install python3
      dnf:
        name: python3
    - name: Pip install
      pip:
        name: docker
    - name: Make sure Docker is running
      service: name=docker state=started
      tags: docker
```

command:
`ansible-playbook -i inventories/setup.yml install_docker.yml`

result:
```
PLAY [all] *****************************************************************************************************************************************

TASK [Clean packages] ******************************************************************************************************************************
changed: [angelo.al-yacoub.takima.cloud]

TASK [Install device-mapper-persistent-data] *******************************************************************************************************
changed: [angelo.al-yacoub.takima.cloud]

TASK [Install lvm2] ********************************************************************************************************************************
changed: [angelo.al-yacoub.takima.cloud]

TASK [add repo docker] *****************************************************************************************************************************
changed: [angelo.al-yacoub.takima.cloud]

TASK [Install Docker] ******************************************************************************************************************************
changed: [angelo.al-yacoub.takima.cloud]

TASK [install python3] *****************************************************************************************************************************
changed: [angelo.al-yacoub.takima.cloud]

TASK [Pip install] *********************************************************************************************************************************
changed: [angelo.al-yacoub.takima.cloud]

TASK [Make sure Docker is running] *****************************************************************************************************************
changed: [angelo.al-yacoub.takima.cloud]

PLAY RECAP *****************************************************************************************************************************************
angelo.al-yacoub.takima.cloud : ok=8    changed=8    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

### using role

Tout ce qu'il y avait dans tasks du fichier install_docker.yml à été déplacé dans le fichier roles/docker/main.yml.
Voici le contenu du fichier instaall_docker.yml:
```yml
- hosts: all
  gather_facts: false
  become: yes

  roles:
    - docker
```

Le contenu de la task main.yml du role docker est le suivant:

```yml
- name: Clean packages
  command:
    cmd: dnf clean -y packages
- name: Install device-mapper-persistent-data
  dnf:
    name: device-mapper-persistent-data
    state: latest
- name: Install lvm2
  dnf:
    name: lvm2
    state: latest
- name: add repo docker
  command:
    cmd: sudo dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
- name: Install Docker
  dnf:
    name: docker-ce
    state: present
- name: install python3
  dnf:
    name: python3
- name: Pip install
  pip:
    name: docker
- name: Make sure Docker is running
  service: name=docker state=started
  tags: docker
```

En lancant la commande suivante :
`ansible-playbook -i inventories/setup.yml install_docker.yml`
On obtient le même résultat qu'au départ.

## Deploy your app

Voici le contenu de install_docker.yml qui sert à installer docker sur le serveur puis de récupérer et lancer chaque image :

```yml
- hosts: all
  gather_facts: false
  become: yes

  roles:
    - docker
    - network
    - database
    - app
    - front
    - proxy
```

Le contenu de la task main.yml du role docker n'a pas changé.

Le contenu de la task main.yml du role database est le suivant:

```yml
- name: Create network backend
  docker_network:
    name: backend

- name: Create network frontend
  docker_network:
    name: frontend
```

Le contenu de la task main.yml du role network est le suivant:

```yml
- name: Run Database
  docker_container:
    name: db
    image: hadestructhor/db
    state: started
    env:
      POSTGRES_DB: db
      POSTGRES_USER: usr
      POSTGRES_PASSWORD: pwd
    networks:
      - name: backend
```

Le contenu de la task main.yml du role network est le suivant:

```yml
- name: Run Backend
  docker_container:
    name: backend
    image: hadestructhor/backend
    state: started
    exposed_ports:
      - 8080
    networks:
      - name: backend
      - name: frontend
```

Le contenu de la task main.yml du role front est le suivant:

```yml
- name: Run Frontend
  docker_container:
    name: frontend
    image: hadestructhor/front
    state: started
    networks:
      - name: frontend
```

Le contenu de la task main.yml du role network est le suivant:

```yml
- name: Run HTTPD
  docker_container:
    name: httpd
    image: hadestructhor/server
    ports: 80:80
    state: started
    networks:
      - name: frontend
```