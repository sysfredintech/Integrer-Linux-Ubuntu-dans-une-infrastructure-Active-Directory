# Intégrer des postes de travail Linux Ubuntu dans une infrastructure Active Directory avec gestion des roaming profiles et des quotas

**Dans ce lab, on utilisera un poste client Ubuntu 24.04 utilisant l'environnement de bureau Gnome avec Wayland, il est possible d'adapter cette procédure pour des distributions ou environnement de bureau différents**

## Objectifs

Intégrer des postes Linux dans un environnement Active Directory afin qu'un groupe d'utilisateurs défini puissent s'y connecter avec leur compte AD. Ces utilisateurs doivent retrouver leur profil d'un poste Linux à l'autre et également continuer à travailler si les serveurs AD ne sont pas disponibles (poste hors-ligne par exemple)

Les permissions d'accès aux dossiers et fichiers de ces utilisateurs doivent être sécurisés, donc visibles et modifiables que par eux-mêmes. (Et éventuellement les administrateurs)

La gestion de l'espace occupé par ces utilisateurs sur le serveur de fichiers doit être maîtrisé à travers l'utilisation de quotas d'espace disque.

## Contexte du lab

- 2 serveurs AD DC : WIN-KO477AGSO9G (192.168.10.28) et SRV-WIN22 (192.168.10.29)
- Nom du domaine : home.lab
- 1 serveur Samba sur Debian 13 comme membre de l'AD : SAMBASRV (192.168.10.245)
Pour l'installation du serveur Samba et sa jointure dans l'AD, voir l'article suivant: [Serveur de fichier Samba](https://www.monabad.fr/home/configurer-un-serveur-samba-comme-membre-dun-domaine-active-directory)
- 1 poste client Ubuntu 24.04 : UBUNTU-PC (DHCP)

## Sur le serveur Samba

### Configuration du service et du partage

1. Créer le dossier de partage pour les profiles utilisateurs Linux et lui affecter les bonnes permissions

```bash
mkdir /srv/shares/linux-profiles
chown -R root:"HOME\domain admins" /srv/shares/linux-profiles
chmod 2770 /srv/shares/linux-profiles
```

2. Configurer Samba

```bash
vim /etc/samba/smb.conf
```
```
[global]
    # ==================== Identification du Domaine ====================
    server role = member server
    workgroup = HOME
    realm = HOME.LAB
    security = ADS
    netbios name = SAMBASRV

    # ==================== Backend IDMAP (AUTORID) ====================
    idmap config * : backend = autorid
    idmap config * : range = 10000-999999
    idmap config * : rangesize = 100000

    # ==================== Paramètres d'Authentification ====================
    kerberos method = secrets and keytab
    dedicated keytab file = /etc/krb5.keytab
    winbind offline logon = false
    winbind enum users = yes
    winbind enum groups = yes
    winbind nss info = rfc2307
    winbind use default domain = yes
    winbind normalize names = yes

    # ==================== Performance et Logs ====================
    server string = %h samba files server
    log file = /var/log/samba/log.%m
    log level = 1
    max log size = 1000
    socket options = TCP_NODELAY SO_KEEPALIVE SO_RCVBUF=131072 SO_SNDBUF=131072

    # ==================== Polices de Sécurité ====================
    client use spnego = yes
    client ntlmv2 auth = yes
    server min protocol = SMB2_10
    client min protocol = SMB2_10
    server smb encrypt = desired
    vfs objects = acl_xattr
    map acl inherit = yes
    store dos attributes = yes
    dos filemode = yes
    nt acl support = yes

    # ============================Divers===========================
    template shell = /usr/bin/bash
    template homedir = /home/%U

    # =================== Définition des partages =================

 # On ajoute cette section pour les dossiers /home
 # Configuration minimale, les acls seront gérés depuis le serveur AD
[Linux-Profiles]
    path = /srv/shares/linux-profiles
    comment = Partage pour les homes Linux
    read only = no
```

```bash
systemctl restart smbd nmbd
```

## Sur le serveur Windows

Par mesure de simplicité, l'OU dédiée aux postes et utilisateurs Linux est placée à la racine du domaine dans ce lab mais il est possible d'adapter ce choix en fonction de l'organisation voulue.

