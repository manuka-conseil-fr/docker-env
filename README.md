# docker-env
fichier docker compose composé de 3 éléments:

### Partie réseau ###
    - Un réseau public permettant l'acces au container VPN
    - UN réseau privé containant les containers PHP et MYSQL



Pour réaliser cela, nous devons configurer les réseaux de manière à ce que l'interface graphique de WireGuard soit uniquement accessible depuis le VPN. Voici un fichier docker-compose.yml qui configure un environnement avec trois conteneurs : un pour Apache et PHP 8.2, un pour PostgreSQL 14, et un pour WireGuard VPN avec une interface graphique. L'interface graphique de WireGuard et Apache-PHP seront accessibles uniquement via le VPN.

yaml
Copier le code
version: '3.8'

services:
  wireguard:
    image: linuxserver/wireguard
    container_name: wireguard
    cap_add:
      - NET_ADMIN
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Paris
      - SERVERURL=yourserverurl
      - SERVERPORT=51820
      - PEERS=1
      - PEERDNS=auto
      - INTERNAL_SUBNET=10.13.13.0/24
    volumes:
      - ./wireguard/config:/config
      - ./wireguard/lib/modules:/lib/modules
    ports:
      - 51820:51820/udp
    restart: unless-stopped
    networks:
      - internal

  wireguard-ui:
    image: cshum/wireguard-ui
    container_name: wireguard-ui
    environment:
      - WGUI_USERNAME=admin
      - WGUI_PASSWORD=admin
      - WGUI_ADDRESS=0.0.0.0
      - WGUI_PORT=5000
    volumes:
      - ./wireguard-ui/data:/data
    restart: unless-stopped
    depends_on:
      - wireguard
    networks:
      - internal

  postgres:
    image: postgres:14
    container_name: postgres
    environment:
      POSTGRES_DB: mydatabase
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypassword
    volumes:
      - ./postgres/data:/var/lib/postgresql/data
    networks:
      - internal

  apache-php:
    image: php:8.2-apache
    container_name: apache-php
    depends_on:
      - postgres
    environment:
      - POSTGRES_DB=mydatabase
      - POSTGRES_USER=myuser
      - POSTGRES_PASSWORD=mypassword
      - POSTGRES_HOST=postgres
    volumes:
      - ./www:/var/www/html
    networks:
      - internal
    extra_hosts:
      - "host.docker.internal:host-gateway"

networks:
  internal:
    internal: true
Explications des configurations :
WireGuard : Conteneur VPN configuré avec l'image linuxserver/wireguard. Il expose le port 51820 en UDP pour les connexions VPN.
WireGuard-UI : Interface graphique pour WireGuard. Elle est accessible sur le réseau interne et non exposée publiquement.
PostgreSQL : Conteneur de base de données PostgreSQL 14. Les données sont stockées dans le volume ./postgres/data.
Apache-PHP : Conteneur Apache avec PHP 8.2. Le code source du site web est monté depuis le répertoire local ./www.
Réseaux :
Internal Network : Réseau interne pour permettre aux conteneurs de communiquer entre eux sans être exposés publiquement.
Instructions supplémentaires :
Configuration de WireGuard : Assurez-vous que les configurations de WireGuard (fichiers de configuration) sont correctement placées dans le répertoire ./wireguard/config.
Accès à WireGuard-UI et Apache-PHP : Pour accéder à l'interface graphique de WireGuard et au serveur Apache-PHP, vous devez d'abord vous connecter au VPN WireGuard. Une fois connecté au VPN, vous pouvez accéder à l'interface graphique via http://10.13.13.x:5000 (où 10.13.13.x est l'adresse IP interne assignée par le VPN) et à Apache-PHP via http://10.13.13.x:80.
Cela garantit que l'accès à l'interface graphique de WireGuard et au serveur Apache-PHP est sécurisé et limité aux utilisateurs connectés via le VPN.
