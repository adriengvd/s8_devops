# TP3

## Load balancing

Pour mettre en place le load balancing, il faut modifier le fichier `httpd.conf` de la manière suivante:

Décommenter ces imports:
```xml
LoadModule proxy_balancer_module modules/mod_proxy_balancer.so
LoadModule slotmem_shm_module modules/mod_slotmem_shm.so
LoadModule lbmethod_byrequests_module modules/mod_lbmethod_byrequests.so
```

et rajouter les lignes suivantes:
```xml
<Proxy "balancer://myAPI">
    BalancerMember "http://simple-api-1:80"
    BalancerMember "http://simple-api-2:80"
</Proxy>


<VirtualHost *:80>
    ProxyPreserveHost On
    ProxyPass        "/api" "balancer://myAPI"
    ProxyPassReverse "/api" "balancer://myAPI"
    ProxyPass / http://front/
    ProxyPassReverse / http://front/
</VirtualHost>
```

On modifie également le fichier `launch_app/tasks/main.yml` pour y rajouter le second serveur:

```yml
- name: Create app container and connect to network
  docker_container:
    name: simple-api-with-db
    image: "adriengvd/tp-devops-cpe:simple-api"
    networks:
      - name: "network_tp3"
      
- name: Create app container and connect to network
  docker_container:
    name: simple-api-2
    image: "adriengvd/tp-devops-cpe:simple-api"
    networks:
      - name: "network_tp3"
```