1. **Créer une OU, y ajouter un groupe pour les postes Linux, un groupe pour les utilisateurs Linux et un utilisateur test membre du groupe**

Dans le gestionnaire de l'Active Directory
- Créer la nouvelle OU

![Config AD](./images/conf-ad1.png)
![Config AD](./images/conf-ad2.png)
- Créer le groupe pour les ordinateurs

![Config AD](./images/conf-ad3.png)
![Config AD](./images/conf-ad4.png)
- Créer le groupe pour les utilisateurs

![Config AD](./images/conf-ad5.png)
- Créer l'utilisateur

![Config AD](./images/conf-ad6.png)
![Config AD](./images/conf-ad7.png)
![Config AD](./images/conf-ad8.png)
- L'ajouter au groupe

![Config AD](./images/conf-ad9.png)
![Config AD](./images/conf-ad10.png)
- Définir sont dossier _home_

![Config AD](./images/conf-ad11.png)

Le dossier de l'utilisateur sera créé automatiquement par Windows avec les bonnes permissions grâce à l'ajout du paramètre _Home folder_

Ce dossier sera accessible par l'utilisateur via la lettre de lecteur choisie s'il se connecte sur un poste Windows, les fichiers et dossiers cachés tels que le _.bashrc_ seront traités comme des fichiers cachés par le poste Windows


2. **Donner les permissions adaptées au groupe d'ordinateurs et au groupe d'utilisateurs sur le partage**

Dans le gestionnaire d'ordinateurs (compmgmt.msc)

- Se connecter au serveur Samba

![Connexion au serveur Samba](./images/conf-pc1.png)
![Connexion au serveur Samba](./images/conf-pc2.png)

- Sélectionner le partage concerné

![Selection du partage Linux-Profiles](./images/conf-pc3.png)
- Changer les permissions à _Modify_ pour le groupe d'ordinateurs

![Security Advanced](./images/conf-pc4.png)
![Definition des permissions](./images/conf-pc5.png)
![Definition des permissions](./images/conf-pc6.png)
- Définir les permissions à Travers / List / Create folders pour le groupe utilisateurs

![Definition des permissions](./images/conf-pc7.png)
![Definition des permissions](./images/conf-pc8.png)
- Ne pas oublier de sélectionner _This folder only_

![Definition des permissions](./images/conf-pc9.png)


## Sur le poste Linux

**Sur une installation vierge d'Ubuntu 24.04 avec un utilisateur local (groupe sudo) que nous utiliserons pour la connexion ssh**

**L'installation doit être réalisée avec une partition séparée pour le montage de _/home_ afin de pouvoir gérer les quotas utilisateurs localement**

Exemple de partitionnement du disque pendant l'installation

![Install Ubuntu](./images/install-ubuntu-1.png)
![Install Ubuntu](./images/install-ubuntu-2.png)
![Install Ubuntu](./images/install-ubuntu-3.png)
![Install Ubuntu](./images/install-ubuntu-4.png)
![Install Ubuntu](./images/install-ubuntu-5.png)
![Install Ubuntu](./images/install-ubuntu-6.png)

1. **Préparation de l'environnement**

```bash
sudo -i
apt update && apt upgrade -y
```
Optionnel, les commandes _vim_ ci-dessous peuvent être remplacées par _nano_
```bash
apt install vim -y
```
Si l'on souhaite poursuivre la procédure en ssh et pour réaliser certains tests, installation et activation du serveur ssh (pourra être désactivée après la procédure)
```bash
apt install openssh-server -y
systemctl enable --now ssh
```


2. **Configuration des serveurs DNS**

```bash
vim /etc/resolv.conf
```
```
nameserver 127.0.0.53
nameserver 192.168.10.28
nameserver 192.168.10.29 # Optionnel
options edns0 trust-ad
search home.lab
```

3. **Basculer la gestion du réseau vers systemd-networkd**

Le service _systemd-networkd_ s'avère plus fiable concernant l'ordre de démarrage, la possibilité de conflits avec la configuration à mettre en place et la gestion des DNS, c'est donc un choix préférable

