# Configuration — Jellyfin

## Fichier Docker Compose

```yaml
services:
  jellyfin:
    image: lscr.io/linuxserver/jellyfin:latest
    container_name: jellyfin
    environment:
      - PUID=1001
      - PGID=100
      - TZ=Africa/Ouagadougou
      - JELLYFIN_PublishedServerUrl=http://192.168.100.21
    volumes:
      - /chemin/vers/Docker/jellyfin/config:/config
      - /chemin/vers/media/series:/data/tvshows
      - /chemin/vers/media/films:/data/movies
    ports:
      - 8096:8096
      - 7359:7359/udp
      - 1900:1900/udp
    restart: unless-stopped
```

## Rôle de chaque port

| Port | Usage |
|---|---|
| `8096` | Interface web principale (HTTP) |
| `7359/udp` | Découverte automatique du serveur sur le réseau local (broadcast UDP, permet aux apps clientes de trouver le serveur sans taper l'IP manuellement) |
| `1900/udp` | Serveur DLNA — permet la lecture depuis des appareils sans app Jellyfin dédiée (téléviseurs plus anciens) |

Port `8920` (HTTPS) volontairement omis : nécessiterait un certificat SSL configuré sur Jellyfin, non mis en place pour l'instant.

## Accès depuis différents appareils

- **PC / navigateur** : `http://192.168.100.21:8096`
- **TV LG (modèles récents)** : application officielle disponible sur le LG Content Store
- **Sans application dédiée** : DLNA (détection automatique par la TV) ou lecture directe des fichiers via le partage SMB dans un lecteur comme VLC

## Transcodage matériel (Quick Sync) — non activé

Le CPU (i3-6100) dispose d'un GPU intégré compatible Quick Sync, qui aurait pu accélérer le transcodage vidéo. Non activé dans ce projet : cela nécessiterait soit un passthrough GPU complet vers la VM OMV (retirerait le GPU à l'hôte Proxmox pour toute autre VM), soit une configuration GVT-g plus complexe et moins stable sur ce matériel. Le transcodage reste donc en mode logiciel (CPU), suffisant pour l'usage actuel.

## Problème de transcodage forcé — résolu

**Symptôme** : le CPU montait à 100% dès la lecture d'un film, même dans des cas où un transcodage n'aurait pas dû être nécessaire (Direct Play attendu).

**Cause** : l'option **"Préférer le conteneur de média fMP4-HLS"** (Réglages de lecture → Avancé) forçait un réempaquetage du flux, empêchant l'envoi direct et brut du fichier original.

**Solution** : décocher cette option dans les paramètres de lecture du client utilisé.

## Bibliothèques configurées

- **Films** → pointée vers `/data/movies` (alimentée par Radarr, voir [arr-stack-setup.md](arr-stack-setup.md))
- **Séries** → pointée vers `/data/tvshows` (alimentée par Sonarr)