# TP2

## First steps into the CI world  



**Main.yml:**
```yml
name: CI devops 2022 CPE
on:
  #to begin you want to launch this job in main and develop
  push:
    branches: 
      - main
      - develop
  pull_request:

jobs:
  test-backend:
    runs-on: ubuntu-18.04
    steps:
      #checkout your github code using actions/checkout@v2.3.3
      - uses: actions/checkout@v2.3.3
      
      #do the same with another action (actions/setup-java@v2) that enable to setup jdk 11
      - name: Set up JDK 11
        uses: actions/setup-java@v2
        with:
          distribution: 'adopt' # See 'Supported distributions' for available options
          java-version: '11'

      - name: Build and test with Maven
        run: mvn clean verify --file ./java/simple-api-with-db/pom.xml 
```

## First steps into the CD world  

On déclare les variables `DOCKERHUB_USERNAME`, `DOCKERHUB_PASSWORD` et `DOCKERHUB_TOKEN` dans la partie secrets de GitHub.


La ligne `needs: test-backend` permet de s'assurer que le job test-backend s'est bien exécuté avant de passer à la prochaine étape. Ainsi, on garantit que les deux jobs ne s'exécutent pas en même temps est que l'on push des images seulement si elles sont valides.

On push les images Docker afin de pouvoir les récupérer, étant donné qu'elles sont générées lors du push.

**main.yml**
```yml
name: CI devops 2022 CPE
on:
  #to begin you want to launch this job in main and develop
  push:
    branches: 
      - main
      - develop
  pull_request:

jobs:
  test-backend:
    runs-on: ubuntu-18.04
    steps:
      #checkout your github code using actions/checkout@v2.3.3
      - uses: actions/checkout@v2.3.3
      
      #do the same with another action (actions/setup-java@v2) that enable to setup jdk 11
      - name: Set up JDK 11
        uses: actions/setup-java@v2
        with:
          distribution: 'adopt' # See 'Supported distributions' for available options
          java-version: '11'

      #finally build your app with the latest command
      - name: Build and test with Maven
        run: mvn clean verify --file ./java/simple-api-with-db/pom.xml 

  # define job to build and publish docker image
  build-and-push-docker-image:
    needs: test-backend
    # run only when code is compiling and tests are passing
    runs-on: ubuntu-latest

    # steps to perform in job
    steps:
    - name: Login to DockerHub
      run: docker login -u ${{secrets.DOCKERHUB_USERNAME}} -p ${{secrets.DOCKERHUB_TOKEN}}

    - name: Checkout code
      uses: actions/checkout@v2

    - name: Build image and push backend
      uses: docker/build-push-action@v2
      with: 
      # relative path to the place where source code with  Dockerfile is located
        context: ./java/simple-api-with-db
        # Note: tags has to be all lower-case
        tags: ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-cpe:simple-api
        # build on feature branches, push only on main branch
        push: ${{github.ref == 'refs/heads/main'}}



    - name: Build image and push database
    # DO the same for database
      uses: docker/build-push-action@v2
      with: 
      # relative path to the place where source code with  Dockerfile is located
        context: ./database
        # Note: tags has to be all lower-case
        tags: ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-cpe:database
        push: ${{github.ref == 'refs/heads/main'}}

    - name: Build image and push httpd
    # DO the same for httpd
      uses: docker/build-push-action@v2
      with: 
      # relative path to the place where source code with  Dockerfile is located
        context: ./http
        # Note: tags has to be all lower-case
        tags: ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-cpe:http-server
        push: ${{github.ref == 'refs/heads/main'}}
```

## Setup Quality Gate

Pour rajouter l'analyse avec Sonar, après s'être créé un compte et avoir rajouté SONAR_TOKEN aux secrets GitHub, on replace les lignes suivantes:

```yml
- name: Build and test with Maven
run: mvn clean verify --file ./java/simple-api-with-db/pom.xml 
```
par celles-ci:
```yml
- name: Analyse with sonar
env: 
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
run: mvn -B verify sonar:sonar -Dsonar.projectKey=adriengvd_s8_devops -Dsonar.organization=adriengvd -Dsonar.host.url=https://sonarcloud.io --file ./java/simple-api-with-db/pom.xml
```

## Split Pipeline


