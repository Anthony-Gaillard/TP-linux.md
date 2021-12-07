# Partie 1 : Préparation de la machine `backup.tp6.linux`

**La machine `backup.tp6.linux` sera chargée d'héberger les sauvegardes.**

Autrement dit, avec des mots simples : la machine `backup.tp6.linux` devra stocker des fichiers, qui seront des archives compressées.

Rien de plus simple non ? Un fichier, ça se met dans un dossier, et walou.

**ALORS OUI**, c'est vrai, mais on va aller un peu plus loin que ça :3

**Ce qu'on va faire, pour augmenter le niveau de sécu de nos données, c'est les stocker sur un espace vraiment dédié. C'est à dire une partition dédiée, sur un disque dur dédié.**

Au menu :

- ajouter un disque dur à la VM
- créer une nouvelle partition sur le disque avec LVM
- formater la partition pour la rendre utilisable
- monter la partition pour la rendre accessible
- rendre le montage de la partition automatique, pour qu'elle soit toujours accessible

> On ne travaille que sur `backup.tp6.linux` dans cette partie !

# I. Ajout de disque

Pour ajouter un disque, bah vous allez au magasin et vous achetez un disque ? n_n

Nan, en vrai, les VMs, c'est virtuel. Leurs disques sont virtuels. Donc on va simplement ajouter un disque virtuel à la VM, depuis VirtualBox.

Je vous laisse faire pour cette partie, avec vos ptites mains, vot' ptite tête et vot' p'tit pote Google. Rien de bien sorcier.

🌞 **Ajouter un disque dur de 5Go à la VM `backup.tp6.linux`**

- pour me prouver que c'est fait dans le compte-rendu, vous le ferez depuis le terminal de la VM
- la commande `lsblk` liste les périphériques de stockage branchés à la machine
- vous mettrez en évidence le disque que vous venez d'ajouter dans la sortie de `lsblk`
```bash
[antho@backup ~]$ lsblk
sdb           8:16   0    5G  0 disk 
```
# II. Partitioning

> [**Référez-vous au mémo LVM pour réaliser cette partie.**](../../cours/memos/lvm.md)

Le partitionnement est obligatoire pour que le disque soit utilisable. Ici on va rester simple : une seule partition, qui prend toute la place offerte par le disque.

Comme vu en cours, le partitionnement dans les systèmes GNU/Linux s'effectue généralement à l'aide de LVM.

Allons !

🌞 **Partitionner le disque à l'aide de LVM**

- créer un *physical volume (PV)* : le nouveau disque ajouté à la VM
- créer un nouveau *volume group (VG)*
  - il devra s'appeler `backup`
  - il doit contenir le PV créé à l'étape précédente
- créer un nouveau *logical volume (LV)* : ce sera la partition utilisable
  - elle doit être dans le VG `backup`
  - elle doit occuper tout l'espace libre
  
```bash
  [antho@backup ~]$  sudo pvcreate /dev/sdb
  Physical volume "/dev/sdb" successfully created.
 
  [antho@backup ~]$ sudo vgcreate backup /dev/sdb
  Volume group "backup" successfully created
 
  [antho@backup ~]$ sudo lvcreate -l 100%FREE backup -n last_data
  [sudo] Mot de passe de antho : 
  Logical volume "last_data" created.
```

🌞 **Formater la partition**

- vous formaterez la partition en ext4 (avec une commande `mkfs`)
  - le chemin de la partition, vous pouvez le visualiser avec la commande `lvdisplay`
  - pour rappel un *Logical Volume (LVM)* **C'EST** une partition

```bash 
[antho@backup ~]$ sudo mkfs -t ext4 /dev/backup/last_data
[sudo] Mot de passe de antho : 
mke2fs 1.45.6 (20-Mar-2020)
En train de créer un système de fichiers avec 1309696 4k blocs et 327680 i-noeuds.
UUID de système de fichiers=a14a7d7e-2241-47f3-997a-f458c108c092
Superblocs de secours stockés sur les blocs : 
	32768, 98304, 163840, 229376, 294912, 819200, 884736

Allocation des tables de groupe : complété                        
Écriture des tables d'i-noeuds : complété                        
Création du journal (16384 blocs) : complété
Écriture des superblocs et de l'information de comptabilité du système de
fichiers : complété
```
🌞 **Monter la partition**

- montage de la partition (avec la commande `mount`)
  - la partition doit être montée dans le dossier `/backup`
  - preuve avec une commande `df -h` que la partition est bien montée
  - prouvez que vous pouvez lire et écrire des données sur cette partition
