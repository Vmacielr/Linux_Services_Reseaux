# Projet 3 : Déploiement de Services Réseau sur Rocky Linux 9

## 🌐 1. Architecture et Prérequis

Ce projet déploie trois services réseau essentiels sur deux machines virtuelles (VMs) communiquant via un réseau privé Host-Only.

| Machine | Rôle | Adresse IP Statique |
| :--- | :--- | :--- |
| **VM 1** | **SRV-LABO** (Serveur) | `192.168.142.10` |
| **VM 2** | **CLI-TEST** (Client) | `192.168.142.20` |

### 1.1. Configuration de la Plateforme (VirtualBox)

  * **Adaptateurs :** Les deux VMs doivent avoir l'**Adaptateur 1** en **NAT** (pour Internet) et l'**Adaptateur 2** en **Réseau Hôte-Seul** (`192.168.142.x`).

### 1.2. Commandes de Configuration IP (Sur chaque VM)

L'interface réseau est présumée être `enp0s8`.

| VM | Commande `nmcli` |
| :--- | :--- |
| **VM 1** | `sudo nmcli con add type ethernet ifname enp0s8 con-name static-hostonly ip4 192.168.142.10/24 gw4 192.168.142.1 autoconnect yes` |
| **VM 2** | `sudo nmcli con add type ethernet ifname enp0s8 con-name static-hostonly ip4 192.168.142.20/24 gw4 192.168.142.1 autoconnect yes` |

-----

## 🛡️ 2. Configuration du Serveur (VM 1 : SRV-LABO)

### 2.1. Installation des Paquets et du Pare-feu

```bash
# Installation des services
sudo dnf install bind bind-utils nginx samba -y

# Activation et ouverture des ports essentiels (HTTPS, DNS, Samba)
sudo systemctl enable --now firewalld
sudo firewall-cmd --permanent --add-port={53/tcp,53/udp,443/tcp,445/tcp}
sudo firewall-cmd --permanent --add-service={http,ssh}
sudo firewall-cmd --reload
```

### 2.2. Service 1 : Résolution de Noms (BIND)

Le domaine **`monlabo.lan`** est utilisé pour éviter les conflits avec le domaine réservé `.local`.

#### A. Configuration du fichier de Zone (`/etc/named.conf`)

  * **Modification des options :** Assurez-vous que l'adresse IP statique est listée dans `listen-on` et le réseau entier dans `allow-query`.
    ```bash
    sudo vi /etc/named.conf
    # Remplacer : listen-on port 53 { 127.0.0.1; 192.168.142.10; };
    # Remplacer : allow-query     { localhost; 192.168.142.0/24; };
    ```
  * **Définition de la Zone :** Ajouter la zone `monlabo.lan` à la fin du fichier.

#### B. Création du fichier de Zone (`/var/named/db.monlabo.lan`)

```bash
sudo cp /var/named/named.localhost /var/named/db.monlabo.lan
# Remplir le fichier avec les enregistrements A (web, fichiers, etc.) pointant tous vers 192.168.142.10.
```

[Image of DNS zone file structure showing A records]

#### C. Démarrage et Finalisation

```bash
sudo named-checkzone monlabo.lan /var/named/db.monlabo.lan # Doit retourner OK
sudo systemctl enable --now named
```

### 2.3. Service 2 : Web Sécurisé (HTTPS/Nginx)

#### A. Certificat Auto-Signé

```bash
sudo mkdir -p /etc/nginx/ssl
# Répondre web.monlabo.lan pour le Common Name (CN)
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/nginx/ssl/labo.key -out /etc/nginx/ssl/labo.crt
```

#### B. Configuration Nginx

  * Créer le bloc de serveur HTTPS pour `web.monlabo.lan` en utilisant les chemins de certificats `/etc/nginx/ssl/labo.crt` et `labo.key`.

<!-- end list -->

```bash
sudo systemctl enable --now nginx
```

### 2.4. Service 3 : Partage de Fichiers (Samba)

#### A. Préparation

```bash
sudo mkdir -p /srv/samba/projet
sudo smbpasswd -a adrien # Crée l'utilisateur Samba
sudo chcon -t samba_share_t /srv/samba/projet # Correction SELinux
```

#### B. Configuration (`/etc/samba/smb.conf`)

  * Ajouter une définition de partage (ex: `[ProjetSecret]`) pointant vers `/srv/samba/projet` et limitant l'accès à `valid users = adrien`.

<!-- end list -->

```bash
sudo systemctl enable --now smb
```

-----

## 🧪 3. Configuration et Validation Client (VM 2 : CLI-TEST)

La section la plus cruciale est de forcer l'ordre de résolution DNS pour que le serveur local soit prioritaire.

### 3.1. Installation des outils de test

```bash
sudo dnf install bind-utils samba-client curl -y
```

### 3.2. Configuration DNS Statique et Prioritaire (La Fixe)

Pour éviter le conflit avec le DNS de l'adaptateur NAT, nous fixons le fichier et empêchons `NetworkManager` de le modifier.

```bash
# Crée le fichier resolv.conf avec l'ordre correct (DNS Labo en premier)
sudo sh -c 'echo "# Configuration Statique Forcee" > /etc/resolv.conf'
sudo sh -c 'echo "nameserver 192.168.142.10" >> /etc/resolv.conf'
sudo sh -c 'echo "nameserver 8.8.8.8" >> /etc/resolv.conf'
sudo sh -c 'echo "search monlabo.lan" >> /etc/resolv.conf'

# Rend le fichier immuable pour empêcher NetworkManager de le réécrire (Solution du conflit)
sudo chattr +i /etc/resolv.conf
```

### 3.3. Validation des Services

| Service | Commande de Test | Résultat Attendu |
| :--- | :--- | :--- |
| **DNS** | `ping -c 3 web.monlabo.lan` | Ping réussi (résolution en 192.168.142.10) |
| **HTTPS** | `curl -k https://web.monlabo.lan` | Affichage du code HTML de la page Nginx |
| **Samba** | `smbclient //web.monlabo.lan/ProjetSecret -U adrien` | Connexion réussie (`smb: \>`) |
