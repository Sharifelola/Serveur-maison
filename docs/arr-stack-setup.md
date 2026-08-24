# Configuration — Prowlarr, Radarr, Sonarr, FlareSolverr

Ce document détaille la configuration technique de ces services. Pour comprendre le flux global et le rôle de chaque brique, voir [media-automation.md](media-automation.md).


## Rappelle de l'arborecense


```
Telechargement/
Docker/
├── Data            
├── config-files
│   ├── Nextcloud
│   ├── flaresolver
│   ├── jellyfin
│   ├── prowlarr
│   ├── qbittorrent
│   ├── radarr
│   └── sonarr
├── jellyfin
│   └── config
├── nextcloud
│   ├── data
│   └── db
├── prowlarr
│   └── config
├── qbittorent
│   └── config
├── radarr
│   └── config
└── sonarr
    └── config


```


## Fichiers Docker Compose

### Prowlarr

```yaml
---
services:
  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    environment:
      - PUID=1001
      - PGID=100
      - TZ=Africa/Ouagadougou
    volumes:
      - /srv/dev-disk-by-uuid-3c3cbb68-c5c9-40ca-99b2-23b19a49cc43/Docker/prowlarr/config:/config
    ports:
      - 9696:9696
    restart: unless-stopped
```

### Radarr

```yaml
---
services:
  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    environment:
      - PUID=1001
      - PGID=100
      - TZ=Africa/Ouagadougou
    volumes:
      - /srv/dev-disk-by-uuid-3c3cbb68-c5c9-40ca-99b2-23b19a49cc43/Docker/radarr/config:/config
      - /srv/dev-disk-by-uuid-3c3cbb68-c5c9-40ca-99b2-23b19a49cc43/media/films:/movies 
      - /srv/dev-disk-by-uuid-3c3cbb68-c5c9-40ca-99b2-23b19a49cc43/Telechargement:/downloads
    ports:
      - 7878:7878
    restart: unless-stopped

```

### Sonarr

```yaml
---
services:
  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    environment:
      - PUID=1001
      - PGID=100
      - TZ=Africa/Ouagadougou
    volumes:
      - /srv/dev-disk-by-uuid-3c3cbb68-c5c9-40ca-99b2-23b19a49cc43/Docker/sonarr/config:/config
      - /srv/dev-disk-by-uuid-3c3cbb68-c5c9-40ca-99b2-23b19a49cc43/media/series:/tv
      - /srv/dev-disk-by-uuid-3c3cbb68-c5c9-40ca-99b2-23b19a49cc43/Telechargement:/downloads
    ports:
      - 8989:8989
    restart: unless-stopped

```

### FlareSolverr

```yaml
---
services:
  flaresolverr:
    image: ghcr.io/flaresolverr/flaresolverr:latest
    container_name: flaresolverr
    environment:
      - LOG_LEVEL=info
      - TZ=Africa/Ouagadougou
    ports:
      - 8191:8191
    restart: unless-stopped
```

Pas de volume de configuration nécessaire ici — FlareSolverr ne stocke rien de permanent, il agit uniquement comme relais à la volée pour résoudre les protections Cloudflare (voir [Résolution Cloudflare](#résolution-cloudflare-flaresolverr) plus bas).

## Compte technique dédié

Tous les conteneurs utilisent le **PUID/PGID** d'un compte OMV créé spécifiquement (`mediaAgent`), plutôt que le compte administrateur personnel — limite l'accès en cas de faille de sécurité dans l'un des services. Voir la justification complète dans le [README principal](../README.md#pourquoi-ces-choix).

## Étapes de configuration (dans l'ordre)

1. **Récupérer les clés API** de Radarr et Sonarr (Settings → General → Security dans chaque application)
2. **Connecter Prowlarr aux deux apps** : Prowlarr → Settings → Apps → ajouter Radarr (URL + clé API) et Sonarr (URL + clé API)
3. **Ajouter les indexeurs** dans Prowlarr (Indexers → Add Indexer)
4. **Connecter Radarr/Sonarr à qBittorrent** : Settings → Download Clients → ajouter qBittorrent (host, port 8081, identifiants, catégorie dédiée par app)
5. **Définir les dossiers racines** : `/movies` dans Radarr, `/tv` dans Sonarr
6. **Configurer les profils de qualité et Custom Formats** (voir ci-dessous)
7. **Déployer FlareSolverr et le lier à Prowlarr** (voir ci-dessous) si un ou plusieurs indexeurs sont protégés par Cloudflare

## Filtrage par langue — Custom Formats

| Custom Format | Regex (Release Title) | Score |
|---|---|---|
| `Multi` | `\bMULTI\b` | 100 |
| `french` | `\b(FRENCH\|VFF\|TRUEFRENCH\|VF\|FR)\b` | 80 |
| `vostfr` | `\b(VOSTFR\|SUBFRENCH)\b` | 10 |

Paramètres du profil de qualité (Settings → Profiles) :
- **Minimum Custom Format Score : 1** — rejette toute release sans aucun tag de langue détecté
- **Upgrade Until Custom Format Score : 100** — autorise le remplacement automatique par une meilleure version linguistique si elle sort plus tard

⚠️ Toutes les conditions de ces Custom Formats sont laissées **sans** l'option "Required" cochée — cela leur donne une logique **OU** (n'importe laquelle suffit), plutôt qu'une exigence que toutes les variantes soient présentes en même temps.

## Résolution Cloudflare (FlareSolverr)

FlareSolverr est déployé séparément (voir fichier Compose ci-dessus) et connecté à Prowlarr via **Settings → Indexer Proxies** (URL : `http://192.168.100.21:8191`).

L'association entre un indexeur et FlareSolverr se fait via un **système de tags** : le même tag (ex: `flaresolverr`) est appliqué à la fois sur le proxy FlareSolverr et sur l'indexeur concerné (Indexers → paramètres de l'indexeur) — Prowlarr applique alors automatiquement le proxy à tout indexeur partageant ce tag.

Voir aussi le schéma dans [media-automation.md](media-automation.md#fonctionnement-de-prowlarr-et-contournement-cloudflare).

## Limite connue : parsing du titre

Le système d'analyse de titre suppose que l'année suit immédiatement le nom (`Titre.2024.TAGS`). Un format non conventionnel (`Titre TAGS 2024`, année en fin de titre) peut échouer à être reconnu — recours possible : import manuel via Radarr/Sonarr → Manual Import.