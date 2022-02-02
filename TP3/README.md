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