```bash
vim /etc/netplan/01-network-manager-all.yaml
```
```
# Let networkd manage all devices on this system
network:
  version: 2
  renderer: networkd
  ethernets:
    ens18: # remplacer par le nom de la carte
      dhcp4: true
      nameservers:
        addresses:
          - 192.168.10.28
          - 192.168.10.29
```
```bash
chmod 600 /etc/netplan/01-network-manager-all.yaml
netplan apply
```
**Le client obtiendra probablement une nouvelle adresse IP, si l'on est connecté en ssh, il faudra récupérer la nouvelle adresse et se reconnecter**

Configuration de systemd-resolved
```bash
vim /etc/systemd/resolved.conf
```
Modifier la section [Resolve] comme suit
```
DNS=192.168.10.28 192.168.10.29
FallbackDNS=192.168.10.254 9.9.9.9
Domains=home.lab
DNSStubListener=no
```

```bash
systemctl disable --now NetworkManager
```

Tester la résolution DNS
```bash
nslookup home.lab
```
Doit renvoyer
```
Server:		127.0.0.53
Address:	127.0.0.53#53

Non-authoritative answer:
Name:	home.lab
Address: 192.168.10.28
Name:	home.lab
Address: 192.168.10.29
```

4. **Configuration du service de temps**

Installation du paquet _chrony_
```bash
apt install chrony -y
vim /etc/chrony/chrony.conf
```
Commenter les lignes commençant par _pool_

Création d'un fichier de configuration pour utiliser le ou les serveurs AD comme serveurs de temps
```bash
vim /etc/chrony/conf.d/adserver.conf
```
Ajouter la ou les lignes suivantes
```
server  192.168.10.28   iburst
server  192.168.10.29   iburst # optionnel
```
```bash
systemctl restart chrony
chronyc sources -v
```
Doit afficher notre ou nos serveurs AD

5. **Installation des paquets nécessaire pour joindre le domaine**

```bash
apt install -y realmd sssd sssd-tools libnss-sss libpam-sss adcli samba-common-bin krb5-user krb5-config autofs cifs-utils keyutils
```
![Realm kerberos](./images/conf-ubuntu1.png)

6. **Jointure de la machine au domaine**

Vérifier que le domaine est joignable
```bash
realm discover home.lab
```
Rejoindre le domaine en plaçant le PC dans la bonne OU
```bash
realm join --computer-ou "OU=Linux,DC=home,DC=lab" --user=administrator home.lab
```
Vérifier la jointure
```bash
realm list
```
Doit renvoyer
```
home.lab
  type: kerberos
  realm-name: HOME.LAB
  domain-name: home.lab
  configured: kerberos-member
  server-software: active-directory
  client-software: sssd
  required-package: sssd-tools
  required-package: sssd
  required-package: libnss-sss
  required-package: libpam-sss
  required-package: adcli
  required-package: samba-common-bin
  login-formats: %U
  login-policy: allow-realm-logins
```

## Sur le serveur Windows


- Ajouter le PC Ubuntu au groupe "pc_linux"

![Group 1](./images/conf-ad-grp1.png)
![Group 2](./images/conf-ad-grp2.png)
![Group 3](./images/conf-ad-grp3.png)


## Sur le poste Linux

1. **Configuration de Kerberos**

```bash
vim /etc/krb5.conf
```
```
[libdefaults]
default_realm = HOME.LAB
        dns_lookup_realm = false
        dns_lookup_kdc = true
        ticket_lifetime = 24h
        renew_lifetime = 7d
        forwardable = true
        rdns = false
udp_preference_limit = 0

[realms]
    HOME.LAB = {
        kdc = WIN-KO477AGSO9G.home.lab
        kdc = SRV-WIN22.home.lab
        admin_server = WIN-KO477AGSO9G.home.lab
        default_domain = home.lab
    }

[domain_realm]
    .home.lan = HOME.LAB
    home.lan = HOME.LAB
```

2. **Configuration de SSSD**

