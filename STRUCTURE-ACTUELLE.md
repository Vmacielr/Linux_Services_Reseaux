# Projet : Architecture Réseau Distribuée et Sécurisée avec Sauvegarde Déportée

## 🌐 1. Architecture et Topologie

Ce projet déploie une infrastructure réseau cloisonnée simulant un environnement d'entreprise robuste. L'architecture respecte les bonnes pratiques de sécurité : séparation des rôles (DNS vs Apps), isolation des sauvegardes, et chiffrement des flux.

### 1.1. Inventaire des Machines (VMs)

Le réseau Host-Only est : **192.168.142.0/24**.

| Machine | Nom d'Hôte | Rôle Principal | IP Statique |
| :--- | :--- | :--- | :--- |
| **VM 1** | **`SRV-DNS`** | Serveur de Noms & Passerelle DNS | **192.168.142.10** |
| **VM 2** | **`SRV-APPS`** | Web (HTTPS) & Fichiers (Samba) | **192.168.142.11** |
| **VM 3** | **`SRV-BACKUP`** | Stockage de Sauvegarde Isolé | **192.168.142.12** |
| **VM 4** | **`CLI-TEST`** | Client Utilisateur | **192.168.142.20** |

### 1.2. Prérequis VirtualBox

  * **Adaptateur 1 :** NAT (Accès Internet).
  * **Adaptateur 2 :** Réseau Hôte-Seul (Host-Only Adapter).

-----

## 🛠️ 2. Configuration IP Initiale (Sur toutes les VMs)

Pour éviter les conflits d'IP et l'écrasement par le DHCP, nous forçons la configuration statique via `nmcli`.

**Sur VM 1 (SRV-DNS) :**

```bash
sudo hostnamectl set-hostname SRV-DNS
sudo nmcli con add type ethernet ifname enp0s8 con-name static-hostonly ip4 192.168.142.10/24 autoconnect yes
sudo nmcli con up static-hostonly
```

**Sur VM 2 (SRV-APPS) :**

```bash
sudo hostnamectl set-hostname SRV-APPS
sudo nmcli con add type ethernet ifname enp0s8 con-name static-hostonly ip4 192.168.142.11/24 autoconnect yes
sudo nmcli con up static-hostonly
```

**Sur VM 3 (SRV-BACKUP) :**

```bash
sudo hostnamectl set-hostname SRV-BACKUP
sudo nmcli con add type ethernet ifname enp0s8 con-name static-hostonly ip4 192.168.142.12/24 autoconnect yes
sudo nmcli con up static-hostonly
```

**Sur VM 4 (CLI-TEST) :**

```bash
sudo hostnamectl set-hostname CLI-TEST
sudo nmcli con add type ethernet ifname enp0s8 con-name static-hostonly ip4 192.168.142.20/24 autoconnect yes
sudo nmcli con up static-hostonly
```

-----

## 🧠 3. VM 1 : SRV-DNS (Le Cerveau du Réseau)

Il résout les noms locaux (`.lan`) et transmet les requêtes inconnues vers Internet (*Forwarding*), permettant aux autres serveurs (comme Backup) de faire leurs mises à jour.

### 3.1. Installation et Pare-feu

```bash
sudo dnf install bind bind-utils -y
sudo systemctl enable --now firewalld
sudo firewall-cmd --permanent --add-port={53/tcp,53/udp}
sudo firewall-cmd --reload
```

### 3.2. Configuration BIND (`/etc/named.conf`)

Modifiez les options pour écouter sur le réseau et activer le forwarding.

```bash
sudo vi /etc/named.conf
```

**Modifications clés :**

```conf
options {
    listen-on port 53 { 127.0.0.1; 192.168.142.10; }; # Ajouter l'IP locale
    allow-query     { localhost; 192.168.142.0/24; }; # Autoriser le réseau
    
    # Activer le forwarding vers Google (pour que les VMs aient internet via le DNS)
    forward only;
    forwarders { 8.8.8.8; };
};

# Ajouter la zone à la fin :
zone "monlabo.lan" IN {
    type master;
    file "db.monlabo.lan";
    allow-update { none; };
};
```

### 3.3. Fichier de Zone (`/var/named/db.monlabo.lan`)

```bash
sudo cp /var/named/named.localhost /var/named/db.monlabo.lan
sudo vi /var/named/db.monlabo.lan
```

**Contenu :**

```text
$TTL 86400
@   IN  SOA     srv-dns.monlabo.lan. root.monlabo.lan. (
    2025121102  ; Serial
    3600        ; Refresh
    1800        ; Retry
    604800      ; Expire
    86400 )     ; Minimum TTL

    IN  NS      srv-dns.monlabo.lan.
    IN  A       192.168.142.10

srv-dns     IN  A   192.168.142.10
srv-apps    IN  A   192.168.142.11
srv-backup  IN  A   192.168.142.12

; Services
web         IN  A   192.168.142.11
fichiers    IN  A   192.168.142.11
```

**Finalisation :**

```bash
sudo systemctl enable --now named
sudo systemctl restart named
```

