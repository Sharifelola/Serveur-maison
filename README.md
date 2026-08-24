# Projet NAS maison — Dell OptiPlex 3040

## Contexte

Réutilisation d'un ancien PC de bureau (Dell OptiPlex 3040) pour en faire un serveur NAS/média maison, plutôt que de le laisser inutilisé. 

## Objectif 

Elaborer un son propre cloud personnel, serveur de streaming automatiser et un apprentissage du self-hosting de linux et docker

## Matériels utilisee 

| Composant | Détail                                                 |
| --------- | ------------------------------------------------------ |
| CPU       | Intel i3-6100 (2 cœurs / 4 threads)                    |
| RAM       | 8 Go (6 Go alloués à la VM OMV)                        |
| Stockage  | 80 Go (système) + 512 Go (données, passthrough direct) |

## Architecture

```
Proxmox VE (hyperviseur, bare metal sur le disque 80 Go)
  └──
   VM OpenMediaVault (OMV)
        ├── Disque 512 Go en passthrough direct (SCSI)
        ├── Docker (via plugin OMV-Compose)
        │     ├── Nextcloud + MariaDB (cloud personnel)
        │     ├── Jellyfin (serveur multimédia)
        │     ├── Prowlarr (agrégateur d'indexeurs)
        │     ├── Radarr (gestion automatisée des films)
        │     ├── Sonarr (gestion automatisée des séries)
        │     ├── qBittorrent (client de téléchargement)
        │     └── FlareSolverr (contournement Cloudflare pour indexeurs)
        └── Partages SMB/CIFS (accès réseau local)
```


## fonctionnalites 
- serveur smb basique
- un serveur nexcloud (car plus intuitif pour la famille que smb)
- un serveur de telechargement 
- un serveur multimedia automatiser ([Détails de l'automatisation média](docs/media-automation.md))

## Pourquoi ces choix

- **OMV en VM sous Proxmox (plutôt qu'une installation directe sur le disque)** : permet de garder Proxmox comme couche de gestion/flexibilité (afin d'ajouter d'autre machine pour plus tard), tout en isolant complètement OMV avec son propre noyau, de plus un passthrough disque direct vers cette VM reste simple à mettre en place.
- **Services Docker hébergés dans la VM OMV (plutôt qu'en conteneurs LXC séparés sur Proxmox)** : le disque de données étant en passthrough exclusif vers la VM OMV, y héberger aussi Docker évite d'avoir à partager ce disque en réseau (SMB/NFS) vers des LXC externes, ce qui aurait recréé les problèmes de permissions rencontrés sur une de mes précédente installation de OMV.
- **Passthrough disque plutôt que disque virtuel** : accès natif SMART, pas de couche de virtualisation supplémentaire sur les données.
- **OpenMediaVault plutôt que TrueNAS Scale** : interface plus légère, moins gourmande en RAM (TrueNAS recommande ~8 Go rien que pour lui-même), suffisant pour un usage perso sur un seul disque de données sans redondance RAID.
  
- **ext4 plutôt que ZFS** : un seul disque de données, sans redondance possible — ZFS n'apporte pas de bénéfice réel dans cette configuration et en plus consommerais plus de performances en arriere plan.

- **Docker Compose natif OMV plutôt que Portainer** : le portaille Portainer ma ete utile dans les anciennes versions de OMV car celui ci ne disposait pas d'interface en ligne pour configurer docker (seullement le ssh) alors que avec les dernires version de OMV, on a un acces fonctionnalite de gestion de docker

- **Hardlinks (pas Copy) entre le dossier de téléchargement et la bibliothèque média** : fonctionnalite proposer par sonarr et radarr, elle permet aux fichiers media d'etre utiliser dans leurs dossiers respectif mais aussi de rester disponible pour les seeder sur le client torrent (preservons la philosophie du torrent ;) )

  
- **PUID/PGID non-root pour tous les conteneurs** : ici j'ai preferer la creation d'un compte separer pour la gestion des medias  avec le minimum de privileges necessaire car 
leur donner un acces total (root) considererer un gros risque si une faille est exploiter dans ces servises.


## Problèmes rencontrés et solutions


### Interface OMV-Extras en erreur 404 après installation
**Cause** : cache du navigateur non actualisé après l'ajout du plugin entrainant une mauvaise actuallisation des onglets de l'interface web.
**Solution** : rechargement forcé (Ctrl+Shift+R), ou en dernier recours utiliser le redémarrage des services web.

### qBittorrent — acces nom autoriser a l'interface web
**Cause** : le port interne du conteneur (`WEBUI_PORT`) ne correspondait pas au port externe utilisé pour y accéder — la protection anti-DNS-rebinding de qBittorrent rejette alors la connexion.
**Solution** : aligner le port interne et externe dans le fichier Compose, ou modifier directement `WebUI\HostHeaderValidation=false` dans `qBittorrent.conf` si on préfère garder des ports différents.

### Permissions refusées sur les dossiers Sonarr/Radarr (`not writable by user 'abc'`)
**Cause** : les permissions appliquées sur un dossier parent dans OMV n'avaient pas été propagées aux sous-dossiers existants (case "Récursif" non cochée).
**Solution** : réappliquer les privilèges avec l'option "Récursif" activée.

### VM n'utilisant qu'1 seul thread malgré 3 cores configurés
**Cause** : le champ "VCPUs" (distinct de "Cores") était resté fixé à 1 manuellement, ce qui prime sur le calcul Sockets × Cores.
**Solution** : aligner VCPUs sur le total Sockets × Cores, puis redémarrer complètement la VM.

### Sonarr/Radarr téléchargent parfois des releases sans piste française
**Cause** : le "Minimum Custom Format Score" du profil de qualité était à 0, ce qui laissait passer des releases sans aucun tag de langue détecté (score 0 ≥ 0).
**Solution** : remonter ce seuil à 1 pour rejeter toute release à score nul, et augmenter "Upgrade Until Custom Format Score" pour permettre les mises à niveau automatiques vers une meilleure version linguistique.

### GPU passthrough pour transcodage matériel Jellyfin — abandonné
**Analyse** : le CPU n'a qu'un seul GPU intégré (pas de GPU dédié séparé) ; un passthrough complet le retirerait entièrement à l'hôte Proxmox, empêchant toute autre VM d'en profiter à l'avenir.  
**Décision** : rester en transcodage logiciel (CPU), suffisant pour l'usage actuel.

## Sécurité

Sera completer au fur et a mesure

## Prochaines étapes

- [ ] Durcissement sécurité (mots de passe uniques par service déjà en place, à compléter avec 2FA / clés SSH)
- [ ] Accès distant via Tailscale
- [ ] Serveur mail auto-hébergé
- [ ] Monitoring UPS (Eaton via NUT)

## Stack technique

Proxmox VE · OpenMediaVault · Docker Compose · Nextcloud · Jellyfin · Prowlarr · Radarr · Sonarr · qBittorrent · FlareSolverr