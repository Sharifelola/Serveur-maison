# Configuration — Proxmox & OpenMediaVault

Ce document détaille la mise en place de la couche infrastructure : hyperviseur, VM, stockage et partage réseau, sur laquelle reposent tous les services décrits dans les autres fichiers de `docs/`.

## Proxmox VE

Installé en barre métal sur le disque de 80 Go — sert uniquement d'hyperviseur et ne fait tourner qu'une seule VM (OMV).

### Configuration CPU de la VM

| Paramètre  | Valeur | Remarque                                                                                                                                              |
| ---------- | ------ | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| Sockets    | 1      | Le CPU physique (i3-6100) n'a qu'un seul socket                                                                                                       |
| Cores      | 3      | Sur 4 threads logiques disponibles au total (2 cœurs physiques × 2 via HyperThreading) — 1 thread laissé à Proxmox                                    |
| VCPUs      | 3      | ⚠️ Doit être aligné manuellement sur Sockets × Cores — laissé à 1 par erreur au départ, ce qui limitait la VM à un seul thread malgré 3 cores alloués |
| Flag `aes` | On     | Accélère le chiffrement matériel (utile pour HTTPS/VPN)                                                                                               |

### RAM allouée : 6 Go (sur 8 Go physiques)

## Répartition des disques

| Disque | Usage                                                                          |
| ------ | ------------------------------------------------------------------------------ |
| 80 Go  | Système Proxmox + disque virtuel (32Go) OS de la VM OMV                        |
| 512 Go | Passthrough direct — entièrement dédié aux données (Docker, médias, Nextcloud) |

## Passthrough du disque de données

Le disque de 512 Go est attaché en **passthrough direct** (pas un disque virtuel classique, permettant a OMV de la voir  comme un disque physique natif (SMART, identifiant réel, etc.).

Identification du disque par ID stable (pas `/dev/sdX`, qui peut changer d'un redémarrage à l'autre) :

D'abord nous identifions le disque par le nom system (ici on l'utilise temporairement car elle peut varier a chaque redémarrage)
```bash
lsblk
```
Apres identification du disque (par exemple sdb), recherche de  l'ID du disque correspondant 
```bash
ls -l /dev/disk/by-id/
```

Attachement à la VM (remplacer `VMID` l'ID de la VM et l'identifiant du disque) :
```bash
qm set VMID -scsi1 /dev/disk/by-id/ata-XXXXXXXXXXXXXXXX
```

Vérification :
```bash
qm config VMID
```



## Compte technique dédié aux conteneurs

Un compte OMV séparé, `mediaAgent`, a été créé pour servir de PUID/PGID à tous les conteneurs Docker — plutôt que d'utiliser le compte administrateur personnel. Voir la justification complète dans le [README principal](../README.md#pourquoi-ces-choix).

## Fuseau horaire

Un décalage de 2h a été constaté après l'installation initiale (le fuseau horaire restait sur une valeur par défaut). Correction :
- **OMV** : Système → Date & Heure → fuseau horaire correct (Africa/Ouagadougou) + NTP activé


## Partages réseau (SMB/CIFS)

SMB choisi plutôt que NFS pour la compatibilité multi-plateforme (Windows, Mac, Linux, mobile). Configuration :
- [x] activer 
- **groupe de travail** : WORKGROUP
- **répertoire personnel** : désactiver car la plus part des utilisateurs auront nextcloud
- **version minimum du protocole** : SMB2 afin d’éviter les failles de la versions 1
- [ ] **activer netbios** 
- [ ] **activer server win**
- [ ] **utiliser sendfile**
- [x] **E/S asynchrone**

## Gestion des permissions — leçon apprise

Lors de la configuration des dossiers partagés destinés aux conteneurs (`media/films`, `media/series`), les permissions appliquées sur un dossier parent ne se propagent **pas automatiquement** aux sous-dossiers déjà existants dans OMV — l'option **"Récursif"** doit être explicitement cochée en plus de "Remplacer" lors de l'application des privilèges, sinon les sous-dossiers gardent leurs anciennes permissions.


## Système de fichiers

**ext4** retenu pour le disque de données (voir justification dans le [README principal](../README.md#pourquoi-ces-choix) — un seul disque sans redondance rend RAID/ZFS peu pertinents ici).

## Docker via OMV-Extras

L'ancien plugin `openmediavault-docker` a été retiré des versions récentes d'OMV et remplacé par `openmediavault-compose` (basé sur Docker Compose) — installable depuis **Système → Extensions**, après avoir activé le dépôt Docker dans **Système → omv-extras**.

⚠️ Après l'activation d'un nouveau plugin/dépôt, un rechargement forcé du navigateur (Ctrl+Shift+R) est souvent nécessaire — l'interface Angular d'OMV garde en cache les anciens fichiers, ce qui peut donner une fausse impression d'erreur 404.