On sépare `main.yml` en deux fichiers. `test_backend.yml` effectue les tests sur les branches `main` et `develop`, et s'il est valide, le job `call-workflow` est exécuté. Ce job appelle le second fichier workflow, `build_and_push_images`, et doit lui passer les secrets utilisés.
On a également ajouté la ligne `cache: 'maven'` dans le job `Set up JDK 11` pour faire gagner du temps.

`build_and_push_images` reprend le code déjà exécuté, mais remplace la condition `on`, pour y mettre `workflow_call`, ainsi que les paramètres requis pour récupérer les secrets.






**test_backend.yml:**
```yml
name: Test backend
on:
  #to begin you want to launch this job in main and develop
  push:
    branches: ['main', 'develop']

jobs:
  test-backend:
    runs-on: ubuntu-18.04
    env:
      working-directory: ./java/simple-api-with-db/
    steps:
      #checkout your github code using actions/checkout@v2.3.3
      - uses: actions/checkout@v2.3.3
      
      #do the same with another action (actions/setup-java@v2) that enable to setup jdk 11
      - name: Set up JDK 11
        uses: actions/setup-java@v2
        with:
          distribution: 'adopt' # See 'Supported distributions' for available options
          java-version: '11'
          cache: 'maven'

      #finally build your app with the latest command
      # - name: Build and test with Maven
      #   run: mvn clean verify --file ./java/simple-api-with-db/pom.xml 

      - name: Analyse with sonar
        env: 
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: mvn -B verify sonar:sonar -Dsonar.projectKey=adriengvd_s8_devops -Dsonar.organization=adriengvd -Dsonar.host.url=https://sonarcloud.io --file ./java/simple-api-with-db/pom.xml
 
  call-workflow:
    needs: test-backend
    uses: adriengvd/s8_devops/.github/workflows/build_and_push_images.yml@main 
    secrets:
      DOCKERHUB_USERNAME: ${{secrets.DOCKERHUB_USERNAME}}
      DOCKERHUB_TOKEN: ${{secrets.DOCKERHUB_TOKEN}}
```

**build_and_push_images.yml:**
```yml
name: Build and push images
on:
  #the job is launched from the test_backend workflow
  workflow_call:
    secrets:
      DOCKERHUB_USERNAME:
        required: true
      DOCKERHUB_TOKEN:
        required: true
    
jobs:
  build-and-push-database:
    # run only when code is compiling and tests are passing
    runs-on: ubuntu-latest

    # steps to perform in job
    steps:
    - name: Login to DockerHub
      run: docker login -u ${{secrets.DOCKERHUB_USERNAME}} -p ${{secrets.DOCKERHUB_TOKEN}}

    - name: Checkout code
      uses: actions/checkout@v2

    - name: Build image and push database
    # DO the same for database
      uses: docker/build-push-action@v2
      with: 
      # relative path to the place where source code with  Dockerfile is located
        context: ./database
        # Note: tags has to be all lower-case
        tags: ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-cpe:database
        push: ${{github.ref == 'refs/heads/main'}}

  build-and-push-http:
    # run only when code is compiling and tests are passing
    runs-on: ubuntu-latest

    # steps to perform in job
    steps:
    - name: Login to DockerHub
      run: docker login -u ${{secrets.DOCKERHUB_USERNAME}} -p ${{secrets.DOCKERHUB_TOKEN}}

    - name: Checkout code
      uses: actions/checkout@v2

    - name: Build image and push httpd
    # DO the same for httpd
      uses: docker/build-push-action@v2
      with: 
      # relative path to the place where source code with  Dockerfile is located
        context: ./http
        # Note: tags has to be all lower-case
        tags: ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-cpe:http-server
        push: ${{github.ref == 'refs/heads/main'}}


  build-and-push-java:
    # run only when code is compiling and tests are passing
    runs-on: ubuntu-latest

    # steps to perform in job
    steps:
    - name: Login to DockerHub
      run: docker login -u ${{secrets.DOCKERHUB_USERNAME}} -p ${{secrets.DOCKERHUB_TOKEN}}

    - name: Checkout code
      uses: actions/checkout@v2

    - name: Build image and push backend
      uses: docker/build-push-action@v2
      with: 
      # relative path to the place where source code with  Dockerfile is located
        context: ./java/simple-api-with-db
        # Note: tags has to be all lower-case
        tags: ${{secrets.DOCKERHUB_USERNAME}}/tp-devops-cpe:simple-api
        # build on feature branches, push only on main branch
        push: ${{github.ref == 'refs/heads/main'}}

```

