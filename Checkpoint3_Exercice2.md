# Exercice 2 : Manipulations pratiques sur VM Linux (temps estimé : 2h30)
Pour cet exercice tu as besoin de la VM SRVLX01.

## Partie 1 : Gestion des utilisateurs
### Q.2.1.1 Sur le serveur, créer un compte pour ton usage personnel.

```bash
sudo adduser wilder
#Attribuer des droits sudo
sudo usermod -aG sudo monutilisateur
```

### Q.2.1.2 Quelles préconisations proposes-tu concernant ce compte ?

Préconisations simples pour le compte :
- Mot de passe sécurisé : Long et complexe (Minimum 12 caractères ,Mélanger majuscules, minuscules, chiffres et caractères spéciaux)
- Droits limités : Ne pas donner sudo si inutile
- Connexion SSH sécurisée : Utiliser une clé SSH
- Expiration du mot de passe : sudo chage -M 90 wilder
- Surveillance : Vérifier les logs avec journalctl -u sshd

## Partie 2 : Configuration de SSH
Un serveur SSH est lancé sur le port par défaut.
Il est possible de s'y connecter avec n'importe quel compte, y compris le compte root.

- Oui , Tous les utilisateurs peuvent se connecter.
- NON, root est désactivé . par défaut PermitRootLogin no

### Q.2.2.1 Désactiver complètement l'accès à distance de l'utilisateur root.
```bash
sudo nano /etc/ssh/sshd_config
PermitRootLogin no
```

### Q.2.2.2 Autoriser l'accès à distance à ton compte personnel uniquement.

```bash
sudo nano /etc/ssh/sshd_config
AllowUsers wilder
```

### Q.2.2.3 Mettre en place une authentification par clé valide et désactiver l'authentification par mot de passe

- 1. Générer une clé SSH sur ton ordinateur local 
```bash
ssh-keygen -t rsa -b 4096
```
2. Copier la clé publique sur le serveur Ubuntu :

```bash
ssh-copy-id wilder@192.168.1.2
```
3. Désactiver l'authentification par mot de passe sur le serveur Ubuntu :
```bash
# Édite le fichier de configuration SSH :
sudo nano /etc/ssh/sshd_config

# Modifie ou ajoute ces lignes :
PasswordAuthentication no
ChallengeResponseAuthentication no
```
4. Redémarrer le service SSH :
```bash
sudo systemctl restart sshd
```
5. Tester la connexion :
```bash
ssh wilder@192.168.1.2
```

## Partie 3 : Analyse du stockage
### Q.2.3.1 Quels sont les systèmes de fichiers actuellement montés ?

- Lsblk
- Df -h

