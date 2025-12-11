# Projet : Déploiement d'une Architecture Réseau Distribuée sur Rocky Linux 9

## 🌐 1. Architecture et Prérequis

Ce projet déploie une infrastructure réseau cloisonnée simulant un environnement d'entreprise. Contrairement à une configuration "tout-en-un", ce projet sépare le rôle de résolution de noms (DNS) des services applicatifs (Web/Fichiers) pour plus de sécurité et de performance.

### 1.1. Inventaire des Machines (VMs)

| Machine | Nom d'Hôte | Rôle | IP Statique (Host-Only) |
| :--- | :--- | :--- | :--- |
| **VM 1** | **SRV-DNS** | Serveur DNS Dédié (BIND) | **192.168.142.10** |
| **VM 2** | **SRV-APPS** | Serveur Web, Fichiers & Sauvegarde | **192.168.142.11** |
| **VM 3** | **CLI-TEST** | Client de Test | **192.168.142.20** |

### 1.2. Configuration de la Plateforme (VirtualBox)

  * **Adaptateur 1 :** NAT (Accès Internet pour l'installation des paquets).
  * **Adaptateur 2 :** Réseau Hôte-Seul (Host-Only Adapter) - Réseau `192.168.142.x`.

### 1.3. Configuration IP Initiale (Commandes `nmcli`)

Exécutez ces commandes sur chaque machine respective pour fixer l'IP sur l'interface Host-Only (généralement `enp0s8`).

**Sur VM 1 (SRV-DNS) :**

```bash
sudo nmcli con add type ethernet ifname enp0s8 con-name static-hostonly ip4 192.168.142.10/24 gw4 192.168.142.1 autoconnect yes
sudo nmcli con up static-hostonly
sudo hostnamectl set-hostname SRV-DNS
```

**Sur VM 2 (SRV-APPS) :**

```bash
sudo nmcli con add type ethernet ifname enp0s8 con-name static-hostonly ip4 192.168.142.11/24 gw4 192.168.142.1 autoconnect yes
sudo nmcli con up static-hostonly
sudo hostnamectl set-hostname SRV-APPS
```

**Sur VM 3 (CLI-TEST) :**

```bash
sudo nmcli con add type ethernet ifname enp0s8 con-name static-hostonly ip4 192.168.142.20/24 gw4 192.168.142.1 autoconnect yes
sudo nmcli con up static-hostonly
sudo hostnamectl set-hostname CLI-TEST
```

-----

## 🛡️ 2. Configuration de la VM 1 : SRV-DNS (Service de Noms)

Ce serveur ne gère **que** la résolution de noms. Il dirige le trafic.

### 2.1. Installation et Pare-feu

```bash
# Installation de BIND
sudo dnf install bind bind-utils -y

# Ouverture des ports DNS (53 TCP/UDP)
sudo systemctl enable --now firewalld
sudo firewall-cmd --permanent --add-port={53/tcp,53/udp}
sudo firewall-cmd --reload
```

### 2.2. Configuration BIND (`/etc/named.conf`)

Editez le fichier de configuration principal pour écouter sur le réseau.

```bash
sudo vi /etc/named.conf
```

  * **listen-on port 53 :** Ajouter `192.168.142.10;`
  * **allow-query :** Ajouter `192.168.142.0/24;`
  * **Ajouter la zone à la fin du fichier :**
    ```text
    zone "monlabo.lan" IN {
        type master;
        file "db.monlabo.lan";
        allow-update { none; };
    };
    ```

### 2.3. Création du Fichier de Zone (`/var/named/db.monlabo.lan`)

C'est ici que nous définissons que les services se trouvent sur l'IP `.11`.

```bash
sudo cp /var/named/named.localhost /var/named/db.monlabo.lan
sudo vi /var/named/db.monlabo.lan
```

**Contenu du fichier :**

```text
$TTL 86400
@   IN  SOA     srv-dns.monlabo.lan. root.monlabo.lan. (
    2025121101  ; Serial
    3600        ; Refresh
    1800        ; Retry
    604800      ; Expire
    86400 )     ; Minimum TTL

    IN  NS      srv-dns.monlabo.lan.
    IN  A       192.168.142.10

srv-dns     IN  A   192.168.142.10
; Les services applicatifs pointent vers la VM 2
web         IN  A   192.168.142.11
fichiers    IN  A   192.168.142.11
client      IN  A   192.168.142.20
```

### 2.4. Activation

```bash
sudo named-checkzone monlabo.lan /var/named/db.monlabo.lan # Vérification
sudo systemctl enable --now named
```

-----

## ⚙️ 3. Configuration de la VM 2 : SRV-APPS (Web, Samba, Sauvegardes)

Ce serveur héberge les applications. Il possède l'IP **192.168.142.11**.

### 3.1. Installation des Paquets et Pare-feu

```bash
# Installation Nginx, Samba, Rsync, Cron
sudo dnf install nginx samba samba-client rsync cronie -y

# Ouverture des ports Web (80/443) et Samba (445)
sudo systemctl enable --now firewalld
sudo firewall-cmd --permanent --add-service={http,https}
sudo firewall-cmd --permanent --add-port=445/tcp
sudo firewall-cmd --reload
```

### 3.2. Service Web Sécurisé (Nginx/HTTPS)

**A. Génération du Certificat SSL Auto-signé**

```bash
sudo mkdir -p /etc/nginx/ssl
# Répondre 'web.monlabo.lan' pour le Common Name (CN)
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/nginx/ssl/labo.key -out /etc/nginx/ssl/labo.crt
```

**B. Configuration Nginx**
Assurez-vous que `/etc/nginx/nginx.conf` inclut un bloc serveur écoutant sur 443 avec ssl activé et pointant vers les certificats créés.

**C. Correctifs SELinux (Pour le Web)**

```bash
sudo restorecon -Rv /etc/nginx/
sudo restorecon -Rv /usr/share/nginx/html/
sudo systemctl enable --now nginx
```

### 3.3. Service de Fichiers (Samba)

**A. Création Dossier et Utilisateur**

```bash
sudo mkdir -p /srv/samba/projet
sudo useradd adrien # Si l'utilisateur système n'existe pas
sudo smbpasswd -a adrien # Définir le mot de passe Samba
```

**B. Configuration (`/etc/samba/smb.conf`)**
Ajoutez à la fin :

```ini
[ProjetSecret]
    path = /srv/samba/projet
    valid users = adrien
    read only = no
    browsable = yes
```

**C. Correctifs SELinux (Crucial pour Samba)**

```bash
sudo chcon -R -t samba_share_t /srv/samba/projet
sudo setsebool -P samba_enable_home_dirs on
sudo setsebool -P samba_export_all_rw on
sudo systemctl enable --now smb nmb
```

### 3.4. Automatisation des Sauvegardes (Rsync + Cron)

Mise en place d'une sauvegarde quotidienne du dossier utilisateur vers `/srv/sauvegardes`.

**A. Création de la tâche Cron**

```bash
# Éditer le crontab root
sudo crontab -e
```

**B. Ajouter la ligne suivante (Sauvegarde à 03h00 du matin) :**

```cron
0 3 * * * /usr/bin/rsync -az --delete /home/adrien/Documents /srv/sauvegardes/ > /dev/null 2>&1
```

-----

## 🧪 4. Configuration de la VM 3 : CLI-TEST (Client)

Le client doit être configuré pour utiliser notre DNS privé en priorité, et ignorer le DNS public fourni par l'interface NAT.

### 4.1. Configuration Réseau (La Fixe NetworkManager)

Pour éviter que le NAT n'écrase le DNS local :

```bash
# 1. Identifier la connexion NAT (ex: "enp0s3" ou "System enp0s3")
nmcli dev status
# 2. Désactiver le DNS sur l'interface NAT (Remplacer "enp0s3" par le nom réel)
sudo nmcli con mod "enp0s3" ipv4.dns-method "none"
sudo nmcli con mod "enp0s3" ipv4.ignore-auto-dns yes

# 3. Forcer le DNS local sur l'interface Host-Only
sudo nmcli con mod static-hostonly ipv4.dns "192.168.142.10"
sudo nmcli con mod static-hostonly ipv4.dns-search "monlabo.lan"

# 4. Appliquer
sudo nmcli con up "enp0s3"
sudo nmcli con up static-hostonly
```

### 4.2. Validation Finale

Une fois configuré, voici les commandes pour valider le projet :

| Test | Commande | Résultat Attendu |
| :--- | :--- | :--- |
| **Ping DNS** | `ping -c 3 192.168.142.10` | 0% packet loss |
| **Résolution** | `dig web.monlabo.lan` | Réponse : **192.168.142.11** |
| **Web HTTPS** | `curl -k https://web.monlabo.lan` | Code HTML affiché |
| **Samba** | `smbclient //web.monlabo.lan/ProjetSecret -U adrien` | Invite de commande `smb: \>` |