```bash
vim /etc/sssd/sssd.conf
```
```
[sssd]
domains = home.lab
config_file_version = 2
services = nss, pam

[domain/home.lab]
default_shell = /bin/bash
krb5_store_password_if_offline = True
cache_credentials = True
krb5_realm = HOME.LAB
realmd_tags = manages-system joined-with-realmd
id_provider = ad
fallback_homedir = /home/%u
override_homedir = /home/%u
create_homedir = true
homedir_umask = 0077
ad_domain = home.lab
use_fully_qualified_names = False
ldap_id_mapping = True
access_provider = ad
auth_provider = ad
chpass_provider = ad
dyndns_update = false
#Les GPO peuvent poser problème donc on ajoute les 2 lignes suivantes
ad_gpo_access_control = disabled
ad_gpo_ignore_unreadable = true

krb5_canonicalize = false
krb5_use_enterprise_principal = false

[pam]
offline_credentials_expiration = 3
```
Cette configuration permettra aux utilisateurs de se connecter avec leurs identifiants AD même si le poste est hors ligne pendant une durée de trois jours.
```bash
systemctl restart sssd
```
Désactiver les sockets SSSD suivantes pour éviter les conflits
```bash
systemctl disable --now sssd-nss.socket sssd-pam-priv.socket sssd-pam.socket
```

3. **Mise en place des quotas sur la partition _/home_ afin de prévenir les erreurs de synchronisation ultérieures avec le dossier distant**

Création d'un utilisateur temporaire
```bash
useradd -M tempo -s /bin/bash -G sudo
passwd tempo
```
Se déconnecter puis se reconnecter avec l'utilisateur temporaire
```
sudo -i
fuser -km /home/fred # A adapter
apt install quota -y
umount /home
tune2fs -O quota /dev/sda4 # sda4 est la partition du /home ici
tune2fs -Q usrquota,grpquota /dev/sda4
vim /etc/fstab
```
Modifier la ligne correspondante au montage de /home pour y ajouter les options usrquota et grpquota
```
/dev/disk/by-uuid/534d04fd-2d43-4549-a8f7-3ee5ebb97891 /boot ext4 defaults,usrquota,grpquota 0 1
```
```bash
systemctl daemon-reload
mount -av
```
Vérifier que la commande _mount_ ne renvoi pas d'erreur

Se déconnecter puis se reconnecter avec l'utilisateur initial
```bash
sudo -i
userdel tempo
```

4. **Configuration du module PAM pour l'authentification**

Créer un script PAM pour le montage à la connexion et le démontage à la déconnexion du _/home_ des utilisateurs avec une synchronisation des données pour les connexions hors ligne.

Si l'utilisateur se connecte sur le poste alors que les serveurs AD sont indisponibles, il retrouvera le contenu de son _/home_ tel qu'il était à la dernière déconnexion.

De même, si l'utilisateur travail hors ligne, lors de la reconnexion sur le domaine, ses données seront synchronisées avec le serveur distant.

_Il faut adapter la valeur de la variable SAMBA_PROFILE selon le chemin vers le partage_
```bash
vim /usr/local/bin/setup_home.sh
```
```bash
#!/bin/bash
# Logs
LOG_FILE=/var/log/pam-script.log
if [ ! -e "$LOG_FILE" ]; then
touch "$LOG_FILE"
fi
exec 3>&1 4>&2
trap 'exec 2>&4 1>&3' 0 1 2 3
exec 1>"$LOG_FILE" 2>&1
# Arrêt du script si utilisateur local
USER_UID=$(id -u "$PAM_USER" 2>/dev/null)
if [ -z "$USER_UID" ] || [ "$USER_UID" -lt 10000 ]; then
    exit 0
fi

# Montage des dossiers utilisateurs depuis sambasrv vers le /home local
# dynperm peut être supprimé pour empêcher les utilisateurs de modifier les permissions dans leur /home et de rendre executable un fichier
if [ "$PAM_TYPE" == "open_session" ]; then
SAMBA_PROFILE="//sambasrv/linux-profiles/$PAM_USER"
LOCAL_PROFILE="/home/$PAM_USER"
# Synchronisation des données depuis le /home local vers le dossier distant via /tmp
    if [ -d "$LOCAL_PROFILE" ]; then
    TMP_DIR="/tmp/$PAM_USER"
    mkdir -p "$TMP_DIR"
    rsync -ar --copy-links --exclude='.cache' "$LOCAL_PROFILE/" "$TMP_DIR/"
    fi
mount -v -t cifs "$SAMBA_PROFILE" "$LOCAL_PROFILE" -o sec=krb5,vers=3.0,uid=$USER_UID,gid=$USER_UID,file_mode=0600,dir_mode=0700,cruid=$USER_UID,dynperm,nobrl,mfsymlinks
if [ -d "$TMP_DIR" ]; then
rsync -aru --copy-links "$TMP_DIR/" "$LOCAL_PROFILE/"
rm -rf $TMP_DIR
fi
exit 0

# Démontage du dossier /home à la déconnexion
elif [ "$PAM_TYPE" == "close_session" ]; then
LOCAL_PROFILE="/home/$PAM_USER"
# Synchronisation des données depuis le dossier distant vers le /home local via /tmp
TMP_DIR="/tmp/$PAM_USER"
mkdir -p "$TMP_DIR"
rsync -ar --copy-links --exclude='.cache' "$LOCAL_PROFILE/" "$TMP_DIR/"
USE_MOUNT=$(fuser -m "$LOCAL_PROFILE")
    while [ -n "$USE_MOUNT" ]; do
            fuser -km "$LOCAL_PROFILE"
            USE_MOUNT=$(fuser -m "$LOCAL_PROFILE")
    done
umount -f -l "$LOCAL_PROFILE"
rsync -aru --copy-links "$TMP_DIR/" "$LOCAL_PROFILE/"
rm -rf $TMP_DIR
exit 0
fi
```