![lsblk](https://github.com/KAOUTARBAH/Checkpoint3/blob/main/Images/lsblk.png)

![df](https://github.com/KAOUTARBAH/Checkpoint3/blob/main/Images/df.png)

### Q.2.3.2 Quel type de système de stockage ils utilisent ?
- - Indiquez le type de système de stockage (ext4, xfs, etc.) utilisé par chaque système de fichiers monté.

```bash
mount 
/dev/sdb1 on /mnt/home type xfs (rw,relatime,attr2,inode64,logbufs=8,logbsize=32k,noquota)
/dev/sdc1 on /data type xfs (rw,relatime,attr2,inode64,logbufs=8,logbsize=32k,noquota)
tmpfs on /run type tmpfs (rw,nosuid,nodev,noexec,relatime,size=401000k,mode=755,inode64)
/dev/sda2 on / type ext4 (rw,relatime)
securityfs on /sys/kernel/security type securityfs (rw,nosuid,nodev,noexec,relatime)
tmpfs on /dev/shm type tmpfs (rw,nosuid,nodev,inode64)
tmpfs on /run/lock type tmpfs (rw,nosuid,nodev,noexec,relatime,size=5120k,inode64)
```
![mont](https://github.com/KAOUTARBAH/Checkpoint3/blob/main/Images/mount.png)

### Q.2.3.3 Ajouter un nouveau disque de 8,00 Gio au serveur et réparer le volume RAID

#### Partitionner le disque `/dev/sdb`

- Lancez l'utilitaire `fdisk` :

```bash
sudo fdisk /dev/sdb
```

- Dans l'interface `fdisk` :
  - Tapez `m` pour afficher l'aide.
  - Ajoutez une nouvelle partition avec `n`.
  - Créez une partition primaire qui prend toute la taille du disque (+8G).
  - Changez le type de partition en RAID Linux :
    - Tapez `t`.
    - Entrez le code hexadécimal `fd` pour RAID Linux.
  - Sauvegardez et quittez en tapant `w`.

- Vérifiez la partition créée :

```bash
sudo fdisk -l
```

#### Répéter l'opération pour `/dev/sdc`

- Exécutez les mêmes commandes pour partitionner `/dev/sdc`.

![partition](https://github.com/KAOUTARBAH/Checkpoint3/blob/main/Images/partition.png)

#### Installation de `mdadm`

- Si `mdadm` n'est pas installé, exécutez :

```bash
sudo apt install mdadm
```

#### Création du RAID 1

- Exécutez la commande suivante pour créer le RAID :

```bash
sudo mdadm --create /dev/md0 --level 1 --raid-devices 2 /dev/sdb1 /dev/sdc1
```

#### Vérification du RAID

- Pour vérifier l'état du RAID :

```bash
cat /proc/mdstat
```

- Ou avec `mdadm` :

```bash
sudo mdadm --detail /dev/md0
```

- Afficher les disques inclus dans le RAID :

```bash
lsblk -f
```

#### Formatage du RAID

- Formatez le volume RAID en `ext4` avec le nom *PersonalData* :

```bash
sudo mkfs.ext4 /dev/md0 -L "PersonalData"
```

- Vérifiez le formatage :

```bash
lsblk -f
```

#### Montage du RAID

- Créez un dossier de montage :

```bash
sudo mkdir -p /home/wilder/Data-RAID1
```

- Montez le RAID dans ce dossier :

```bash
sudo mount /dev/md0 /home/wilder/Data-RAID1/
```
![montRaid](https://github.com/KAOUTARBAH/Checkpoint3/blob/main/Images/montRaid.png)


#### Montage automatique au démarrage

- Ajoutez la ligne suivante à `/etc/fstab` :

```bash
/dev/md0 /home/wilder/Data-RAID1 ext4 nofail 0 0
```

## Verrouillage du nom `md0`

- Exécutez la commande suivante pour verrouiller le nom du RAID :

```bash
sudo mdadm --detail --scan >> /etc/mdadm/mdadm.conf
sudo update-initramfs -u
```

## Test du bon fonctionnement du RAID

- Créez un fichier et un dossier dans le RAID :

```bash
touch /home/wilder/Data-RAID1/test_raid1.txt
mkdir /home/wilder/Data-RAID1/dossier_test
```

- Vérifiez leur présence :

```bash
ls -l /home/wilder/Data-RAID1
```

### Q.2.3.4 Ajouter un nouveau volume logique LVM de 2 Gio qui servira à héberger des sauvegardes. Ce volume doit être monté automatiquement à chaque démarrage dans l'emplacement par défaut : /var/lib/bareos/storage.



#### Créer un Volume Physique (PV)
Si vous avez ajouté un disque supplémentaire à votre machine virtuelle, vous pouvez le transformer en volume physique (PV) avec la commande suivante :

```bash
sudo apt-get update
sudo apt-get install lvm2

sudo pvcreate /dev/sdg  # Créez un PV sur le disque /dev/sdg
```
![pv](https://github.com/KAOUTARBAH/Checkpoint3/blob/main/Images/pv.png)

### 3. Créer un Groupe de Volumes (VG)

- Une fois que vous avez un PV, vous pouvez créer un groupe de volumes (VG) nommé **vg_datas** pour regrouper plusieurs PV. Si vous avez plusieurs disques (par exemple /dev/sdb et /dev/sdc), vous pouvez les ajouter à un VG.

```bash
sudo vgcreate vg_datas /dev/sdb 
```
![vg](https://github.com/KAOUTARBAH/Checkpoint3/blob/main/Images/vg.png)

- Vérifier avec vgdisplay que la création s'est bien passée (tu dois avoir un VG Size de la taille totale des 2 disques)
```bash
sudo vgdisplay
```
### 4. Créer un Volume Logique (LV)

- Une fois le VG créé, vous pouvez créer un LV. Par exemple, pour créer un LV de 10 Go :

```bash
sudo lvcreate -L 2G -n lv_bareos_storage vg_datas
 # Crée un LV 'lv_datas' de 2 Go dans 'vg_datas'
```
![lv](https://github.com/KAOUTARBAH/Checkpoint3/blob/main/Images/lv.png)

- Vérifier avec lvdisplay que tout est bien crée (tu dois avoir un LV Size de la taille que tu as choisi).
```bash
sudo lvdisplay
```

### 5. Formater et monter le LV

Après avoir créé le LV, vous devez le formater avec un système de fichiers (par exemple ext4) et le monter.

```bash
# Formater le LV avec ext4
sudo mkfs.ext4 /dev/vg_datas/lv_bareos_storage

# Créer un point de montage
sudo mkdir /var/lib/bareos/storage


# Monter le LV
sudo mount /dev/vg_datas/lv_bareos_storage /var/lib/bareos/storage

``` 
![montage](https://github.com/KAOUTARBAH/Checkpoint3/blob/main/Images/montage.png)

- Pour le montage automatique au démarrage, ajouter au fichier /etc/fstab la ligne /dev/vg_datas/lv_datas /mnt/datas ext4 defaults 0 2

```bash
/etc/fstab
/etc/fstab /dev/vg_datas/lv_bareos_storage /var/lib/bareos/storage ext4 defaults 0 2
```
- Vérifier la configuration
```bash
sudo mount -a
```


### Q.2.3.5 Combien d'espace disponible reste-t-il dans le groupe de volume ?

## Partie 4 : Sauvegardes
Le logiciel bareos est installé sur le serveur.
Les composants bareos-dir, bareos-sd et bareos-fd sont installés avec une configuration par défaut.

### Q.2.4.1 Expliquer succinctement les rôles respectifs des 3 composants bareos installés sur la VM.

- **Bareos Director** : Chef d’orchestre du système de sauvegarde, il planifie, contrôle et lance les tâches.  
Il gère tous les autres composants et est installé sur le serveur de gestion des sauvegardes.  → Chef d’orchestre

- **Bareos File Daemon (Client)** : Installé sur chaque machine à sauvegarder, il collecte les fichiers et les envoie au Storage Daemon. → Collecteur des fichiers

- **Bareos Storage Daemon** : Il gère le stockage des sauvegardes sur disque, bande ou autre média. Il reçoit les données du File Daemon et les écrit dans le stockage défini. → Stockeur des données

## Partie 5 : Filtrage et analyse réseau
### Q.2.5.1 Quelles sont actuellement les règles appliquées sur Netfilter ?

![iptables](https://github.com/KAOUTARBAH/Checkpoint3_Blanc/blob/main/Images/iptables.png)


### Q.2.5.2 Quels types de communications sont autorisées ?
Les types de communications autorisées sont généralement celles que vous avez spécifiquement autorisées dans les règles de votre pare-feu. Par exemple :

- HTTP (port 80)
- HTTPS (port 443)
-bSSH (port 22)
- DNS (port 53)

### .2.5.3 Quels types sont interdit ?

Les communications interdites sont celles qui n'ont pas de règle autorisant leur passage. Par exemple, des ports comme :

- SMB (port 445)
- Telnet
- FTP non sécurisé

### Q.2.5.4 Sur nftables, ajouter les règles nécessaires pour autoriser bareos à communiquer avec les clients bareos potentiellement présents sur l'ensemble des machines du réseau local sur lequel se trouve le serveur.

Rappel : Bareos utilise les ports TCP 9101 à 9103 pour la communication entre ses différents composants.

```bash
sudo nft add rule inet filter input ip saddr 192.168.1.0/24 tcp dport 9101 accept
sudo nft add rule inet filter input ip saddr 192.168.1.0/24 tcp dport 9102 accept
```

## Partie 6 : Analyse de logs
### Q.2.6.1 Lister les 10 derniers échecs de connexion ayant eu lieu sur le serveur en indiquant pour chacun :

- La date et l'heure de la tentative
- L'adresse IP de la machine ayant fait la tentative

```bash
sudo journalctl | grep "Failed password" | tail -n 10 | 
```
![log2](https://github.com/KAOUTARBAH/Checkpoint3_Blanc/blob/main/Images/log2.png)

```bash
sudo grep "Failed password" /var/log/auth.log | tail -n 10
```
![log](https://github.com/KAOUTARBAH/Checkpoint3_Blanc/blob/main/Images/log.png)