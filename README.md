# 📋 Projet Infrastructure ESNlearn

## 🎯 Contexte du Projet

Vous êtes consultant dans une **SSII (Société de Services en Ingénierie Informatique)** et vous êtes dépêché chez le client **"ESNlearn"**.

### Problématiques Actuelles
- ❌ Serveur web vieillissant sous Windows Server 2003
- ❌ Serveur DNS à remplacer
- ❌ Absence de gestion centralisée
- ❌ Pas de sécurisation HTTPS

---

## 📊 Besoins du Client

### 1. **Gestion Centralisée**
Mise en place d'un serveur central pour gérer :
- Les comptes utilisateurs
- Les droits d'accès
- Les fichiers partagés
- L'authentification

**Solution** : Samba 4 AD DC (Linux) ou Active Directory (Windows)

### 2. **Remplacement du Serveur Web**
- **Ancien** : Windows Server 2003 + IIS
- **Nouveau** : Linux + Apache/Nginx + PHP + MySQL/PostgreSQL

### 3. **Serveur DNS Open Source**
- Solution : **BIND9** ou **dnsmasq**
- Intégration DDNS via redirecteur

### 4. **Autorité de Certification (Optionnel)**
- Génération de certificats SSL
- Sécurisation HTTPS du site web
- Solution : **OpenSSL** ou **Let's Encrypt**

---

## 🏗️ Architecture à Implémenter

### 1. Infrastructure Réseau et Nomenclature

#### Tableau d'Adressage IP

| Serveur | Rôle | IP | Sous-réseau | Gateway |
|---------|------|-------|-------------|---------|
| ubuntuserver | DC + DNS + SAN (Multi-rôle) | 192.168.10.193 | 255.255.255.128 | 192.168.10.129 |
| srv2 | Serveur Web (Nginx + PHP + MySQL) | 192.168.10.194 | 255.255.255.128 | 192.168.10.129 |

#### Schéma d'Infrastructure

```
                        INTERNET
                            │
                            ▼
                ┌──────────────────┐
                │  Routeur/Firewall│
                │  192.168.10.129  │
                └─────────┬────────┘
                          │
            ┌─────────────┴─────────────┐
            │                           │
            ▼                           ▼
  ┌──────────────────┐        ┌──────────────────┐
  │  ubuntuserver    │        │      srv2        │
  │  192.168.10.193  │◄──────►│  192.168.10.194  │
  │                  │        │                  │
  │ ┌──────────────┐ │        │ ┌──────────────┐ │
  │ │  Samba 4 AD  │ │        │ │    Nginx     │ │
  │ │  (DC + LDAP) │ │        │ │   HTTPS      │ │
  │ └──────────────┘ │        │ └──────────────┘ │
  │ ┌──────────────┐ │        │ ┌──────────────┐ │
  │ │   BIND9      │ │        │ │  PHP 8.x     │ │
  │ │   DNS/DDNS   │ │        │ │  PHP-FPM     │ │
  │ └──────────────┘ │        │ └──────────────┘ │
  │ ┌──────────────┐ │        │ ┌──────────────┐ │
  │ │   RAID 5     │ │        │ │   MySQL      │ │
  │ │   4x20Go     │ │        │ │   Database   │ │
  │ │   /mnt/san   │ │        │ └──────────────┘ │
  │ └──────────────┘ │        └──────────────────┘
  │ ┌──────────────┐ │               
  │ │  Partages    │ │        (Accède aux partages  
  │ │   Samba      │ │         Samba via réseau)
  │ └──────────────┘ │               
  └──────────────────┘

  Serveur Multi-Rôles          Serveur Web Dédié
```

**Avantages de cette architecture** :
- ✅ Réduction des coûts (2 serveurs au lieu de 4)
- ✅ Simplification de la gestion
- ✅ Centralisation des services d'infrastructure
- ✅ Séparation du serveur web pour la sécurité
- ✅ Facilité de sauvegarde

---

## 🖥️ Configuration des Serveurs

### 2. Configuration du Serveur ubuntuserver (DC + DNS + SAN)

**Serveur** : `ubuntuserver` (192.168.10.193)
**Rôles** : 
- Contrôleur de Domaine (Samba 4 AD DC)
- Serveur DNS/DDNS (BIND9)
- Serveur de Stockage SAN (RAID 5)
- Serveur de Fichiers (Partages Samba)

---

#### A. Installation et Configuration DNS (BIND9)

**Objectif** : Configurer le serveur DNS pour la résolution des noms de domaine.

**Étape 1 : Installation de BIND9**

```bash
# Mise à jour du système
sudo apt update && sudo apt upgrade -y

# Installation de BIND9 et outils
sudo apt install -y bind9 bind9utils bind9-doc dnsutils

# Vérifier l'installation
named -v
# Résultat attendu : BIND 9.x.x
```

**Étape 2 : Configuration du redirecteur DNS**

```bash
# Éditer la configuration des options
sudo nano /etc/bind/named.conf.options
```

#### Configuration avec Redirecteur

```bash
# /etc/bind/named.conf.options
options {
    directory "/var/cache/bind";
    
    # Redirecteurs DNS externes
    forwarders {
        8.8.8.8;        # Google DNS
        1.1.1.1;        # Cloudflare DNS
    };
    
    forward only;
    
    dnssec-validation auto;
    listen-on-v6 { any; };
};
```

