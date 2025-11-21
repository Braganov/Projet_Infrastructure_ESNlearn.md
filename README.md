# 📋 Infrastructure ESNlearn - Guide Simplifié

## 🌐 Infrastructure Réseau et Nomenclature

### Tableau d'Adressage IP

| Serveur | Rôle | IP | Sous-réseau | Gateway |
|---------|------|-------|-------------|---------|
| **ubuntuserver** | DC + DNS + SAN (Multi-rôle) | 192.168.10.193 | 255.255.255.128 | 192.168.10.129 |
| **srv2** | Serveur Web | 192.168.10.194 | 255.255.255.128 | 192.168.10.129 |

---

## 📊 Schéma d'Infrastructure

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
  │ │   4x5Go      │ │        │ │   Database   │ │
  │ │   /mnt/san   │ │        │ └──────────────┘ │
  │ └──────────────┘ │        └──────────────────┘
  │ ┌──────────────┐ │               
  │ │  Partages    │ │        (Accède aux partages  
  │ │   Samba      │ │         Samba via réseau)
  │ └──────────────┘ │               
  └──────────────────┘

  Serveur Multi-Rôles          Serveur Web Dédié
```

### ✅ Avantages de cette architecture

- **Réduction des coûts** (2 serveurs au lieu de 4)
- **Simplification de la gestion**
- **Centralisation des services d'infrastructure**
- **Séparation du serveur web pour la sécurité**
- **Facilité de sauvegarde**

---

## 🖥️ Configuration des Serveurs

### 1. Configuration du Serveur ubuntuserver (DC + DNS + SAN)

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

**Configuration avec Redirecteur :**

```bind
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

---

#### Configuration Zone DDNS

**Objectif** : Configurer les zones DNS pour permettre les mises à jour dynamiques (DDNS).

**Étape 1 : Éditer le fichier de configuration des zones locales**

```bash
# Ouvrir le fichier de configuration
sudo nano /etc/bind/named.conf.local
```

**Étape 2 : Ajouter la configuration des zones**

```bind
# /etc/bind/named.conf.local

# Zone de résolution directe (Forward Zone)
zone "esnlearn.lab" {
    type master;                              # Type de zone : master (primaire)
    file "/var/lib/bind/db.esnlearn.lab";     # Chemin du fichier de zone
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
# Vérifier la syntaxe de la configuration
sudo named-checkconf /etc/bind/named.conf.local
# Si aucune erreur n'est affichée, la configuration est correcte
```

📝 **Explication des paramètres :**
- `type master` : Ce serveur est autoritaire pour cette zone
- `file` : Emplacement du fichier de zone (dans /var/lib/bind)
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

**Contenu du fichier de zone :**

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
srv2           IN      A       192.168.10.194

