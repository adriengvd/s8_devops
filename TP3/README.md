# TP3

## Intro

**setup.yml**
```yml
all:
  vars:
    ansible_user: centos
    ansible_ssh_private_key_file: ~/.ssh/id_rsa
  children:
    prod:
      hosts: 
        adrien.gervraud.takima.cloud:
```

Commands used
```ansible
```

## Playbooks

On déplace les tasks du `playbook.yml` vers `tasks/m
ain.yml`, et on appelle le role docker dans le playbook.

**tasks/main.yml:**
```yml
---
# tasks file for roles/docker

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

**playbook.yml:**
```yml
- hosts: all
  gather_facts: false
  become: True
# Install Docker
  roles:
    - docker
```


## Deploy your app

On créé les 5 roles avec les `tasks/main.yml` correspondants.
Le role install_docker a juste été renommé, on utilise le même fichier qu'au dessus.

**create_network/tasks/main.yml:**
```yml
---
# tasks file for roles/launch_proxy
- name: Create proxy container and connect to network
  docker_container:
    name: simple-api-with-db
    image: "adriengvd/tp-devops-cpe:simple-api"
    networks:
      - name: "network_tp3"
```

**launch_database/tasks/.yml:**
```yml
---
# tasks file for roles/launch_database

- name: Create db container and connect to network
  docker_container:
    name: "postgresdb"
    image: "adriengvd/tp-devops-cpe:database"
    networks:
      - name: "network_tp3"
    volumes:
        - ./database/data
```

**launch_app/tasks/main.yml:**
```yml
---
# tasks file for roles/launch_proxy
- name: Create proxy container and connect to network
  docker_container:
    name: simple-api-with-db
    image: "adriengvd/tp-devops-cpe:simple-api"
    networks:
      - name: "network_tp3"
```

**launch_proxy/tasks/main.yml:**
```yml
---
# tasks file for roles/launch_proxy
- name: Create proxy container and connect to network
  docker_container:
    name: http_server
    image: "adriengvd/tp-devops-cpe:http-server"
    networks:
      - name: "network_tp3"
    ports:
      - '80:80'
```

On appelle ensuite tous ces rôles dans le bon ordre dans le fichier `playbook.yml`

**playbook.yml:**
```yml
- hosts: all
  gather_facts: false
  become: True
# Install Docker
  roles:
    - install_docker
    - create_network
    - launch_database
    - launch_app
    - launch_proxy
```


## Front

Pour ajouter le front à notre serveur, il faut d'abord mettre à jout notre workflow `build_and_push_images`, pour y rajouter l'image du front, que l'on pourra `pull` depuis le serveur.
rajouter un rôle launch_front:

**launch_front/tasks/main.yml:**
```yml
---
# tasks file for roles/launch_front
- name: Create front container and connect to network
  docker_container:
    name: front
    image: "adriengvd/tp-devops-cpe:front"
    networks:
      - name: "network_tp3"
```

Il ne faut pas oublier de l'ajouter à notre `playbook.yml` :
```yml
( ... )
    - launch_front
```


## Continuous deployment 


On créée un nouveau workflow `deploy.yml` avec le contenu suivant:
```yml
name: Run ansible playbook
on:
  #the job is launched from the build_and_push_images workflow
  workflow_call:
    secrets:
      SSH_KEY:
        required: true

jobs:
  run-playbook:

    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2

    - name: Run playbook
      uses: dawidd6/action-ansible-playbook@v2
      with:
        # Required, playbook filepath
        playbook: ./ansible/playbook.yml
        # Optional, SSH private key
        key: ${{secrets.SSH_KEY}}
        # Optional, literal inventory file contents
        inventory: |
          all:
            vars:
              ansible_user: centos
            children:
              prod:
                hosts: 
                  adrien.gervraud.takima.cloud:
```

Ce workflow est appelé depuis test_backend.yml. On le modifie pour appeler `build_and_push_images` et `deploy.yml` de la manière suivante:

```yml
  call-build-and-push-images:
    needs: test-backend
    uses: adriengvd/s8_devops/.github/workflows/build_and_push_images.yml@main 
    secrets:
      DOCKERHUB_USERNAME: ${{secrets.DOCKERHUB_USERNAME}}
      DOCKERHUB_TOKEN: ${{secrets.DOCKERHUB_TOKEN}}

  call-deploy:
    needs: call-build-and-push-images
    uses: adriengvd/s8_devops/.github/workflows/deploy.yml@main 
    secrets:
      SSH_KEY: ${{secrets.SSH_KEY}}
 ```

Il faut bien penser à rajouter SSH_KEY à nos secrets GitHub.