#### Configuration Zone DDNS

**Objectif** : Configurer les zones DNS pour permettre les mises à jour dynamiques (DDNS).

**Étape 1 : Éditer le fichier de configuration des zones locales**

```bash
# Ouvrir le fichier de configuration
sudo nano /etc/bind/named.conf.local
```

**Étape 2 : Ajouter la configuration des zones**

```bash
# /etc/bind/named.conf.local

# Zone de résolution directe (Forward Zone)
zone "esnlearn.lab" {
    type master;                              # Type de zone : master (primaire)
    file "/var/lib/bind/db.esnlearn.lab";   # Chemin du fichier de zone
    notify yes;                               # Notifier les serveurs secondaires
};

# Zone de résolution inverse (Reverse Zone)
zone "10.168.192.in-addr.arpa" {
    type master;
    file "/var/lib/bind/rev.esnlearn.lab";
    notify yes;
};
```

**Étape 3 : Sauvegarder et vérifier la syntaxe**

```bash
# Sauvegarder : Ctrl+O puis Entrée, puis Ctrl+X pour quitter

# Vérifier la syntaxe de la configuration
sudo named-checkconf /etc/bind/named.conf.local
# Si aucune erreur n'est affichée, la configuration est correcte
```

**Explication des paramètres** :
- `type master` : Ce serveur est autoritaire pour cette zone
- `file` : Emplacement du fichier de zone (dans `/var/lib/bind`)
- `notify yes` : Notifie les serveurs DNS secondaires des changements

---

#### Création du Fichier de Zone

**Objectif** : Créer le fichier de zone DNS avec les enregistrements initiaux.

**Étape 1 : Créer le répertoire et définir les permissions**

```bash
# Créer le répertoire pour les zones dynamiques
sudo mkdir -p /var/lib/bind

# Définir les permissions appropriées
sudo chown bind:bind /var/lib/bind
sudo chmod 775 /var/lib/bind
```

**Étape 2 : Créer le fichier de zone forward**

```bash
# Créer et éditer le fichier de zone
sudo nano /var/lib/bind/db.esnlearn.lab
```

**Contenu du fichier de zone** :

```bind
$TTL    604800
@       IN      SOA     ubuntuserver.esnlearn.lab. root.esnlearn.lab. (
                              1         ; Serial (incrémenter à chaque modification)
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
; Serveurs de noms
@       IN      NS      ubuntuserver.esnlearn.lab.

; Enregistrements A
ubuntuserver   IN      A       192.168.10.193
srv2    IN      A       192.168.10.194

; Alias (CNAME)
www     IN      CNAME   srv2
```

**Étape 3 : Sauvegarder et définir les permissions**

```bash
# Sauvegarder le fichier (Ctrl+O, Entrée, Ctrl+X)

# Définir le propriétaire et les permissions
sudo chown bind:bind /var/lib/bind/db.esnlearn.lab
sudo chmod 644 /var/lib/bind/db.esnlearn.lab
```

**Étape 4 : Créer le fichier de zone inverse (Reverse Zone)**

```bash
# Créer le fichier de zone inverse
sudo nano /var/lib/bind/rev.esnlearn.lab
```

**Contenu du fichier de zone inverse** :

```bind
$TTL    604800
@       IN      SOA     ubuntuserver.esnlearn.lab. admin.esnlearn.lab. (
                              2024112101 ; Serial (même numéro que la zone forward)
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL

; Serveur de noms
@       IN      NS      ubuntuserver.esnlearn.lab.

; Enregistrements PTR (Pointer) pour la résolution inverse
; Format : dernier octet de l'IP IN PTR nom-complet-avec-point-final
193     IN      PTR     ubuntuserver.esnlearn.lab.  ; 192.168.10.193 (DC + DNS + SAN)
194     IN      PTR     srv2.esnlearn.lab.          ; 192.168.10.194 (Serveur Web)
```

**Étape 5 : Définir les permissions de la zone inverse**

```bash
sudo chown bind:bind /var/lib/bind/rev.esnlearn.lab
sudo chmod 644 /var/lib/bind/rev.esnlearn.lab
```

**Étape 6 : Vérifier la syntaxe des fichiers de zone**

```bash
# Vérifier la zone forward
sudo named-checkzone esnlearn.lab /var/lib/bind/db.esnlearn.lab

# Résultat attendu :
# zone esnlearn.lab/IN: loaded serial 2024112101
# OK

# Vérifier la zone inverse
sudo named-checkzone 10.168.192.in-addr.arpa /var/lib/bind/rev.esnlearn.lab

# Résultat attendu :
# zone 10.168.192.in-addr.arpa/IN: loaded serial 2024112101
# OK
```

**Étape 7 : Redémarrer et activer BIND9**

```bash
# Redémarrer le service BIND9
sudo systemctl restart bind9

# Activer le démarrage automatique
sudo systemctl enable bind9

# Vérifier le statut du service
sudo systemctl status bind9
```

---

#### B. Configuration du Serveur SAN (RAID 5)

**Configuration RAID 5 avec 4 partitions de 5 Go**

**1. Lister les disques et créer les partitions**

```bash
lsblk
sudo fdisk /dev/sdb
```

