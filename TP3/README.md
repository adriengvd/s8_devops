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

**create_network.yml:**
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

**launch_database.yml:**
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

**launch_app.yml:**
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

**launch_proxy.yml:**
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