-----

## ⚙️ 4. VM 2 : SRV-APPS (Services Web & Fichiers)

### 4.1. Installation et Sécurité

```bash
sudo dnf install nginx samba samba-client rsync cronie -y
sudo systemctl enable --now firewalld
sudo firewall-cmd --permanent --add-service={http,https}
sudo firewall-cmd --permanent --add-port=445/tcp
sudo firewall-cmd --reload
```

### 4.2. HTTPS (Nginx)

```bash
# Certificat
sudo mkdir -p /etc/nginx/ssl
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/nginx/ssl/labo.key -out /etc/nginx/ssl/labo.crt
# (Configurer nginx.conf pour utiliser ces fichiers sur le port 443)

# SELinux Web
sudo restorecon -Rv /etc/nginx/
sudo systemctl enable --now nginx
```

### 4.3. Partage Samba

```bash
# Utilisateur
sudo useradd adrien
sudo smbpasswd -a adrien

# Dossier
sudo mkdir -p /srv/samba/projet
sudo chown adrien:adrien /srv/samba/projet
sudo chmod 770 /srv/samba/projet

# Config (/etc/samba/smb.conf)
# Ajouter :
# [ProjetSecret]
#    path = /srv/samba/projet
#    valid users = adrien
#    read only = no

# SELinux Samba (Critique)
sudo chcon -R -t samba_share_t /srv/samba/projet
sudo setsebool -P samba_enable_home_dirs on
sudo setsebool -P samba_export_all_rw on
sudo systemctl enable --now smb nmb
```

-----

## 💾 5. VM 3 : SRV-BACKUP (Le Coffre-fort)

Ce serveur est isolé. Il n'ouvre que le port SSH.

### 5.1. Installation

*Note : Si le DNS n'est pas encore prêt, utiliser `sudo nmcli con mod enp0s3 ipv4.dns 8.8.8.8` temporairement pour l'install.*

```bash
sudo dnf install rsync openssh-server -y
sudo systemctl enable --now sshd
```

### 5.2. Sécurité Maximale (Pare-feu)

On ferme tout, sauf SSH.

```bash
sudo firewall-cmd --permanent --remove-service={dhcpv6-client,cockpit,http,https}
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --reload
```

### 5.3. Dossier de Réception

```bash
sudo mkdir -p /mnt/sauvegardes/srv-apps
# Créer l'utilisateur adrien s'il n'existe pas
sudo useradd adrien
sudo chown -R adrien:adrien /mnt/sauvegardes
sudo chmod -R 770 /mnt/sauvegardes
```

-----

## 🔄 6. Automatisation de la Sauvegarde (Rsync over SSH)

La sauvegarde est initiée par **SRV-APPS** vers **SRV-BACKUP**.

### 6.1. Échange de Clés SSH (Sur SRV-APPS)

Nous configurons l'utilisateur `root` pour qu'il puisse se connecter sans mot de passe.

```bash
# Sur SRV-APPS
sudo -i  # Passer en root
ssh-keygen -t rsa # (Entrée, Entrée, Entrée)
ssh-copy-id adrien@192.168.142.12
exit # Quitter root
```

### 6.2. Tâche Cron (Sur SRV-APPS)

```bash
sudo crontab -e
```

Ajouter la ligne (tous les jours à 03h00) :

```cron
0 3 * * * /usr/bin/rsync -az --delete /srv/samba/projet/ adrien@192.168.142.12:/mnt/sauvegardes/srv-apps/ > /dev/null 2>&1
```

-----

## 🧪 7. VM 4 : CLI-TEST (Configuration Client & Tests)

### 7.1. Forcer le DNS Local (NetworkManager)

Il faut empêcher l'interface NAT d'écraser le DNS.

```bash
# 1. Identifier la connexion NAT (ex: enp0s3)
sudo nmcli con mod "enp0s3" ipv4.dns-method "none"
sudo nmcli con mod "enp0s3" ipv4.ignore-auto-dns yes

# 2. Configurer le Host-Only pour utiliser SRV-DNS
sudo nmcli con mod static-hostonly ipv4.dns "192.168.142.10"
sudo nmcli con mod static-hostonly ipv4.dns-search "monlabo.lan"

# 3. Appliquer
sudo nmcli con up "enp0s3"
sudo nmcli con up static-hostonly
```

### 7.2. Tableau de Validation

| Test | Commande | Résultat Attendu |
| :--- | :--- | :--- |
| **Ping DNS** | `ping -c 3 192.168.142.10` | 0% packet loss |
| **Résolution** | `dig web.monlabo.lan` | Réponse : **192.168.142.11** |
| **Accès Web** | `curl -k https://web.monlabo.lan` | Code HTML affiché |
| **Accès Fichiers** | `smbclient //web.monlabo.lan/ProjetSecret -U adrien` | Connexion réussie (`smb: \>`) |
| **Vérif. Backup** | `ls -l /mnt/sauvegardes/srv-apps` (Sur SRV-BACKUP) | Fichiers présents |