Créer 4 partitions : `n` → `p` → numéro → `+5G`
Changer le type en RAID : `t` → numéro → `fd`
Sauvegarder : `w`

**📸 Screenshot : Résultat de `lsblk` montrant les 4 partitions**

---

**2. Installer mdadm et créer le RAID 5**

```bash
sudo apt install -y mdadm
sudo mdadm --create --verbose /dev/md0 --level=5 --raid-devices=4 /dev/sdb1 /dev/sdb2 /dev/sdb3 /dev/sdb4
cat /proc/mdstat
```

**📸 Screenshot : État du RAID avec `cat /proc/mdstat`**

---

**3. Formater et monter le RAID**

```bash
sudo mkfs.ext4 /dev/md0
sudo mkdir -p /mnt/san/lun1
sudo mount /dev/md0 /mnt/san/lun1
df -h | grep md0
```

**📸 Screenshot : Montage du RAID**

---

**4. Configuration du montage automatique**

```bash
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
sudo nano /etc/fstab
```

Ajouter : `/dev/md0  /mnt/san/lun1  ext4  defaults,nofail  0  2`

```bash
sudo update-initramfs -u
```

---

**5. Créer la structure des partages**

```bash
sudo mkdir -p /mnt/san/lun1/{direction,comptabilite,informatique,commun}
ls -la /mnt/san/lun1/
```

**📸 Screenshot : Structure des répertoires créés**

**Étape 11 : Configuration du monitoring RAID**

```bash
# Configurer les alertes email (optionnel)
sudo dpkg-reconfigure mdadm

# Questions posées :
# - Faut-il surveiller le RAID ? Oui
# - Email pour les alertes : votre@email.com
# - Intervalle de vérification : daily

# Activer le monitoring
sudo systemctl enable mdmonitor
sudo systemctl start mdmonitor

# Vérifier le service
sudo systemctl status mdmonitor
```

**📊 Monitoring et Maintenance du RAID**

```bash
# Vérifier l'état du RAID en temps réel
watch -n 5 cat /proc/mdstat

# Afficher les détails complets
sudo mdadm --detail /dev/md0

# Vérifier l'intégrité (scrubbing)
echo check > /sys/block/md0/md/sync_action

# Surveiller la vérification
cat /sys/block/md0/md/sync_action
```

**⚠️ Simulation de panne (OPTIONNEL - À faire seulement en test)**

```bash
# Simuler une panne de disque
sudo mdadm --manage /dev/md0 --fail /dev/sdb2

# Vérifier l'état (le RAID doit rester fonctionnel)
cat /proc/mdstat
sudo mdadm --detail /dev/md0

# Retirer le disque défaillant
sudo mdadm --manage /dev/md0 --remove /dev/sdb2

# Réinsérer le disque (reconstruction automatique)
sudo mdadm --manage /dev/md0 --add /dev/sdb2

# Surveiller la reconstruction
watch -n 1 cat /proc/mdstat
```

**✅ Le serveur SAN est maintenant configuré et prêt pour les partages Samba !**

---

#### C. Installation et Configuration Samba 4 AD DC

**Sur le serveur** : `ubuntuserver` (192.168.10.193)

**Objectif** : Installer et configurer Samba 4 comme contrôleur de domaine Active Directory.

**Prérequis** :
- Système mis à jour
- Nom d'hôte configuré correctement (dc01.esnlearn.lab)
- IP statique configurée (192.168.10.193)

**Étape 1 : Vérifier le nom d'hôte (déjà configuré)**

```bash
# Vérifier le nom d'hôte actuel
hostname -f
# Doit afficher : ubuntuserver.esnlearn.lab

# Vérifier /etc/hosts
cat /etc/hosts
# Doit contenir les deux serveurs :
# 192.168.10.193  ubuntuserver.esnlearn.lab ubuntuserver
# 192.168.10.194  srv2.esnlearn.lab srv2
```

**Étape 2 : Mettre à jour le système**

```bash
# Mise à jour complète
sudo apt update && sudo apt upgrade -y

# Redémarrer si nécessaire
sudo reboot
```

**Étape 3 : Installation des paquets Samba**

```bash
# Installer Samba et les outils nécessaires
sudo apt install -y samba smbclient winbind krb5-user libpam-winbind libnss-winbind

# Pendant l'installation, krb5-user demandera :
# - Default Kerberos version 5 realm: ESNLEARN.LAB
# - Kerberos servers for your realm: dc01.esnlearn.lab
# - Administrative server for your Kerberos realm: dc01.esnlearn.lab
```

**Étape 4 : Arrêter et désactiver les services par défaut**

```bash
# Arrêter les services Samba standards
sudo systemctl stop smbd nmbd winbind

# Désactiver leur démarrage automatique
sudo systemctl disable smbd nmbd winbind

# Vérifier qu'ils sont bien arrêtés
sudo systemctl status smbd nmbd winbind
```

**Explication** : Les services `smbd`, `nmbd` et `winbind` sont pour Samba en mode serveur de fichiers classique. En mode AD DC, on utilisera le service `samba-ad-dc` à la place
```

#### Promotion en Contrôleur de Domaine

**Objectif** : Provisionner le domaine Active Directory avec Samba.

**Étape 1 : Sauvegarder la configuration par défaut**

```bash
# Sauvegarder le fichier de configuration original
sudo mv /etc/samba/smb.conf /etc/samba/smb.conf.bak

