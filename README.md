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
# 1-3 Http Server