; Alias (CNAME)
www            IN      CNAME   srv2
```

**Étape 3 : Définir les permissions**

```bash
# Définir le propriétaire et les permissions
sudo chown bind:bind /var/lib/bind/db.esnlearn.lab
sudo chmod 644 /var/lib/bind/db.esnlearn.lab
```

**Étape 4 : Créer le fichier de zone inverse (Reverse Zone)**

```bash
# Créer le fichier de zone inverse
sudo nano /var/lib/bind/rev.esnlearn.lab
```

**Contenu du fichier de zone inverse :**

```bind
$TTL    604800
@       IN      SOA     ubuntuserver.esnlearn.lab. admin.esnlearn.lab. (
                              2024112101 ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL

; Serveur de noms
@       IN      NS      ubuntuserver.esnlearn.lab.

; Enregistrements PTR (Pointer) pour la résolution inverse
193     IN      PTR     ubuntuserver.esnlearn.lab.  ; 192.168.10.193
194     IN      PTR     srv2.esnlearn.lab.          ; 192.168.10.194
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

📸 **Screenshot** : État du service BIND9

---

#### B. Configuration du Serveur SAN (RAID 5)

**Configuration RAID 5 avec 4 partitions de 5 Go**

**1. Lister les disques et créer les partitions**

```bash
lsblk
sudo fdisk /dev/sdb
```

📸 **Screenshot** : Résultat de lsblk montrant les 4 partitions

**2. Installer mdadm et créer le RAID 5**

```bash
sudo apt install -y mdadm
sudo mdadm --create --verbose /dev/md0 --level=5 --raid-devices=4 /dev/sdb1 /dev/sdb2 /dev/sdb3 /dev/sdb4
cat /proc/mdstat
```

📸 **Screenshot** : État du RAID (cat /proc/mdstat)

**3. Formater et monter le RAID**

```bash
sudo mkfs.ext4 /dev/md0
sudo mkdir -p /mnt/san/lun1
sudo mount /dev/md0 /mnt/san/lun1
df -h | grep md0
```

📸 **Screenshot** : Montage du RAID (df -h)

**4. Configuration du montage automatique**

```bash
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
sudo nano /etc/fstab
```

**Ajouter :**
```
/dev/md0 /mnt/san/lun1 ext4 defaults,nofail 0 2
```

```bash
sudo update-initramfs -u
```

**5. Créer la structure des partages**

```bash
sudo mkdir -p /mnt/san/lun1/{direction,comptabilite,informatique,commun}
ls -la /mnt/san/lun1/
```

📸 **Screenshot** : Structure des répertoires

**6. Configuration du monitoring RAID**

```bash
# Configurer les alertes email (optionnel)
sudo dpkg-reconfigure mdadm

# Activer le monitoring
sudo systemctl enable mdmonitor
sudo systemctl start mdmonitor

# Vérifier le service
sudo systemctl status mdmonitor
```

**Monitoring et Maintenance du RAID**

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

---

#### C. Configuration Samba - Partages de Fichiers

**Installation et configuration de base**

```bash
sudo apt update
sudo apt install samba

# Créer le répertoire de partage
sudo mkdir -p /srv/samba/share
sudo chmod 777 /srv/samba/share 
```

**Créer un utilisateur Samba**

```bash
sudo adduser toto

# Définir le mot de passe Samba
sudo smbpasswd -a toto
sudo smbpasswd -e toto
```

**Configuration du fichier smb.conf**

```bash
sudo nano /etc/samba/smb.conf
```

**Contenu de base :**

```ini
[global]
   workgroup = WORKGROUP
   server string = Serveur Samba ESNlearn
   netbios name = ubuntuserver
   security = user
   map to guest = bad user

[share]
   comment = Partage de fichiers
   path = /srv/samba/share
   browseable = yes
   read only = no
   guest ok = no
   valid users = toto
```

**Vérifier la configuration et démarrer les services**

```bash
testparm

sudo systemctl enable smbd --now
sudo systemctl enable nmbd --now
```

📸 **Screenshot** : Statut des services Samba

---

#### D. Configuration Samba 4 AD DC (Contrôleur de Domaine)

**Sauvegarder la configuration existante**

```bash
sudo mv /etc/samba/smb.conf /etc/samba/smb.conf.bak
```

**Installation des paquets nécessaires**

```bash
sudo apt install -y samba smbclient winbind krb5-user libpam-winbind libnss-winbind
```

**Provisionner le domaine Active Directory**

```bash
sudo samba-tool domain provision --use-rfc2307 --interactive
```

**Paramètres suggérés lors du provisionnement :**
- **Realm** : `ESNLEARN.LAB`
- **Domain** : `ESNLEARN`
- **Server Role** : `dc` (Domain Controller)
- **DNS backend** : `SAMBA_INTERNAL`
- **DNS forwarder** : `8.8.8.8`
- **Administrator password** : `[Mot de passe fort]`

**Configuration du fichier smb.conf (généré automatiquement)**

```bash
sudo nano /etc/samba/smb.conf
```

**Exemple de configuration AD DC :**

```ini
[global]
    dns forwarder = 8.8.8.8
    netbios name = UBUNTUSERVER
    realm = ESNLEARN.LAB
    server role = active directory domain controller
    workgroup = ESNLEARN
    idmap_ldb:use rfc2307 = yes

[netlogon]
    path = /var/lib/samba/sysvol/esnlearn.lab/scripts
    read only = No

[sysvol]
    path = /var/lib/samba/sysvol
    read only = No
```

**Démarrer et activer le service**

```bash
sudo systemctl enable samba-ad-dc
sudo systemctl start samba-ad-dc
sudo systemctl status samba-ad-dc
```

📸 **Screenshot** : Statut du contrôleur de domaine

**Vérifier le domaine**

```bash
# Tester la configuration Kerberos
kinit administrator@ESNLEARN.LAB

# Lister les utilisateurs du domaine
samba-tool user list

# Vérifier le DNS
host -t A ubuntuserver.esnlearn.lab
host -t SRV _ldap._tcp.esnlearn.lab
```

---

### 2. Configuration du Serveur srv2 (Serveur Web)

**Serveur** : `srv2` (192.168.10.194)  
**Rôles** :
- Serveur Web (Nginx)
- PHP-FPM
- MySQL/MariaDB
- HTTPS avec certificats SSL

---

#### A. Installation LEMP Stack

```bash
# Mise à jour du système
sudo apt update && sudo apt upgrade -y

# Installation Nginx, PHP et MySQL
sudo apt install -y nginx mysql-server php-fpm php-mysql php-mbstring php-xml php-curl

# Sécuriser MySQL
sudo mysql_secure_installation
```

📸 **Screenshot** : Services installés

---

#### B. Configuration MySQL

```bash
# Se connecter à MySQL
sudo mysql -u root -p
```

**Créer la base de données :**

```sql
CREATE DATABASE esnlearn_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'esnlearn_user'@'localhost' IDENTIFIED BY 'MotDeP@sse123!';
GRANT ALL PRIVILEGES ON esnlearn_db.* TO 'esnlearn_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

---

#### C. Configuration Nginx avec PHP

**Créer le répertoire du site**

```bash
sudo mkdir -p /var/www/esnlearn
sudo chown -R www-data:www-data /var/www/esnlearn
```

**Créer un fichier index.php de test**

```bash
sudo nano /var/www/esnlearn/index.php
```

```php
<?php
phpinfo();
?>
```

**Configuration Nginx**

```bash
sudo nano /etc/nginx/sites-available/esnlearn
```

**Contenu de base :**

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name esnlearn.lab www.esnlearn.lab srv2.esnlearn.lab;
    
    root /var/www/esnlearn;
    index index.php index.html index.htm;
    
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.1-fpm.sock;
    }
    
    location ~ /\. {
        deny all;
    }
}
```

**Activer le site**

```bash
sudo ln -s /etc/nginx/sites-available/esnlearn /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl restart nginx
```

📸 **Screenshot** : Page phpinfo() accessible

---

#### D. Configuration HTTPS avec SSL

**Créer une autorité de certification (CA)**

```bash
# Créer le répertoire pour les certificats
sudo mkdir -p /etc/ssl/esnlearn/{ca,certs,private}
cd /etc/ssl/esnlearn

# Créer la clé privée de la CA
sudo openssl genrsa -aes256 -out ca/ca.key 4096

# Créer le certificat de la CA
sudo openssl req -new -x509 -days 3650 -key ca/ca.key -out ca/ca.crt \
    -subj "/C=FR/ST=Bretagne/L=Rennes/O=ESNlearn/OU=IT/CN=ESNlearn Root CA"

# Créer la clé privée du serveur
sudo openssl genrsa -out private/srv2.key 2048

# Créer une demande de signature (CSR)
sudo openssl req -new -key private/srv2.key -out certs/srv2.csr \
    -subj "/C=FR/ST=Bretagne/L=Rennes/O=ESNlearn/OU=IT/CN=srv2.esnlearn.lab"

# Signer le certificat avec la CA
sudo openssl x509 -req -in certs/srv2.csr \
    -CA ca/ca.crt -CAkey ca/ca.key -CAcreateserial \
    -out certs/srv2.crt -days 365 -sha256

# Définir les permissions
sudo chmod 600 private/srv2.key
sudo chmod 644 certs/srv2.crt
```

**Configuration Nginx avec HTTPS**

```bash
sudo nano /etc/nginx/sites-available/esnlearn
```

**Configuration complète avec HTTPS :**

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
    
    # Certificats SSL
    ssl_certificate /etc/ssl/esnlearn/certs/srv2.crt;
    ssl_certificate_key /etc/ssl/esnlearn/private/srv2.key;
    
    # Configuration SSL sécurisée
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    
    # Headers de sécurité
    add_header Strict-Transport-Security "max-age=31536000" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    
    # Configuration PHP
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.1-fpm.sock;
    }
    
    location ~ /\. {
        deny all;
    }
}
```

**Redémarrer Nginx**

```bash
sudo nginx -t
sudo systemctl restart nginx
```

📸 **Screenshot** : Site accessible en HTTPS

---

## 📝 Notes Importantes

### Configuration Réseau
- **Sous-réseau** : 192.168.10.0/25 (255.255.255.128)
- **Plage d'adresses** : 192.168.10.0 - 192.168.10.127
- **Gateway** : 192.168.10.129

### Domaine
- **DNS** : esnlearn.lab
- **AD Domain** : ESNLEARN.LAB
- **Kerberos Realm** : ESNLEARN.LAB

### RAID 5
- **Capacité totale** : 15 Go (4x5Go en RAID 5)
- **Espace utilisable** : ~15 Go
- **Point de montage** : /mnt/san/lun1

### Sécurité
- Utiliser des mots de passe forts pour tous les services
- Configurer le pare-feu (ufw) pour limiter l'accès
- Mettre à jour régulièrement le système
- Sauvegarder régulièrement les données

---

## ✅ Checklist de Déploiement

### ubuntuserver (192.168.10.193)
- [ ] DNS (BIND9) configuré et fonctionnel
- [ ] RAID 5 créé et monté
- [ ] Samba Shares configurés
- [ ] Samba AD DC provisionné et actif
- [ ] Monitoring RAID activé

### srv2 (192.168.10.194)
- [ ] Nginx installé et configuré
- [ ] PHP-FPM fonctionnel
- [ ] MySQL configuré avec base de données
- [ ] Certificats SSL créés
- [ ] HTTPS activé et sécurisé
- [ ] Résolution DNS fonctionnelle

---

**Date de création** : 21 novembre 2025  
**Version** : 1.0
