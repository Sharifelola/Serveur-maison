# Projet NAS maison — Dell OptiPlex 3040

## Contexte

Réutilisation d'un ancien PC de bureau (Dell OptiPlex 3040) pour en faire un serveur NAS/média maison, plutôt que de le laisser inutilisé. 

## Objectif 

Élaborer mon propre cloud personnel, serveur de streaming automatisé et un apprentissage du self-hosting de Linux et Docker

## Matériel utilisé 

| Composant | Détail                                                 |
| --------- | ------------------------------------------------------ |
| CPU       | Intel i3-6100 (2 cœurs / 4 threads)                    |
| RAM       | 8 Go (6 Go alloués à la VM OMV)                        |
| Stockage  | 80 Go (système) + 512 Go (données, passthrough direct) |

## Architecture

```
Proxmox VE (hyperviseur, bare metal sur le disque 80 Go)
  └── VM OpenMediaVault (OMV)
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


## Fonctionnalités 
- [serveur NAS basé sur OMV](docs/omv-setup.md)
- [serveur smb basique](docs/omv-setup.md#partages-réseau-smbcifs) 
- [un serveur Nextcloud](docs/nextcloud-setup.md) 
- [un serveur de telechargement](docs/qbittorent-setup.md)
- [un serveur multimédia automatisé](docs/media-automation.md)

## Pourquoi ces choix

- **OMV en VM sous Proxmox (plutôt qu'une installation directe sur le disque)** : permet de garder Proxmox comme couche de gestion/flexibilité (afin d'ajouter d'autre machine pour plus tard), tout en isolant complètement OMV avec son propre noyau, de plus un passthrough disque direct vers cette VM reste simple à mettre en place.
- **Services Docker hébergés dans la VM OMV (plutôt qu'en conteneurs LXC séparés sur Proxmox)** : le disque de données étant en passthrough exclusif vers la VM OMV, y héberger aussi Docker évite d'avoir à partager ce disque en réseau (SMB/NFS) vers des LXC externes, ce qui aurait recréé les problèmes de permissions rencontrés sur une de mes précédente installation de OMV.
- **Passthrough disque plutôt que disque virtuel** : accès natif SMART, pas de couche de virtualisation supplémentaire sur les données.
- **OpenMediaVault plutôt que TrueNAS Scale** : interface plus légère, moins gourmande en RAM (TrueNAS recommande ~8 Go rien que pour lui-même), suffisant pour un usage perso sur un seul disque de données sans redondance RAID.
  
- **ext4 plutôt que ZFS** : un seul disque de données, sans redondance possible — ZFS n'apporte pas de bénéfice réel dans cette configuration et en plus consommerais plus de performances en arrière plan.

- **Docker Compose natif OMV plutôt que Portainer** : le portail Portainer m'a été utile dans les anciennes versions de OMV car celui ci ne disposait pas d'interface en ligne pour configurer docker (seulement le ssh) alors que avec les derniers version de OMV, on a un acces fonctionnalité de gestion de docker

- **Hardlinks (pas Copy) entre le dossier de téléchargement et la bibliothèque média** : fonctionnalité proposer par sonarr et radarr, elle permet aux fichiers média d’être utiliser dans leurs dossiers respectif mais aussi de rester disponible pour les seeder sur le client torrent (préservons la philosophie du torrent ;) )

  
- **PUID/PGID non-root pour tous les conteneurs** : ici j'ai préférer la création d'un compte séparer pour la gestion des médias  avec le minimum de privilèges nécessaire car 
leur donner un acces total (root)serait considéré comme un gros risque si une faille est exploiter dans ces services.


## Problèmes rencontrés et solutions


### Interface OMV-Extras en erreur 404 après installation
**Cause** : cache du navigateur non actualisé après l'ajout du plugin entrainant une mauvaise actuallisation des onglets de l'interface web.
**Solution** : rechargement forcé (Ctrl+Shift+R), ou en dernier recours utiliser le redémarrage des services web.

### qBittorrent — acces non autoriser a l'interface web
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

### GPU passthrough pour transcodage matériel Jellyfin — tenté puis abandonné
Voir le récit détaillé du troubleshooting dans [jellyfin-setup.md](docs/jellyfin-setup.md#transcodage-matériel-quick-sync--tenté-et-abandonné). En résumé : configuration IOMMU/VFIO correctement mise en place (GPU isolé, pilote lié à vfio-pci), mais échec persistant au démarrage de la VM (fault DMAR répété) malgré plusieurs correctifs communautaires testés. Abandon au profit d'un réencodage préventif ciblé des fichiers les plus coûteux (HEVC 10-bit/HDR).


## Sécurité

Tout les mot de passe utiliser dans la configuration ont ete generer avec Bitwaren avec un minimum de 21 characteres pour avoir plus de securite et ainsi eviter les bias humain.

## Prochaines étapes

- [ ] Durcissement sécurité (mots de passe uniques par service déjà en place, à compléter avec 2FA / clés SSH)
- [ ] mise en place d'un réencodage préventif pour améliorer la qualite de service de Jellyfin (via HandBrake)
- [ ] amielorer la synchronisation sur Nextcloud
- [ ] Monitoring UPS (Eaton via NUT)
- [ ] Accès distant via Tailscale
- [ ] Serveur mail auto-hébergé

## Stack technique

Proxmox VE · OpenMediaVault · Docker Compose · Nextcloud · Jellyfin · Prowlarr · Radarr · Sonarr · qBittorrent · FlareSolverr