# Vérifier que le fichier a été renommé
ls -la /etc/samba/
```

**Étape 2 : Provisionner le domaine Active Directory**

```bash
# Lancer le provisionnement sur ubuntuserver
sudo samba-tool domain provision \
    --use-rfc2307 \
    --realm=ESNLEARN.LAB \
    --domain=ESNLEARN \
    --adminpass='P@ssw0rd123!' \
    --server-role=dc \
    --dns-backend=SAMBA_INTERNAL \
    --host-ip=192.168.10.193 \
    --host-name=ubuntuserver
```

**Explication des paramètres** :
- `--use-rfc2307` : Active les attributs POSIX (UID/GID) pour compatibilité Unix/Linux
- `--realm=ESNLEARN.LAB` : Nom du domaine Kerberos (TOUJOURS EN MAJUSCULES)
- `--domain=ESNLEARN` : Nom NetBIOS du domaine (15 caractères max)
- `--adminpass` : Mot de passe de l'administrateur (complexité requise)
- `--server-role=dc` : Rôle du serveur (dc = Domain Controller)
- `--dns-backend=SAMBA_INTERNAL` : Utiliser le DNS interne de Samba
- `--host-ip` : Adresse IP du contrôleur de domaine

**⚠️ Politique de mot de passe** :
Le mot de passe administrateur doit respecter :
- Au moins 8 caractères
- Majuscules + minuscules + chiffres + caractères spéciaux
- Ne pas contenir le nom d'utilisateur

**Résultat attendu** :
```
Looking up IPv4 addresses
Looking up IPv6 addresses
No IPv6 address will be assigned
Setting up secrets.ldb
Setting up the registry
Setting up the privileges database
Setting up idmap db
Setting up SAM db
Setting up sam.ldb partitions and settings
Setting up sam.ldb rootDSE
Pre-loading the Samba 4 and AD schema
Adding DomainDN: DC=esnlearn,DC=lab
Adding configuration container
Setting up sam.ldb schema
Setting up sam.ldb configuration data
Setting up display specifiers
Adding users container
Modifying users container
Adding computers container
Modifying computers container
Setting up sam.ldb data
Setting up well known security principals
Setting up sam.ldb users and groups
Setting up self join
Adding DNS accounts
Creating CN=MicrosoftDNS,CN=System,DC=esnlearn,DC=lab
Creating DomainDnsZones and ForestDnsZones partitions
Populating DomainDnsZones and ForestDnsZones partitions
Setting up sam.ldb rootDSE marking as synchronized
Fixing provision GUIDs
A Kerberos configuration suitable for Samba AD has been generated at /var/lib/samba/private/krb5.conf
Setting up fake yp server settings
Once the above files are installed, your Samba AD server will be ready to use
Server Role:           active directory domain controller
Hostname:              ubuntuserver
NetBIOS Domain:        ESNLEARN
DNS Domain:            esnlearn.lab
FOREST Level:          2008_R2
DOMAIN Level:          2008_R2
ADMINISTRATOR password: [HIDDEN]
```

**Étape 3 : Copier la configuration Kerberos générée**

```bash
# Sauvegarder le fichier krb5.conf existant
sudo mv /etc/krb5.conf /etc/krb5.conf.bak

# Copier la configuration générée par Samba
sudo cp /var/lib/samba/private/krb5.conf /etc/krb5.conf

# Vérifier le contenu
cat /etc/krb5.conf
```

**Étape 4 : Démarrer le service Samba AD DC**

```bash
# Démasquer le service (si masqué)
sudo systemctl unmask samba-ad-dc

# Activer le démarrage automatique
sudo systemctl enable samba-ad-dc

# Démarrer le service
sudo systemctl start samba-ad-dc

# Vérifier le statut (DOIT être "active (running)")
sudo systemctl status samba-ad-dc
```

**Étape 5 : Vérifier que Samba écoute sur les bons ports**

```bash
# Vérifier les ports ouverts
sudo netstat -tulnp | grep samba

# Ports attendus :
# 53 (DNS)
# 88 (Kerberos)
# 135 (RPC)
# 139 (NetBIOS)
# 389 (LDAP)
# 445 (SMB)
# 464 (Kerberos kpasswd)
# 636 (LDAPS)
# 3268 (Global Catalog)
# 3269 (Global Catalog SSL)

# Alternative avec ss
sudo ss -tulnp | grep samba
```

**Étape 6 : Tester l'authentification Kerberos**

```bash
# Obtenir un ticket Kerberos pour l'administrateur
kinit administrator@ESNLEARN.LAB
# Entrer le mot de passe : P@ssw0rd123!

# Vérifier les tickets obtenus
klist

# Résultat attendu :
# Ticket cache: FILE:/tmp/krb5cc_1000
# Default principal: administrator@ESNLEARN.LAB
#
# Valid starting     Expires            Service principal
# 21/11/24 10:00:00  21/11/24 20:00:00  krbtgt/ESNLEARN.LAB@ESNLEARN.LAB
```

**Étape 7 : Vérifier le domaine**

```bash
# Afficher les informations du domaine
sudo samba-tool domain level show

# Résultat attendu :
# Domain and forest function level for domain 'DC=esnlearn,DC=lab'
# Forest function level: (Windows) 2008 R2
# Domain function level: (Windows) 2008 R2
# Lowest function level of a DC: (Windows) 2008 R2