- définir un montage automatique de la partition (fichier `/etc/fstab`)
  - vous vérifierez que votre fichier `/etc/fstab` fonctionne correctement

```bash
[antho@backup ~]$ sudo mkdir /mnt/backup
[antho@backup ~]$ sudo mount /dev/backup/last_data /mnt/backup
[antho@backup ~]$ df -h
Sys. de fichiers             Taille Utilisé Dispo Uti% Monté sur
devtmpfs                       387M       0  387M   0% /dev
tmpfs                          405M       0  405M   0% /dev/shm
tmpfs                          405M    5,6M  400M   2% /run
tmpfs                          405M       0  405M   0% /sys/fs/cgroup
/dev/mapper/rl-root            6,2G    2,8G  3,5G  44% /
/dev/sda1                     1014M    266M  749M  27% /boot
tmpfs                           81M       0   81M   0% /run/user/1000
/dev/mapper/backup-last_data   4,9G     20M  4,6G   1% /mnt/backup
,,,

```bash
[antho@backup ~]$ sudo nano /etc/fstab
[sudo] Mot de passe de antho : 
[antho@backup ~]$ sudo umount /mnt/backup
[antho@backup ~]$ sudo mount -av
/                         : ignoré
/boot                     : déjà monté
none                      : ignoré
mount : /mnt/backup ne contient pas d'étiquettes SELinux.
       Vous avez monté un système de fichiers permettant l'utilisation
       d’étiquettes, mais qui n'en contient pas, sur un système SELinux.
       Les applications sans droit généreront probablement des messages
       AVC et ne pourront pas accéder à ce système de fichiers.
       Pour plus de précisions, consultez restorecon(8) et mount(8).
/mnt/backup              : successfully mounted
[antho@backup ~]$ 
````

# Partie 2 : Setup du serveur NFS sur `backup.tp6.linux`

Bon cette partie, je pense vous commencez à être rodés :

- install un paquet qui contient un service
- conf le service
- lancer le service
- analyser le service
- tester le service

Oè oè oè. Limite redondant l'histoire. Ca devrait pas poser de problème alors ? :)

Le principe d'un serveur NFS :

- on crée des dossiers sur le serveur NFS
- le service NFS a pour but de rendre accessibles ces dossiers sur le réseau
- pour ça, il faut qu'une autre machine utilise un client NFS : elle accédera alors au dossier qui se trouve en réalité sur le serveur NFS

> Un bête partage de dossier quoi.

🌞 **Préparer les dossiers à partager**

- créez deux sous-dossiers dans l'espace de stockage dédié
  - `/backup/web.tp6.linux/`
  - `/backup/db.tp6.linux/`

  ```bash
  [antho@backup ~]$ sudo mkdir web.tp6.linux
[sudo] Mot de passe de antho : 
[antho@backup ~]$ sudo mkdir db.tp6.linux
[antho@backup ~]$ ls
db.tp6.linux  myfile.conf  pvcreate  web.tp6.linux
[antho@backup ~]$ 
```

🌞 **Install du serveur NFS**

- installez le paquet `nfs-utils`
```bash
[antho@backup ~]$ sudo dnf install nfs-utils
Installé:
  gssproxy-0.8.0-19.el8.x86_64              keyutils-1.5.10-9.el8.x86_64        
  libverto-libevent-0.3.0-5.el8.x86_64      nfs-utils-1:2.3.3-46.el8.x86_64     
  rpcbind-1.2.5-8.el8.x86_64               

Terminé !
```
🌞 **Conf du serveur NFS**

- fichier `/etc/idmapd.conf`

```bash
[antho@backup ~]$ sudo nano /etc/idmapd.conf
```


- fichier `/etc/exports`

```bash
# Pour ajouter un nouveau dossier /toto à partager, en autorisant le réseau `192.168.1.0/24` à l'utiliser
/toto 192.168.1.0/24(rw,no_root_squash)
```

Dans notre cas, vous n'ajouterez pas le dossier `/toto` à ce fichier, mais évidemment `/backup/web.tp6.linux/` et `/backup/db.tp6.linux/` (deux partages donc).  
Aussi, le réseau à autoriser n'est PAS `192.168.1.0/24` dans ce TP, à vous d'adapter la ligne.

Les machins entre parenthèses `(rw,no_root_squash)` sont les options de partage. **Vous expliquerez ce que signifient ces deux-là.**

🌞 **Démarrez le service**