```bash
chmod +x /usr/local/bin/setup_home.sh
```

**Il faut s'assurer que l'utilisateur gdm ait les permissions nécessaires sur le dossier /home de façon récursive pour appliquer les paramètres utilisateur à l'ouverture de session**
```bash
setfacl -R -m u:gdm:rwX /home
```

Modifier la configuration PAM pour l'ensemble des modes de connexions
```bash
vim /etc/pam.d/common-session
```
Configurer le bloque additionnel comme suit et adapter les valeurs de quotas hard et soft en fonction des besoins (valeur en Ko)

Ces valeurs devront être reportées dans le script de définition des quotas pour les utilisateurs du groupe AD sur le serveur Samba que nous feront par la suite
```
# and here are more per-package modules (the "Additional" block)
session required        pam_unix.so
session optional        pam_sss.so
session optional        pam_mkhomedir.so
session optional        pam_exec.so     /usr/local/bin/setup_home.sh
session optional        pam_setquota.so bsoftlimit=10485760 bhardlimit=11534336 startuid=10000 enduid=9999999999 overwrite=1
session optional        pam_systemd.so
# end of pam-auth-update config
```


5. **Mise en place de la configuration de Wayland via dconf pour le mappage du clavier pour tous les utilisateurs (AD inclus)**

```bash
vim /etc/dconf/profile/user
```
```
user-db:user
system-db:local
```
```bash
mkdir /etc/dconf/db/local.d
vim /etc/dconf/db/local.d/01-keyboard
```
```
[org/gnome/desktop/input-sources]
sources=[('xkb', 'fr')]
mru-sources=[('xkb', 'fr')]

[org/gnome/desktop/peripherals/keyboard]
remember-numlock-state=true
```

```bash
dconf update
```
On peut vérifier que la configuration est bien appliquée avec cette commande
```bash
dconf dump /org/gnome/desktop/input-sources/
```
Doit renvoyer
```
[/]
mru-sources=[('xkb', 'fr')]
sources=[('xkb', 'fr+latin9')]
xkb-options=@as []
```

Le démon ibus peut générer un crash à l'ouverture de la première session et il ne sera d'aucune utilité dans une utilisation standard du poste, il est possible de le désactiver
```bash
systemctl --global mask ibus.service
systemctl --global mask ibus-x11.service
```

6. **Remplacer l'explorateur de fichiers natif de Gnome qui pose des problèmes de gestion de base de données lors du passage d'un poste Linux à l'autre par l'explorateur de fichiers Nemo qui s'intègre très bien dans l'environnement Gnome**

Désinstallation complète de Nautilus et installation de Nemo avec l'intégralité des plugins
```bash
apt remove --purge nautilus* -y
apt install nemo* -y
apt autoremove -y
```

Création des lanceurs épinglés dans la barre d'applications pour tous les utilisateurs, à adapter en fonctions des logiciels disponibles et des besoins
```bash
vim /etc/dconf/db/local.d/20-nemo-fav
```
```
[org/gnome/shell]
favorite-apps=['firefox.desktop', 'nemo.desktop', 'org.gnome.Calculator.desktop']
```
Pour les applications installées avec snap, comme Firefox, il faudra copier les fichiers .desktop correspondants dans _/usr/share/applications/_ et modifier le lanceur pour afficher l'icône
```bash
cp /snap/firefox/6565/firefox.desktop /usr/share/applications/
sed -i 's/Icon=\/default256.png/Icon=\/snap\/firefox\/6565\/default256.png/g' /usr/share/applications/firefox.desktop
```
```bash
dconf update
```