# Lister les utilisateurs du domaine
sudo samba-tool user list

# Résultat attendu (au minimum) :
# Administrator
# Guest
# krbtgt
```

**🎉 Le contrôleur de domaine est maintenant opérationnel !**

---

#### C. Configuration du Serveur SAN (RAID 5)

**Sur le serveur** : `ubuntuserver` (192.168.10.193)

**Objectif** : Configurer le stockage RAID 5 pour héberger les partages Samba.
```

#### Configuration Kerberos (Détails)

**Objectif** : Vérifier et ajuster la configuration Kerberos si nécessaire.

**Note** : La configuration Kerberos a normalement été générée automatiquement lors du provisionnement.

**Étape 1 : Vérifier le fichier de configuration**

```bash
# Afficher la configuration Kerberos actuelle
cat /etc/krb5.conf
```

**Configuration type générée par Samba** :

```ini
# /etc/krb5.conf - Configuration Kerberos pour Samba AD

[libdefaults]
    default_realm = ESNLEARN.LAB          # Domaine par défaut (MAJUSCULES)
    dns_lookup_realm = false               # Ne pas chercher le realm via DNS
    dns_lookup_kdc = true                  # Chercher les KDC via DNS
    ticket_lifetime = 24h                  # Durée de vie des tickets
    renew_lifetime = 7d                    # Durée de renouvellement
    forwardable = yes                      # Tickets transférables
    rdns = false                           # Désactiver DNS inverse
    default_ccache_name = KEYRING:persistent:%{uid}  # Cache des tickets

[realms]
    ESNLEARN.LAB = {
        kdc = dc01.esnlearn.lab:88         # Serveur Kerberos (KDC)
        admin_server = dc01.esnlearn.lab   # Serveur d'administration
        default_domain = esnlearn.lab      # Domaine par défaut
    }

[domain_realm]
    .esnlearn.lab = ESNLEARN.LAB           # Mapping domaine → realm
    esnlearn.lab = ESNLEARN.LAB

[logging]
    kdc = FILE:/var/log/krb5kdc.log        # Logs KDC
    admin_server = FILE:/var/log/kadmin.log  # Logs admin
    default = FILE:/var/log/krb5lib.log    # Logs généraux
```

#### Structure des Partages selon Organigramme

```bash
# Créer les répertoires pour les partages (sur les LUNs du SAN)
sudo mkdir -p /mnt/san/lun1/{direction,comptabilite,informatique,commun}

# Définir les permissions
sudo chmod 770 /mnt/san/lun1/*
```

#### Configuration des Partages Samba

```ini
# /etc/samba/smb.conf (ajouter après la section [global])

[Direction]
    path = /mnt/san/lun1/direction
    valid users = @direction
    read only = no
    browseable = yes
    create mask = 0770
    directory mask = 0770

[Comptabilite]
    path = /mnt/san/lun1/comptabilite
    valid users = @comptabilite
    read only = no
    browseable = yes
    create mask = 0770
    directory mask = 0770

[Informatique]
    path = /mnt/san/lun1/informatique
    valid users = @informatique
    read only = no
    browseable = yes
    create mask = 0770
    directory mask = 0770

[Commun]
    path = /mnt/san/lun1/commun
    valid users = @utilisateurs
    read only = no
    browseable = yes
    create mask = 0775
    directory mask = 0775
```

#### Création des Groupes et Utilisateurs

```bash
# Créer les groupes
sudo samba-tool group add direction
sudo samba-tool group add comptabilite
sudo samba-tool group add informatique
sudo samba-tool group add utilisateurs

# Créer des utilisateurs exemples
sudo samba-tool user create jdupont P@ssw0rd123!
sudo samba-tool user create mmartin P@ssw0rd123!

# Ajouter aux groupes
sudo samba-tool group addmembers direction jdupont
sudo samba-tool group addmembers comptabilite mmartin

# Redémarrer Samba
sudo systemctl restart samba-ad-dc
```

---

### 3. Configuration du Serveur srv2 (Serveur Web)

#### Exigences
- 4 partitions de 5 Go
- Configuration RAID (RAID 5 recommandé)
- LUNs pour les partages Samba

#### Création des Partitions

```bash
# Lister les disques disponibles
lsblk
sudo fdisk -l

# Créer les partitions (exemple avec /dev/sdb)
sudo fdisk /dev/sdb
# Créer 4 partitions de 5 Go chacune
# n -> p -> 1 -> +5G
# n -> p -> 2 -> +5G
# n -> p -> 3 -> +5G
# n -> p -> 4 -> +5G
# t -> 1 -> fd (Linux RAID)
# t -> 2 -> fd
# t -> 3 -> fd
# t -> 4 -> fd
# w (write and exit)
```

#### Configuration RAID 5

```bash
# Installer mdadm
sudo apt install mdadm

# Créer le RAID 5
sudo mdadm --create --verbose /dev/md0 \
    --level=5 \
    --raid-devices=4 \
    /dev/sdb1 /dev/sdb2 /dev/sdb3 /dev/sdb4

# Vérifier la création
cat /proc/mdstat
sudo mdadm --detail /dev/md0
```

#### Formatage et Montage

