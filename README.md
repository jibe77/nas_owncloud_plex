# Mise en place d'un NAS dans un environnement conteneurisé avec Docker-Compose

Ce tutoriel permet de mettre en place un NAS conteneurisé sur Raspberry Pi 5, en se basant sur :
* Docker-Compose,
* Owncloud, 
* Plex, 
* Let's encrypt, 
* et différents outils de gestion des Torrents avec notamment Transmission.

OwnCloud permet de mettre à disposition ses fichiers depuis une interface web et de synchroniser les fichiers depuis une machine desktop (par exemple https://owncloud.com/desktop-app/).
 Flex permet de consulter vos médias depuis une interface web ou une application mobile (https://www.plex.tv/apps-devices/).

Let’s encrypt fournit un certificat dédié au chiffrement des données afin de sécuriser les échanges de données. 

Cette configuration se base sur Traefik comme reverse-proxy. Ce dernier s’intègre simplement avec docker pour exposer l’ensemble des services de façon dynamique. Cela facilite la configuration et le support de HTTPS via l’intégration de Let’s encrypt.

![nas.drawio.png](images/nas.drawio.png)

## Prérequis : 

Un raspberry Pi, idéalement le 5 afin d’avoir suffisamment de puissance notamment pour Plex

Un installation basée sur Debian ou Ubuntu

Un nom de domaine, dans mon cas j’en ai un dynamique fournit par noip.com (https://www.noip.com/) qui permet de lier mon adresse IP dynamique à un nom de domaine personnalisé. Dans ce tutoriel j’utilise le domaine ‘my_domain.ddns.net' mais il faudra le remplacer par le votre. Cet élément est configuré directement au niveau de mon routeur donc je n’ai pas eu besoin de le configuré dans ce tutoriel. Cet élément est un nécessaire pour que « Let’s encrypt » puisse fonctionner.

## Étapes de l'installation : 

Installer Docker et Docker Compose : Assurez-vous que Docker et Docker Compose sont installés sur votre machine. Vous pouvez les installer en suivant les instructions officielles sur le site de Docker. Si ce n’est pas le cas :

Mettre à jour le système :

> sudo apt update && sudo apt upgrade -y

Installer le repo de docker et docker-compose

> sudo apt install apt-transport-https ca-certificates curl software-properties-common

> curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

> echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

> sudo curl -L "https://github.com/docker/compose/releases/download/v2.23.3/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

Installation de docker et docker-compose : 

> sudo apt update

> sudo apt install docker-ce

> sudo apt install -y docker-compose-plugin

Appliquer les permissions d’exécution :

> sudo chmod +x /usr/local/bin/docker-compose

Vérifier que les applications sont installées :

> sudo docker --version

> sudo docker-compose --version

Créer l’utilisateur owncloud, avec lequel on va installer et faire fonctionner l’environnement.

> sudo adduser owncloud

> sudo usermod -aG docker owncloud

Créer les répertoires nécessaires : Avant de lancer les conteneurs, assurez-vous que les répertoires mentionnés dans le fichier docker-compose.yml existent sur votre machine :

	/home/owncloud/portainer/portainerdata 

	/home/owncloud/transmission/config
	/home/owncloud/transmission/watch

	/home/owncloud/prowlarr/config_files
	/home/owncloud/lidarr/lidarr_config
	/home/owncloud/radarr/radarr_config
	/home/owncloud/sonarr/sonarr_config
	/home/owncloud/readarr/readarr_config
	/home/owncloud/bazarr/bazarr_config

	/home/owncloud/plex/plex_config

	/home/owncloud/oc_mysql
	/home/owncloud/oc_redis
	/home/owncloud/oc_files/files/nas/files/my_movies
	/home/owncloud/oc_files/files/nas/files/my_movies
	/home/owncloud/oc_files/files/nas/files/my_music
	/home/owncloud/oc_files/files/nas/files/my_series
	
	/var/tmp/transmission : stockage des fichiers temporaires pour le client Torrent

Le fichier [docker-compose.yml](docker-compose.yml) et [conf/traefik.toml](conf/traefik.toml) ci-joint peuvent être récupérés sur ce repo Git.
Nous allons les décrire pas à pas.

## Configuration de Traefik avec son interface d'administration

Création du répertoire « docker-compose.yml » avec Traefik dans le répertoire /home/owncloud/traefik: 

     version: '3'
     
     services:
       traefik:
         image: traefik:mimolette
         container_name: traefik
         ports:
           - "443:443"
           - "80:80"
           - "8080:8080"
         volumes:
           - /var/run/docker.sock:/var/run/docker.sock
           - /home/owncloud/traefik/config:/etc/traefik
         healthcheck:
           test: ["CMD-SHELL", "wget --no-verbose --tries=1 --spider http://localhost:8080/dashboard || exit 1"]
           interval: 30s
           timeout: 10s
           retries: 3
           start_period: 30s
        labels:
           - "traefik.passHostHeader=true"
  
On spécifie également le fichier de configuration config/traefik.toml contenant en autre les paramètres de let’s encrypt:

     [entryPoints]
      [entryPoints.http]
      address = ":80"
      [entryPoints.https]
      address = ":443"
    
    [api]
      dashboard = true
      insecure = true
    
    [providers.docker]
      endpoint = "unix:///var/run/docker.sock"
    
    [certificatesResolvers.letsencrypt.acme]
      email = "my_email@my_provider.com"
      storage = "/etc/traefik/acme.json"
      [certificatesResolvers.letsencrypt.acme.httpChallenge]
        entryPoint = "http"

Les commandes suivantes permettent de démarrer docker-compose : 

> $ docker-compose pull

> $ docker-compose up -d

On vérifie l’accès à l’interface d’administration :

![Traefix admin page](/images/capture1.png)

## Configuration de Portainer pour administrer l'environnement Docker-Compose

On décrit ensuite Portainer, qui est une interface graphique permettant de surveiller l’état des conteneurs : 

      portainer:
        image: portainer/portainer-ce:latest
        container_name: portainer
        restart: always
        volumes:
          - /var/run/docker.sock:/var/run/docker.sock
          - /home/owncloud/portainer/portainerdata:/data
        depends_on:
          - traefik
        labels:
          - « traefik.http.routers.portainer.rule=Host(`my_domain.ddns.net`) && PathPrefix(`/portainer`)"
          - "traefik.http.routers.portainer.service=portainer-svc"
          - "traefik.http.services.portainer-svc.loadbalancer.server.port=9000"
          - "traefik.http.services.portainer-svc.loadbalancer.server.scheme=http"
          - "traefik.http.routers.portainer.entrypoints=https"
          - "traefik.http.routers.portainer.tls.certresolver=letsencrypt"
          - "traefik.http.routers.portainer.tls=true"
          - "traefik.http.routers.portainer.middlewares=portainer-stripprefix"
          - "traefik.http.middlewares.portainer-stripprefix.stripprefix.prefixes=/portainer"

On remarque les labels qui permettent d’indiquer à Traefkik que le port 9000 est redirigé vers le contexte /portainer lorsqu’on accède depuis mon domaine ‘my_domain.ddns.net' (vous remplacerez ici par votre domaine).

Le label let’s encrypt permet d’activer le chiffrement des données sur les communications vers ce service :

> "traefik.http.routers.portainer.entrypoints=https"
>
> "traefik.http.routers.portainer.tls.certresolver=letsencrypt"
>
> "traefik.http.routers.portainer.tls=true"

On redémarre docker-compose : 

> $ docker-compose restart

On le voit ici dans l’interface d’administration de Traefik :

![Traefix admin page](/images/capture2.png)

Voici une copie d’écran de l’accès à l’interface web de Portainer : 

![Traefix admin page](/images/capture3.png)

## Configuration des outils de gestion des torrents

Transmission est un client Torrent : 

      transmission:
        image: lscr.io/linuxserver/transmission:latest
        container_name: transmission
        environment:
          - PUID=33
          - PGID=33
          - TZ=Europe/Paris
          - USER=my_user
          - PASS=my_password
        volumes:
          - /home/owncloud/transmission/config:/config
          - /home/owncloud/oc_files/files/nas/files:/downloads/complete
          - /var/tmp/transmission:/downloads/incomplete
          - /home/owncloud/transmission/watch:/watch
        ports:
          - 9091:9091
          - 51413:51413
          - 51413:51413/udp
        restart: unless-stopped
        depends_on:
          - traefik
        labels:
          - "traefik.http.routers.transmission.rule=Host(`my_domain.ddns.net`) && PathPrefix(`/transmission`)"
          - "traefik.http.routers.transmission.service=transmission-svc"
          - "traefik.http.services.transmission-svc.loadbalancer.server.port=9091"
          - "traefik.http.services.transmission-svc.loadbalancer.server.scheme=http"
          - "traefik.http.routers.transmission.middlewares=auth"
          - "traefik.http.middlewares.auth.basicauth.users=my_user:my_hashed_password"
          - "traefik.http.routers.transmission.entrypoints=https"
          - "traefik.http.routers.transmission.tls.certresolver=letsencrypt"
          - "traefik.http.routers.transmission.tls=true"

      prowlarr:
        container_name: prowlarr
        image: ghcr.io/hotio/prowlarr
        depends_on:
          - traefik
        ports:
          - 9696:9696
        environment:
          - PUID=33
          - PGID=33
          - UMASK=002
          - TZ=Europe/Paris
        volumes:
          - /home/owncloud/prowlarr/config_files:/config
        healthcheck:
          test: ["CMD", "curl", "-f", "http://localhost:9696"]
          interval: 30s
          timeout: 10s
          retries: 3
          start_period: 60s
        labels:
          - "traefik.http.routers.prowlarr.rule=Host(`my_domain.ddns.net`) && PathPrefix(`/prowlarr`)"
          - "traefik.http.routers.prowlarr.service=prowlarr-svc"
          - "traefik.http.services.prowlarr-svc.loadbalancer.server.port=9696"
          - "traefik.http.services.prowlarr-svc.loadbalancer.server.scheme=http"
          - "traefik.http.routers.prowlarr.entrypoints=https"
          - "traefik.http.routers.prowlarr.tls.certresolver=letsencrypt"
          - "traefik.http.routers.prowlarr.tls=true"
          # Note : il faut spécifier /prowlarr dans le base url des paramètres généraux. 

      lidarr:
        container_name: lidarr
        image: ghcr.io/hotio/lidarr
        depends_on:
          - traefik
        environment:
          - PUID=33
          - PGID=33
          - UMASK=002
          - TZ=Europe/Paris
        volumes:
          - /home/owncloud/lidarr/lidarr_config:/config
          - /home/owncloud/oc_files/files/nas/files/my_music:/data
        ports:
          - 8686:8686
        healthcheck:
          test: ["CMD", "curl", "-f", "http://localhost:8686"]
          interval: 30s
          timeout: 10s
          retries: 3
          start_period: 60s
        labels:
          - « traefik.http.routers.lidarr.rule=Host(`my_domain.ddns.net`) && PathPrefix(`/lidarr`)"
          - "traefik.http.routers.lidarr.service=lidarr-svc"
          - "traefik.http.services.lidarr-svc.loadbalancer.server.port=8686"
          - "traefik.http.services.lidarr-svc.loadbalancer.server.scheme=http"
          - "traefik.http.routers.lidarr.entrypoints=https"
          - "traefik.http.routers.lidarr.tls.certresolver=letsencrypt"
          - "traefik.http.routers.lidarr.tls=true"

      radarr:
        container_name: radarr
        image: ghcr.io/hotio/radarr
        ports:
          - 7878:7878
        depends_on:
          - traefik
        environment:
          - PUID=33
          - PGID=33
          - UMASK=002
          - TZ=Europe/Paris
        volumes:
          - /home/owncloud/radarr/radarr_config:/config
          - /home/owncloud/oc_files/files/nas/files/my_movies:/data
        labels:
          - "traefik.http.routers.radarr.rule=Host(`my_domain.ddns.net`) && PathPrefix(`/radarr`)"
          - "traefik.http.routers.radarr.service=radarr-svc"
          - "traefik.http.services.radarr-svc.loadbalancer.server.port=7878"
          - "traefik.http.services.radarr-svc.loadbalancer.server.scheme=http"
          - "traefik.http.routers.radarr.entrypoints=https"
          - "traefik.http.routers.radarr.tls.certresolver=letsencrypt"
          - "traefik.http.routers.radarr.tls=true"

      sonarr:
        container_name: sonarr
        image: ghcr.io/hotio/sonarr
        depends_on:
          - traefik
        environment:
          - PUID=33
          - PGID=33
          - UMASK=002
          - TZ=Europe/Paris
        volumes:
          - /home/owncloud/sonarr/sonarr_config:/config
          - /home/owncloud/oc_files/files/nas/files/my_movies:/data
        ports:
          - 8989:8989
        healthcheck:
          test: ["CMD", "curl", "-f", "http://localhost:8989"]
          interval: 30s
          timeout: 10s
          retries: 3
          start_period: 60s
        labels:
          - "traefik.http.routers.sonarr.rule=Host(`my_domain.ddns.net`) && PathPrefix(`/sonarr`)"
          - "traefik.http.routers.sonarr.service=sonarr-svc"
          - "traefik.http.services.sonarr-svc.loadbalancer.server.port=8989"
          - "traefik.http.services.sonarr-svc.loadbalancer.server.scheme=http"
          - "traefik.http.routers.sonarr.entrypoints=https"
          - "traefik.http.routers.sonarr.tls.certresolver=letsencrypt"
          - "traefik.http.routers.sonarr.tls=true"

      readarr:
        container_name: readarr
        image: ghcr.io/hotio/readarr
        depends_on:
          - traefik
        environment:
          - PUID=33
          - PGID=33
          - UMASK=002
          - TZ=Europe/Paris
        ports:
          - 8787:8787
        volumes:
          - /home/owncloud/readarr/readarr_config:/config
          - /home/owncloud/oc_files/files/nas/files:/data
        healthcheck:
          test: ["CMD", "curl", "-f", "http://localhost:8787"]
          interval: 30s
          timeout: 10s
          retries: 3
          start_period: 60s
        labels:
          - "traefik.http.routers.readarr.rule=Host(`my_domain.ddns.net`) && PathPrefix(`/readarr`)"
          - "traefik.http.routers.readarr.service=readarr-svc"
          - "traefik.http.services.readarr-svc.loadbalancer.server.port=8787"
          - "traefik.http.services.readarr-svc.loadbalancer.server.scheme=http"
          - "traefik.http.routers.readarr.entrypoints=https"
          - "traefik.http.routers.readarr.tls.certresolver=letsencrypt"
          - "traefik.http.routers.readarr.tls=true"

      bazarr:
        container_name: bazarr
        image: ghcr.io/hotio/bazarr
        depends_on:
          - traefik
        environment:
          - PUID=33
          - PGID=33
          - UMASK=002
          - TZ=Europe/Paris
        volumes:
          - /home/owncloud/bazarr/bazarr_config:/config
          - /home/owncloud/oc_files/files/nas/files/my_movies:/data
        ports:
          - 6767:6767
        healthcheck:
          test: ["CMD", "curl", "-f", "http://localhost:6767"]
          interval: 30s
          timeout: 10s
          retries: 3
          start_period: 30s
        labels:
          - "traefik.http.routers.bazarr.rule=Host(`my_dmain.ddns.net`) && PathPrefix(`/bazarr`)"
          - "traefik.http.routers.bazarr.service=bazarr-svc"
          - "traefik.http.services.bazarr-svc.loadbalancer.server.port=6767"
          - "traefik.http.services.bazarr-svc.loadbalancer.server.scheme=http"
          - "traefik.http.routers.bazarr.entrypoints=https"
          - "traefik.http.routers.bazarr.tls.certresolver=letsencrypt"
          - "traefik.http.routers.bazarr.tls=true"

On voit que les labels Traefik permettent de rediriger le port 9091 vers le préfixe /transmission.

L’accès à Transmission se fait via un identifiant dont le mot de passe est hashé en utilisant l’utilitaire htpasswd :

> $ htpasswd -nbB my_user my_password

Il faut donc spécifier la valeur retournée dans le paramètre « traefik.http.middlewares.auth.basicauth.users ».

Pour plus d’information : https://doc.traefik.io/traefik/middlewares/http/basicauth/ 

Voici une copie d’écran de Transmission : 

![Transmission page](/images/capture4.png)

Certains conteneurs sont configurés pour fonctionner avec l'utilisateur www-data afin de permettre 
le partage de données entre les différents conteneurs, notamment avec Owncloud :  

>      - PUID=33
>
>      - PGID=33

On configure ensuite Owncloud. Celui-ci se base sur mariadb et redis :

      mariadb:
        image: mariadb:lts 
        container_name: owncloud_mariadb
        restart: always
        environment:
          - MYSQL_ROOT_PASSWORD=my_password
          - MYSQL_USER=my_user
          - MYSQL_PASSWORD=my_password
          - MYSQL_DATABASE=owncloud
          - MARIADB_AUTO_UPGRADE=1
        command: ["--max-allowed-packet=128M", "--innodb-log-file-size=64M"]
        depends_on:
          - traefik
        healthcheck:
          test: ["CMD", "mariadb-admin", "ping", "-u", "root", "--password=owncloud"]
          interval: 10s
          timeout: 5s
          retries: 5
        volumes:
          - /home/owncloud/oc_mysql:/var/lib/mysql
        labels:
          - "traefik.enable=false"

      redis:
        image: redis:6
        container_name: owncloud_redis
        restart: always
        command: ["--databases", "1"]
        depends_on:
          - traefik
        healthcheck:
          test: ["CMD", "redis-cli", "ping"]
          interval: 10s
          timeout: 5s
          retries: 5
        volumes:
          - /home/owncloud/oc_redis:/data
        labels:
          - "traefik.enable=false"

On peut ensuite déclarer owncloud, on déclare qu’il y a une dépendance vers ‘mariadb’ et ‘redis’ via le champs depends_on :

      owncloud:
        image: owncloud/server:latest
        container_name: owncloud_server
        restart: always
        depends_on:
          - traefik
          - mariadb
          - redis
        environment:
          - OWNCLOUD_DOMAIN=${OWNCLOUD_DOMAIN}
          - OWNCLOUD_TRUSTED_DOMAINS=${OWNCLOUD_TRUSTED_DOMAINS}
          - OWNCLOUD_DB_TYPE=mysql
          - OWNCLOUD_DB_NAME=owncloud
          - OWNCLOUD_DB_USERNAME=my_user
          - OWNCLOUD_DB_PASSWORD=my_password
          - OWNCLOUD_DB_HOST=mariadb
          - OWNCLOUD_ADMIN_USERNAME=${ADMIN_USERNAME}
          - OWNCLOUD_ADMIN_PASSWORD=${ADMIN_PASSWORD}
          - OWNCLOUD_MYSQL_UTF8MB4=true
          - OWNCLOUD_REDIS_ENABLED=true
          - OWNCLOUD_REDIS_HOST=redis
          - OWNCLOUD_MAIL_DOMAIN=${OWNCLOUD_MAIL_DOMAIN}
          - OWNCLOUD_MAIL_FROM_ADDRESS=${OWNCLOUD_MAIL_FROM_ADDRESS}
          - OWNCLOUD_MAIL_SMTP_MODE=${OWNCLOUD_MAIL_SMTP_MODE}
          - OWNCLOUD_MAIL_SMTP_AUTH=${OWNCLOUD_MAIL_SMTP_AUTH}
          - OWNCLOUD_MAIL_SMTP_AUTH_TYPE=${OWNCLOUD_MAIL_SMTP_AUTH_TYPE}
          - OWNCLOUD_MAIL_SMTP_HOST=${OWNCLOUD_MAIL_SMTP_HOST}
          - OWNCLOUD_MAIL_SMTP_PORT=${OWNCLOUD_MAIL_SMTP_PORT}
          - OWNCLOUD_MAIL_SMTP_NAME=${OWNCLOUD_MAIL_SMTP_NAME}
          - OWNCLOUD_MAIL_SMTP_PASSWORD=${OWNCLOUD_MAIL_SMTP_PASSWORD}
          - OWNCLOUD_MAIL_SMTP_SECURE=${OWNCLOUD_MAIL_SMTP_SECURE}
          - OWNCLOUD_FILELOCKING_ENABLED=true
          - OWNCLOUD_FILESYSTEM_CHECK_CHANGES=1
        healthcheck:
          test: ["CMD", "/usr/bin/healthcheck"]
          interval: 30s
          timeout: 10s
          retries: 5
        volumes:
          - /home/owncloud/oc_files:/mnt/data
        labels:
          - « traefik.http.routers.owncloud.rule=Host(`my_domain.ddns.net`) && PathPrefix(`/owncloud`)"
          - "traefik.http.routers.owncloud.service=owncloud-svc"
          - "traefik.http.services.owncloud-svc.loadbalancer.server.port=8080"
          - "traefik.http.services.owncloud-svc.loadbalancer.server.scheme=http"
          - "traefik.http.routers.owncloud.entrypoints=https"
          - "traefik.http.routers.owncloud.tls.certresolver=letsencrypt"
          - "traefik.http.routers.owncloud.tls=true"
          - "traefik.http.routers.owncloud.middlewares=owncloud-stripprefix"
          - "traefik.http.middlewares.owncloud-stripprefix.stripprefix.prefixes=/owncloud"

Dans le fichier de configuration de Owncloud, il est important de configurer certains paramètres pour accéder à la base de données pour que cela correspond aux paramètres déclaré précédemment dans la configuration du conteneur  : 

      'dbtype' => 'mysql',
      'dbhost' => 'mariadb:3306',
      'dbname' => 'owncloud',
      'dbuser' => ‘my_user',
      'dbpassword' => ‘my_password’,

Il faut spécifier le nom de domaine dans la liste des domaines de confiance :

      'trusted_domains' =>
      array (
        0 => '10.0.0.8',
        1 => ‘my_domain.ddns.net',
      ),

Il faut surcharger l’URL :

>  'overwrite.cli.url' => ‘http://my_domain.ddns.net/',

>  'overwritehost' => 'my_domain.ddns.net',
>
>  'overwriteprotocol' => 'https',
>
>  'overwritewebroot' => '/owncloud',

On peut donc ensuite accéder à Owncloud depuis l’interface web pour y terminer la configuration :

![Owncloud page](/images/capture5.png)

On peut ensuite déclarer Plex : 

      plex:
        container_name: plex
        image: lscr.io/linuxserver/plex:latest
        depends_on:
          - traefik
        ports:
          - "32400:32400"
          - "1900:1900"
          - "5353:5353"
          - "8324:8324"
          - "32410:32410"
          - "32412:32412"
          - "32413:32413"
          - "32414:32414"
          - "32469:32469"
        environment:
          - PUID=33
          - PGID=33
          - TZ=Europe/Paris
          - VERSION=docker
          - PLEX_CLAIM=claim-1234567890
        volumes:
          - /home/owncloud/plex/plex_config:/config
          - /home/owncloud/oc_files/files/nas/files/my_movies:/movies
          - /home/owncloud/oc_files/files/nas/files/my_music:/music
          - /home/owncloud/oc_files/files/nas/files/my_series:/tv_shows
          - /var/tmp:/var/tmp
        restart: unless-stopped
        healthcheck:
          test: ["CMD", "curl", "-f", "http://localhost:32400/web"]
          interval: 30s
          timeout: 10s
          retries: 3
          start_period: 60s
        labels:
          - « traefik.http.routers.plex.rule=Host(`my_domain.ddns.net`) && PathPrefix(`/plex`)"
          - "traefik.http.routers.plex.service=plex-svc"
          - "traefik.http.services.plex-svc.loadbalancer.server.port=32400"
          - "traefik.http.services.plex-svc.loadbalancer.server.scheme=http"
          - "traefik.http.routers.plex.entrypoints=https"
          - "traefik.http.routers.plex.tls.certresolver=letsencrypt"
          - "traefik.http.routers.plex.tls=true"
          - "traefik.http.routers.plex.middlewares=plex-stripprefix"
          - "traefik.http.middlewares.plex-stripprefix.stripprefix.prefixes=/plex"
          - "traefik.http.routers.plex2.rule=Host(`my_domain.ddns.net`) && PathPrefix(`/web`)"
          - "traefik.http.routers.plex2.service=plex2-svc"
          - "traefik.http.services.plex2-svc.loadbalancer.server.port=32400"
          - "traefik.http.services.plex2-svc.loadbalancer.server.scheme=http"
          - "traefik.http.routers.plex2.entrypoints=https"
          - "traefik.http.routers.plex2.tls.certresolver=letsencrypt"
          - "traefik.http.routers.plex2.tls=true"
          - "traefik.http.routers.plex2.middlewares=plex2-stripprefix"
          - "traefik.http.middlewares.plex2-stripprefix.stripprefix.prefixes=/plex"

On peut ensuite consulter le contenu du NAS via le media center :

![Plex page](/images/capture6.png)
  
Plex vient ainsi récupérer les fichiers audio et vidéo qui sont déclarés sur OwnCloud.

Pour terminer on déclare le conteneur « WatchTower ». Ce dernier a pour tâche de mettre à jour tous les conteneurs chaque jour à 4h du matin : 

      watchtower:
        image: containrrr/watchtower
        volumes:
          - /var/run/docker.sock:/var/run/docker.sock
        environment:
          - WATCHTOWER_CLEANUP=true
          - WATCHTOWER_SCHEDULE=0 0 4 * * *
