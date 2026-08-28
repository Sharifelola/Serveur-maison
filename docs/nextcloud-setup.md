# Configuration — Nextcloud

## Fichier Docker Compose

```yaml
services:
  db:
    image: mariadb:10.6
    restart: always
    command: --transaction-isolation=READ-COMMITTED --log-bin=binlog --binlog-format=ROW
    volumes:
      - /srv/dev-disk-by-uuid-3c3cbb68-c5c9-40ca-99b2-23b19a49cc43/Docker/nextcloud/db/:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=MonSuperMDPRootPourMySQL
      - MYSQL_PASSWORD=MonSuperMDPPourQueNextcloudAccedeALaDB
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud

  app:
    image: nextcloud
    restart: always
    ports:
      - 8080:80
    links:
      - db
    volumes:
      - /srv/dev-disk-by-uuid-3c3cbb68-c5c9-40ca-99b2-23b19a49cc43/Docker/nextcloud/data/:/var/www/html
    environment:
      - MYSQL_PASSWORD=MonSuperMDPPourQueNextcloudAccedeALaDB
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_HOST=db

```



## Deux conteneurs, deux rôles

- **`db`** (MariaDB) — stocke toutes les données structurées de Nextcloud (utilisateurs, métadonnées de fichiers, calendrier, contacts). N'est **jamais exposé** sur le réseau (pas de `ports:` dans son bloc) : seul le conteneur `app` peut lui parler, via le réseau interne Docker et son nom de service (`MYSQL_HOST=db`).
- **`app`** (Nextcloud) — l'application elle-même, exposée sur le port `8080`.

## Rôle du port

| Port | Usage |
|---|---|
| `8080:80` | Interface web Nextcloud (HTTP). Le `80` interne est fixe (imposé par l'image) ; `8080` est le port choisi côté hôte, `8080` déjà pris par ailleurs sur OMV le cas échéant à ajuster. |

## Compte administrateur

Contrairement aux autres services de la stack, Nextcloud ne demande **qu'un seul compte** lors du premier lancement (accessible sur `http://192.168.100.6:8080`) — ce compte reçoit automatiquement tous les droits d'administration (gestion des utilisateurs, des apps, des paramètres système). Pas de séparation entre compte "technique" et compte "personnel" nécessaire pour un usage à un seul utilisateur ; des comptes utilisateurs standards (sans droits admin) peuvent être ajoutés plus tard depuis les paramètres si l'accès est partagé avec d'autres personnes.

## Stockage des données

Deux volumes séparés :
- `Docker/nextcloud/db` — les données de la base MariaDB
- `Docker/nextcloud/data` — les fichiers Nextcloud eux-mêmes (documents, photos, configuration de l'application)

*(À compléter : détails sur les apps Nextcloud installées, la configuration du client de synchronisation, et un éventuel accès distant via Tailscale une fois cette étape du projet réalisée.)*