```bash
# Formater en ext4
sudo mkfs.ext4 /dev/md0

# Créer le point de montage
sudo mkdir -p /mnt/san/lun1

# Monter le RAID
sudo mount /dev/md0 /mnt/san/lun1

# Montage automatique au démarrage
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
echo '/dev/md0 /mnt/san/lun1 ext4 defaults 0 0' | sudo tee -a /etc/fstab

# Mettre à jour initramfs
sudo update-initramfs -u
```

#### Création des LUNs

```bash
# Créer la structure des LUNs
sudo mkdir -p /mnt/san/lun1/{direction,comptabilite,informatique,commun}

# Définir les permissions
sudo chown -R root:root /mnt/san/lun1
sudo chmod 755 /mnt/san/lun1
sudo chmod 770 /mnt/san/lun1/{direction,comptabilite,informatique}
sudo chmod 775 /mnt/san/lun1/commun
```

#### Monitoring du RAID

```bash
# Vérifier l'état du RAID
sudo mdadm --detail /dev/md0

# Surveiller en temps réel
watch -n 1 cat /proc/mdstat

# Simuler une panne (test)
sudo mdadm --manage /dev/md0 --fail /dev/sdb2

# Remplacer un disque défaillant
sudo mdadm --manage /dev/md0 --remove /dev/sdb2
sudo mdadm --manage /dev/md0 --add /dev/sdb2
```

---

---

### 3. Configuration du Serveur srv2 (Serveur Web)

**Serveur** : `srv2` (192.168.10.194)
**Rôles** : 
- Serveur Web (Nginx)
- PHP-FPM
- MySQL/MariaDB
- HTTPS avec certificats SSL

---

#### A. Configuration du Nom d'Hôte

**Étape 1 : Définir le hostname**

```bash
# Sur le serveur srv2
sudo hostnamectl set-hostname srv2.esnlearn.lab

# Vérifier
hostname -f
# Résultat attendu : srv2.esnlearn.lab
```

**Étape 2 : Configurer /etc/hosts**

```bash
sudo nano /etc/hosts
```

**Contenu** :
```
127.0.0.1       localhost
192.168.10.194  srv2.esnlearn.lab srv2
192.168.10.193  ubuntuserver.esnlearn.lab ubuntuserver

::1             localhost ip6-localhost ip6-loopback
```

**Étape 3 : Configurer le DNS**

```bash
# Éditer resolv.conf pour utiliser ubuntuserver comme DNS
sudo nano /etc/resolv.conf
```

**Ajouter** :
```
nameserver 192.168.10.193
search esnlearn.lab
```

**Étape 4 : Tester la résolution DNS**

```bash
# Tester la résolution
ping -c 2 ubuntuserver.esnlearn.lab
nslookup srv2.esnlearn.lab 192.168.10.193
```

---

#### B. Installation LEMP Stack

```bash
# Installer Nginx, PHP et MySQL
sudo apt update
sudo apt install nginx mysql-server php-fpm php-mysql php-mbstring php-xml php-curl

# Sécuriser MySQL
sudo mysql_secure_installation
```

#### Configuration MySQL

```bash
# Créer la base de données
sudo mysql -u root -p

CREATE DATABASE esnlearn_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'esnlearn_user'@'localhost' IDENTIFIED BY 'MotDeP@sse123!';
GRANT ALL PRIVILEGES ON esnlearn_db.* TO 'esnlearn_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

#### Création de l'Autorité de Certification

```bash
# Créer le répertoire pour les certificats
sudo mkdir -p /etc/ssl/esnlearn/{ca,certs,private}
cd /etc/ssl/esnlearn

# 1. Créer la clé privée de la CA
sudo openssl genrsa -aes256 -out ca/ca.key 4096

# 2. Créer le certificat de la CA
sudo openssl req -new -x509 -days 3650 -key ca/ca.key -out ca/ca.crt \
    -subj "/C=FR/ST=Bretagne/L=Rennes/O=ESNlearn/OU=IT/CN=ESNlearn Root CA"

# 3. Créer la clé privée du serveur web (srv2)
sudo openssl genrsa -out private/srv2.key 2048

# 4. Créer une demande de signature (CSR)
sudo openssl req -new -key private/srv2.key -out certs/srv2.csr \
    -subj "/C=FR/ST=Bretagne/L=Rennes/O=ESNlearn/OU=IT/CN=srv2.esnlearn.lab"

# 5. Signer le certificat avec la CA
sudo openssl x509 -req -in certs/srv2.csr \
    -CA ca/ca.crt -CAkey ca/ca.key -CAcreateserial \
    -out certs/srv2.crt -days 365 -sha256

# Définir les permissions
sudo chmod 600 private/srv2.key
sudo chmod 644 certs/srv2.crt
```

#### Configuration Nginx avec HTTPS

```bash
# Créer le répertoire du site
sudo mkdir -p /var/www/esnlearn
sudo chown -R www-data:www-data /var/www/esnlearn

# Créer un fichier index.php de test
sudo nano /var/www/esnlearn/index.php
```

```php
<?php
phpinfo();
?>
```

```bash
# Configuration Nginx
sudo nano /etc/nginx/sites-available/esnlearn
```

```nginx
# Redirection HTTP vers HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name esnlearn.lab www.esnlearn.lab srv2.esnlearn.lab;
    
    return 301 https://$server_name$request_uri;
}

