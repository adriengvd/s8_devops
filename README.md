# s8_devops


# 1-1 Database

Il faut utiliser l'option `-e` pour préciser les variables d'environnements comme le mot de passe car ce sont des données sensibles. C'est donc plus sécurité que de les mettre en clair dans le Dockerfile.

Il est nécessaire d'attacher un volume à notre container afin de faire persister les données. Sinon, la base de données se réinitialiserait à chaque fois que le container redémarre.

Dockerfile utilisé: 

```Dockerfile
FROM postgres:11.6-alpine

ENV POSTGRES_DB=db \
    POSTGRES_USER=usr \
    POSTGRES_PASSWORD=pwd

COPY init_db/*.sql /docker-entrypoint-initdb.d/
```

Commandes utilisées depuis ```s8_devops/database```:

```docker
docker build -t adriengvd/postgres_db .

docker run -p 5432:5432 -v /home/agervrau/s8_devops/database/data:/var/lib/postgresql/data  -d adriengvd/postgres_db
```

# 1-2 Backend API

Dockerfile app java première étape : 

```Dockerfile
FROM openjdk:11-jre
COPY . /usr/src/myapp
WORKDIR /usr/src/myapp
# Run java code with the JRE
CMD ["java", "Main"]
```

App java deuxième étape:
```Dockerfile
FROM openjdk:11
COPY . /usr/src/myapp
WORKDIR /usr/src/myapp
RUN javac Main.java
FROM openjdk:11-jre
COPY --from=0 /usr/src/myapp/Main.class .
# Run java code with the JRE
CMD ["java", "Main"]
```

Simple API:

On utilise un multistage build afin de créer des images de plus petite taille. 

```Dockerfile
# Build
FROM maven:3.6.3-jdk-11 AS myapp-build
# ENV est une variable d'environnement
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
# On copie le fichier de dépendences du projet + les sources
COPY pom.xml .
COPY src ./src
# Compile le projet
RUN mvn package -DskipTests

# Run
FROM openjdk:11-jre
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
# On copie le fichier .jar créé à l'étape précedente et on l'exécute
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar
ENTRYPOINT java -jar myapp.jar
```

Simple Api avec base de données :

```Dockerfile
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

Appliction.yaml ->

```yaml
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
    url: jdbc:postgresql://postgresdb:5432/db
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

Commandes pour run les conatainers -> 
```docker
docker build -t adriengvd/simple-api-main .
docker run -p 8080:8080 --net network_tp01 adriengvd/simple-api-main
docker run -p 5432:5432 -v /home/agervrau/s8_devops/database/data:/var/lib/postgresql/data --net network_tp01 --name postgresdb -d adriengvd/postgres_db
```

# 1-3 Http Server