## Redémarrage du poste Linux et tests

1. **Redémarrer la machine et s'assurer qu'on a pas d'erreur au boot**

```bash
sudo -i
journalctl -b
```
S'assurer que les quotas activés
```
quotaon /home
```

2. **Ouvrir une session avec l'utilisateur de l'AD**

Pour surveiller le bon déroulement de l'ouverture de session et de l'exécution du script PAM, il est possible de lancer les commandes suivante avec le compte utilisateur local en ssh
```bash
sudo -i
tail -f /var/log/auth.log
tail -f /var/log/pam-script.log
```

Cliquez sur "absent de la liste" pour entrer manuellement le login et le mot de passe, nous avons configuré sssd pour utiliser les noms courts donc on ne saisit que le nom de l'utilisateur sans le @home.lab

![open session](./images/open-session-1.png)

Depuis la connexion ssh, vérifier le montage du dossier utilisateur
```bash
mount | grep tux
```
Doit renvoyer
```
//sambasrv/linux-profiles/tux on /home/tux type cifs (rw,relatime,vers=3.0,sec=krb5,cruid=1031601244,cache=strict,upcall_target=app,username=tux,uid=1031601244,forceuid,gid=1031601244,forcegid,addr=192.168.10.245,file_mode=0600,dir_mode=0700,soft,nounix,serverino,mapposix,dynperm,reparse=nfs,nativesocket,symlink=native,rsize=4194304,wsize=4194304,bsize=1048576,retrans=1,echo_interval=60,actimeo=1,closetimeo=1)
```

Lancer le gestionnaire de fichier pour constater la création des éléments pour l'utilisateur

![home dir](./images/home-dir1.png)

Ouvrir un terminal pour vérifier la disposition du clavier et vérifier les permissions
```bash
ls -la ~/
```
Doit renvoyer
```
total 28
drwx------  2 tux  1031601244    0 janv. 21 19:07 .
drwxr-xr-x+ 6 root root       4096 janv. 22 15:02 ..
-rw-------  1 tux  1031601244  571 janv. 22 16:18 .bash_history
drwx------  2 tux  1031601244    0 janv. 21 17:38 Bureau
drwx------  2 tux  1031601244    0 janv. 22 14:53 .cache
drwx------  2 tux  1031601244    0 janv. 22 15:51 .config
drwx------  2 tux  1031601244    0 janv. 22 16:13 Documents
drwx------  2 tux  1031601244    0 janv. 21 17:38 Images
drwx------  2 tux  1031601244    0 janv. 21 17:37 .local
drwx------  2 tux  1031601244    0 janv. 21 17:38 Modèles
drwx------  2 tux  1031601244    0 janv. 21 17:38 Musique
drwx------  2 tux  1031601244    0 janv. 21 17:38 Public
drwx------  2 tux  1031601244    0 janv. 21 18:00 snap
drwx------  2 tux  1031601244    0 janv. 21 17:38 Téléchargements
drwx------  2 tux  1031601244    0 janv. 21 17:38 Vidéos
-rw-------  1 tux  1031601244 9190 janv. 21 19:07 .viminfo
```

Créer un fichier test dans le /home

![test file](./images/test-file.png)

Depuis la connexion ssh avec le compte local, vérifier la bonne application des quotas pour l'utilisateur AD sur la partition _/home_
```bash
repquota -s /home
```
Doit renvoyer
```
*** Rapport pour les quotas user sur le périphérique /dev/sda4
Période de sursis bloc : 7days ; période de sursis inode : 7days
                        Space limits                File limits
Utilisateur     utilisé souple stricte sursis utilisé souple stricte sursis
----------------------------------------------------------------------
root      --     20K      0K      0K              2     0     0       
fred      --  14728K      0K      0K            356     0     0       
#1031601244 --   5932K  10240M  11264M            449     0     0
```


## Sur le serveur Samba