# Configuration HTTPS
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name esnlearn.lab www.esnlearn.lab srv2.esnlearn.lab;
    
    root /var/www/esnlearn;
    index index.php index.html index.htm;
    
    # Certificats SSL (srv2)
    ssl_certificate /etc/ssl/esnlearn/certs/srv2.crt;
    ssl_certificate_key /etc/ssl/esnlearn/private/srv2.key;
    
    # Configuration SSL sécurisée
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    
    # Headers de sécurité
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    
    # Logs
    access_log /var/log/nginx/esnlearn_access.log;
    error_log /var/log/nginx/esnlearn_error.log;
    
    # Configuration PHP
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.1-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
    
    # Bloquer l'accès aux fichiers cachés
    location ~ /\. {
        deny all;
    }
    
    # Configuration des fichiers statiques
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|svg|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

```bash
# Activer le site
sudo ln -s /etc/nginx/sites-available/esnlearn /etc/nginx/sites-enabled/

# Désactiver le site par défaut
sudo rm /etc/nginx/sites-enabled/default

# Tester la configuration
sudo nginx -t

# Redémarrer Nginx
sudo systemctl restart nginx
```

---

## 🔐 Sécurité et Monitoring

### Monitoring DNS

### Monitoring DNS

```bash
# Logs BIND9
sudo tail -f /var/log/syslog | grep named

# Statistiques DNS
sudo rndc stats
cat /var/cache/bind/named.stats
```

### Monitoring Samba/AD

```bash
# Logs Samba
sudo tail -f /var/log/samba/log.samba

# Réplication AD (si plusieurs DC)
samba-tool drs showrepl

# Vérifier la santé du domaine
samba-tool dbcheck
```

### Monitoring RAID

```bash
# Surveillance continue
watch -n 5 cat /proc/mdstat

# Email en cas de problème
sudo apt install mdadm mailutils
sudo dpkg-reconfigure mdadm
# Configurer l'email d'alerte
```

### Monitoring Nginx

```bash
# Logs d'accès
sudo tail -f /var/log/nginx/esnlearn_access.log

# Logs d'erreur
sudo tail -f /var/log/nginx/esnlearn_error.log

# Statistiques (installer nginx-module-vts)
# Ou utiliser GoAccess
sudo apt install goaccess
sudo goaccess /var/log/nginx/esnlearn_access.log -o /var/www/esnlearn/stats.html --log-format=COMBINED
```

---

## 🔒 Sécurité

### Firewall (UFW)

```bash
# Installation et activation
sudo apt install ufw

# Règles de base
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Autoriser SSH
sudo ufw allow 22/tcp

# Autoriser DNS
sudo ufw allow 53/tcp
sudo ufw allow 53/udp

# Autoriser HTTP/HTTPS
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Autoriser Samba
sudo ufw allow 137/udp
sudo ufw allow 138/udp
sudo ufw allow 139/tcp
sudo ufw allow 445/tcp

# Autoriser Kerberos
sudo ufw allow 88/tcp
sudo ufw allow 88/udp

# Autoriser LDAP
sudo ufw allow 389/tcp
sudo ufw allow 636/tcp

# Activer le firewall
sudo ufw enable
sudo ufw status verbose
```

### Fail2ban

```bash
# Installation
sudo apt install fail2ban

# Configuration pour SSH
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local

# [sshd]
# enabled = true
# port = 22
# logpath = /var/log/auth.log
# maxretry = 3
# bantime = 3600

# Redémarrer
sudo systemctl restart fail2ban
sudo fail2ban-client status sshd
```

### Sauvegarde

```bash
# Script de sauvegarde
sudo nano /usr/local/bin/backup-esnlearn.sh
```

```bash
#!/bin/bash
BACKUP_DIR="/backup/esnlearn"
DATE=$(date +%Y%m%d_%H%M%S)

# Créer le répertoire
mkdir -p $BACKUP_DIR

# Sauvegarder les configurations
tar -czf $BACKUP_DIR/configs_$DATE.tar.gz \
    /etc/nginx \
    /etc/samba \
    /etc/bind \
    /etc/ssl/esnlearn

# Sauvegarder la base de données
mysqldump -u root -p esnlearn_db | gzip > $BACKUP_DIR/db_$DATE.sql.gz

# Sauvegarder les partages (LUNs)
tar -czf $BACKUP_DIR/luns_$DATE.tar.gz /mnt/san/lun1

# Nettoyer les sauvegardes de plus de 30 jours
find $BACKUP_DIR -name "*.tar.gz" -mtime +30 -delete
find $BACKUP_DIR -name "*.sql.gz" -mtime +30 -delete

echo "Sauvegarde terminée : $DATE"
```

```bash
# Rendre exécutable
sudo chmod +x /usr/local/bin/backup-esnlearn.sh

# Ajouter au cron (tous les jours à 2h du matin)
sudo crontab -e
# 0 2 * * * /usr/local/bin/backup-esnlearn.sh >> /var/log/backup-esnlearn.log 2>&1
```

---

## 📝 Livrables du Projet

### Structure du Document à Rendre (PDF/Word)

1. **Page de Garde**
   - Titre du projet
   - Nom de la SSII
   - Nom du client (ESNlearn)
   - Date
   - Votre nom

2. **Table des Matières**