- le service s'appelle `nfs-server`
- après l'avoir démarré, prouvez qu'il est actif
- faites en sorte qu'il démarre automatiquement au démarrage de la machine
```bash
[antho@backup ~]$ sudo systemctl start nfs-server
[sudo] Mot de passe de antho : 
[antho@backup ~]$ sudo systemctl status nfs-server
● nfs-server.service - NFS server and services
   Loaded: loaded (/usr/lib/systemd/system/nfs-server.service; disabled; vendor>
   Active: active (exited) since Tue 2021-12-07 23:20:11 CET; 11s ago
  Process: 5939 ExecStart=/bin/sh -c if systemctl -q is-active gssproxy; then s>
  Process: 5927 ExecStart=/usr/sbin/rpc.nfsd (code=exited, status=0/SUCCESS)
  Process: 5926 ExecStartPre=/usr/sbin/exportfs -r (code=exited, status=0/SUCCE>
 Main PID: 5939 (code=exited, status=0/SUCCESS)

déc. 07 23:20:11 backup.tp6.linux systemd[1]: Starting NFS server and services.>
déc. 07 23:20:11 backup.tp6.linux systemd[1]: Started NFS server and services.
lines 1-10/10 (END)
```
🌞 **Firewall**

- le port à ouvrir et le `2049/tcp`
- prouvez que la machine écoute sur ce port (commande `ss`)
```bash
[antho@backup ~]$ sudo ss -lntp | grep 2049
LISTEN 0      64           0.0.0.0:2049       0.0.0.0:*                                                                                                          
LISTEN 0      64              [::]:2049          [::]:*                                                                                                          
```
# Partie 3 : Setup des clients NFS : `web.tp6.linux` et `db.tp6.linux`

Ok, ça va être rapide cette partie : on va tester que `web.tp6.linux` et `db.tp6.linux` peuvent correctement accéder aux répertoires partagés par `backup.tp6.linux`.

---

On commence par `web.tp6.linux`.

🌞 **Install'**

- le paquet à install pour obtenir un client NFS c'est le même que pour le serveur : `nfs-utils`
```bash
[antho@web ~]$ sudo dnf install nfs-utils
```

🌞 **Conf'**

- créez un dossier `/srv/backup` dans lequel sera accessible le dossier partagé
- pareil que pour le serveur : fichier `/etc/idmapd.conf`

```bash
# Trouvez la ligne "Domain =" et modifiez la pour correspondre à notre domaine :
Domain = tp6.linux
```
```bash
[antho@web ~]$  sudo mkdir /srv/backup

[antho@web ~]$ cat /etc/idmapd.conf | grep Domain
Domain = tp6.linux
# the old method (comparing the domain in the string to the Domain value,
[antho@web ~]$ 
````


Eeeeet c'est tout ! Testons qu'on peut accéder au dossier partagé.  
Comment on fait ? Avec une commande `mount` !

Ui pareil qu'à la partie 1 ! Le dossier partagé sera vu comme une partition de type NFS.

La commande pour monter une partition en NFS :

```bash
$ sudo mount -t nfs <IP_SERVEUR>:</dossier/à/monter> <POINT_DE_MONTAGE>
```

Dans notre cas :

- le serveur NFS porte l'IP `10.5.1.13`
- le dossier à monter est `/backup/web.tp6.linux/`
- le point de montage, vous venez de le créer : `/srv/backup`
````bash
[antho@web ~]$ sudo mount -t nfs 10.5.1.13:/backup/web.tp6.linux /srv/backup
mount.nfs: No route to host
```
🌞 **Montage !**

- montez la partition NFS `/backup/web.tp6.linux/` avec une comande `mount`
  - la partition doit être montée sur le point de montage `/srv/backup`
  - preuve avec une commande `df -h` que la partition est bien montée
  - prouvez que vous pouvez lire et écrire des données sur cette partition
- définir un montage automatique de la partition (fichier `/etc/fstab`)
  - vous vérifierez que votre fichier `/etc/fstab` fonctionne correctement

---

🌞 **Répétez les opérations sur `db.tp6.linux`**

- le point de montage sur la machine `db.tp6.linux` est aussi sur `/srv/backup`
- le dossier à monter est `/backup/db.tp6.linux/`
- vous ne mettrez dans le compte-rendu pour `db.tp6.linux` que les preuves de fonctionnement :
  - preuve avec une commande `df -h` que la partition est bien montée
  - preuve que vous pouvez lire et écrire des données sur cette partition
  - preuve que votre fichier `/etc/fstab` fonctionne correctement

---

Final step : [mettre en place la sauvegarde, c'est la partie 4](./part4.md).
---