1. **Vérifier la présence du fichier créé et les permissions**

Vérifier les acls sur le dossier de l'utilisateur
```bash
sudo -i
getfacl /srv/shares/linux-profiles/tux
```
Doit renvoyer ceci
```
getfacl: Removing leading '/' from absolute path names
# file: srv/shares/linux-profiles/tux
# owner: root
# group: domain_admins
# flags: -s-
user::rwx
user:root:rwx
user:domain_admins:rwx
user:pc_linux:rwx
user:tux:rwx
group::rwx
group:BUILTIN\\administrators:rwx
group:domain_admins:rwx
group:pc_linux:rwx
group:tux:rwx
mask::rwx
other::---
default:user::rwx
default:user:root:rwx
default:user:domain_admins:rwx
default:user:pc_linux:rwx
default:user:tux:rwx
default:group::r-x
default:group:BUILTIN\\administrators:rwx
default:group:domain_admins:rwx
default:group:pc_linux:rwx
default:group:tux:rwx
default:mask::rwx
default:other::---
```

Vérifier la présence du fichier précédemment créé par l'utilisateur
```bash
ls -la /srv/shares/linux-profiles/tux/Documents/
```
Doit renvoyer ceci
```
total 20
drwxrws---+  2 root domain_admins 4096 Jan 22 16:13 ./
drwxrws---+ 14 root domain_admins 4096 Jan 21 19:07 ../
-rwxrwx---+  1 tux  domain_users     0 Jan 22 16:13 test.txt*
```

2. **Créer un script pour la gestion des quotas pour les utilisateurs du groupe users_linux**

Reporter les valeurs définis sur le dossier _/home_ du poste Linux afin d'éviter des erreurs de synchronisation des données
```bash
vim /usr/local/bin/linux-profiles-quotas.sh
```
```bash
#!/bin/bash
# Ajuster les limites soft et hard en fonctions des besoins et la variable SHARE en fonction de la nomenclature
SOFT="10G"
HARD="11G"
SHARE="/srv/"
# Lister les utilisateurs du groupe users_linux
LIST=$(net rpc group members users_linux -P -S 192.168.10.28 | awk -F '\\' '{print $2}' || net rpc group members users_linux -P -S 192.168.10.29 | awk -F '\\' '{print $2}')
# Récupérer les UID des utilisateurs
USER_IDS=$(for users in $LIST; do id -u $users; done)
# Appliquer les quotas sur tous les utilisateurs du groupe
for users in $USER_IDS; do
setquota -u $users "$SOFT" "$HARD" 0 0 "$SHARE"
done
```
```bash
chmod +x /usr/local/bin/linux-profiles-quotas.sh
```

L'exécution du script appliquera les quotas pour tous les utilisateurs du groupe, il est possible de planifier son lancement de manière périodique via la crontab de root
```bash
crontab -e
```
```
# Pour une exécution tous les jours ouvrés à 1h00
0	1 	*	*	1-5	/usr/local/bin/linux-profiles-quotas.sh
```

## Tester les accès utilisateurs depuis le poste Linux

1. **Se connecter avec l'utilisateur local en ssh et tenter d'accéder au dossier de l'utilisateur de l'AD**

```bash
fred@ubuntu-pc:~$ ls -la /home/tux
ls: impossible d'ouvrir le répertoire '/home/tux': Permission non accordée
```

2. **Fermer la session de l'utilisateur AD et vérifier le démontage de son _home_**

```bash
mount | grep tux
```
Ne doit rien renvoyer

3. **Créer un utilisateur supplémentaire dans l'AD de la même façon que le premier et se connecter sur le poste Linux**

![new user ad](./images/new-ad-user.png)
![new user ad](./images/new-ad-user2.png)

Vérifier le montage du _home_, son contenu et les permissions

![new user ad](./images/new-ad-user3.png)

Refaire les étapes de vérifications vues précédemment sur le poste Linux concernant le démontage du _home_ et sur le serveur Samba concernant les permissions ainsi que l'application du script de gestion des quotas

## Conclusion

Nous avons maintenant un écosystème prêt à recevoir des postes de travail Ubuntu dans une infrastructure Active Directory avec la prise en charge des profils itinérants pour les utilisateurs et le stockage de leurs données et de leurs quotas sur un serveur de fichier Samba.