3. **Introduction**
   - Contexte du projet
   - Problématiques identifiées
   - Objectifs

4. **Architecture Réseau**
   - Schéma d'infrastructure complet
   - Tableau d'adressage IP
   - Nomenclature des serveurs
   - Justification des choix techniques

5. **Serveur DNS/DDNS**
   - Configuration BIND9
   - Zone DNS
   - Redirecteur
   - Captures d'écran
   - Tests de validation

6. **Serveur DC + Fichiers**
   - Installation Samba 4 AD DC
   - Structure organisationnelle
   - Configuration des partages
   - Intégration avec SAN
   - Captures d'écran
   - Tests d'authentification

7. **Serveur SAN**
   - Configuration RAID 5
   - Création des partitions
   - Création des LUNs
   - Monitoring RAID
   - Captures d'écran

8. **Serveur Web**
   - Stack LEMP
   - Autorité de certification
   - Configuration HTTPS
   - Sécurisation
   - Captures d'écran
   - Tests de connexion

9. **Sécurité**
   - Firewall (UFW)
   - Fail2ban
   - Certificats SSL
   - Politique de mots de passe

10. **Monitoring et Maintenance**
    - Outils de surveillance
    - Logs
    - Sauvegardes
    - Procédures de maintenance

11. **Conclusion**
    - Résultats obtenus
    - Difficultés rencontrées
    - Solutions apportées
    - Améliorations possibles
    - Recommandations

13. **Annexes**
    - Fichiers de configuration complets
    - Scripts utilisés
    - Commandes détaillées
    - Références documentaires

---

## ✅ Check-list de Validation

### Infrastructure Réseau
- [ ] Schéma réseau clair et détaillé
- [ ] Tableau d'adressage IP complet
- [ ] Nomenclature des serveurs cohérente
- [ ] Connectivité entre tous les serveurs validée

### Serveur DNS/DDNS
- [ ] BIND9 installé et configuré
- [ ] Zone DNS fonctionnelle
- [ ] Redirecteur configuré vers DNS externes
- [ ] Résolution DNS testée (nslookup, dig)
- [ ] DDNS fonctionnel

### Serveur DC + Fichiers
- [ ] Samba 4 AD DC installé
- [ ] Domaine provisionné et fonctionnel
- [ ] Utilisateurs et groupes créés
- [ ] Partages configurés selon organigramme
- [ ] Partages pointent vers LUNs du SAN
- [ ] Authentification testée
- [ ] Accès aux partages validé

### Serveur SAN
- [ ] 4 partitions de 5 Go créées
- [ ] RAID 5 configuré et fonctionnel
- [ ] RAID monté automatiquement au démarrage
- [ ] LUNs créés et accessibles
- [ ] Performance RAID testée
- [ ] Monitoring RAID en place

### Serveur Web
- [ ] Stack LEMP installée
- [ ] Autorité de certification créée
- [ ] Certificat SSL généré et signé
- [ ] Nginx configuré avec HTTPS
- [ ] Redirection HTTP → HTTPS fonctionnelle
- [ ] PHP fonctionnel
- [ ] MySQL/Base de données configurée
- [ ] Site accessible en HTTPS

### Sécurité
- [ ] Firewall configuré (UFW)
- [ ] Fail2ban installé et actif
- [ ] Certificats SSL valides
- [ ] Headers de sécurité configurés
- [ ] Politique de mots de passe forte

### Documentation
- [ ] Toutes les configurations commentées
- [ ] Captures d'écran de chaque étape
- [ ] Tests documentés avec résultats
- [ ] Schémas clairs et légendés
- [ ] Document professionnel (PDF/Word)

---

## 🚀 Recommandations Finales

### Points d'Attention

1. **Sauvegarde Régulière**
   - Mettre en place des sauvegardes automatiques quotidiennes
   - Tester régulièrement la restauration

2. **Monitoring**
   - Installer des outils de surveillance (Nagios, Zabbix, Prometheus)
   - Configurer des alertes email/SMS

3. **Mise à Jour**
   - Planifier des fenêtres de maintenance
   - Appliquer les mises à jour de sécurité

4. **Documentation**
   - Maintenir une documentation à jour
   - Former les administrateurs

5. **Plan de Reprise d'Activité**
   - Documenter les procédures de récupération
   - Tester régulièrement les scénarios de panne

### Évolutions Possibles

- **Haute Disponibilité** : Cluster de serveurs DC/DNS
- **Load Balancing** : HAProxy pour le serveur web
- **Conteneurisation** : Migration vers Docker/Kubernetes
- **Backup Distant** : Réplication vers site distant
- **VPN** : Accès distant sécurisé pour les utilisateurs

---

## 📚 Ressources Complémentaires

### Documentation Officielle
- [BIND9 Documentation](https://bind9.readthedocs.io/)
- [Samba Wiki](https://wiki.samba.org/)
- [Nginx Documentation](https://nginx.org/en/docs/)
- [Linux RAID Documentation](https://raid.wiki.kernel.org/)

### Guides Utiles
- Ubuntu Server Guide
- Debian Administrator's Handbook
- Linux System Administrator's Guide

---

**Projet réalisé dans le cadre de la mission chez ESNlearn**

*Consultant : [Votre Nom]*  
*SSII : [xxx]*  
*Date : Novembre 2025*
