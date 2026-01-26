### 🐧 Rocky Linux : Pourquoi ?

C'est une distribution **Entreprise Linux** (clone de RHEL) reconnue pour sa stabilité, sa sécurité renforcée (via SELinux) et sa longue durée de support (LTS). Elle est le standard pour héberger des serveurs en production où la fiabilité est la priorité absolue.

### 🌐 BIND : Résolution de nom (DNS)

**À quoi ça sert ?** C'est la colonne vertébrale du réseau. Il permet aux utilisateurs et aux services d'accéder aux machines par leur nom (ex: `web.monlabo.lan`) plutôt que par leur adresse IP. J'ai aussi configuré le **Forwarding** pour que les serveurs isolés puissent accéder à Internet via le DNS.

```bash
sudo systemctl status named
dig web.monlabo.lan
nslookup web.monlabo.lan
ping google.com

```

### 🔒 Firewalld : La Sentinelle

**À quoi ça sert ?** Il assure la sécurité périmétrale en bloquant par défaut toutes les connexions entrantes. J'ai appliqué une politique de **moindre privilège** en n'ouvrant que les ports strictement nécessaires (DNS, HTTPS, Samba, SSH et les flux de monitoring).

```bash
firewall-cmd --list-all

```

### 🚀 Nginx : Serveur Web Haute Performance

**À quoi ça sert ?** Il diffuse le contenu web de manière sécurisée. J'ai forcé l'utilisation du protocole **HTTPS** avec un certificat auto-signé généré via OpenSSL.
**Sécurité :** Même en environnement de test, le chiffrement garantit que les données échangées entre le client et le serveur ne sont pas lisibles en clair sur le réseau.

```bash
curl -kI https://web.monlabo.lan

```

### 📁 Samba : Partage de Fichiers Centralisé

**À quoi ça sert ?** Il permet un stockage collaboratif sécurisé. L'accès au dossier `ProjetSecret` est restreint par une authentification utilisateur (`adrien`) et protégé par des contextes **SELinux** (`samba_share_t`) pour éviter toute fuite de données hors du répertoire.

```bash
smbclient //web.monlabo.lan/ProjetSecret -U adrien
put test-presentation.txt
get confidentiel.txt

```

### 🔄 Rsync & Cron : Sauvegarde et Automatisation

**À quoi ça sert ?** Rsync assure la **résilience** des données en synchronisant le dossier Samba vers **SRV-BACKUP**. L'utilisation du **Cron** automatise cette tâche chaque nuit à 03h00, rendant la stratégie de sauvegarde autonome et fiable.

```bash
# Test manuel avec gestion des permissions (fix --no-perms)
rsync -rtv --no-perms --no-owner --no-group /srv/samba/projet/ adrien@192.168.142.12:/mnt/sauvegardes/srv-apps/
# Vérifier la programmation automatique
sudo crontab -l

```

### 📊 Grafana, Prometheus & Loki : L'Observabilité Totale

**À quoi ça sert ?** On ne se contente plus de surveiller, on observe. **Prometheus** collecte les métriques (CPU/RAM), tandis que **Loki** (via Promtail) centralise tous les logs système.
**Corrélation :** Cela permet de lier un pic de charge (métrique) à un événement précis (log), comme l'arrêt d'un service.

```bash
stress-ng --cpu 4 --timeout 150s
sudo setenforce 0

sudo systemctl stop nginx


```

---

### 💡 Le petit "plus" pour ton oral

"Grâce à cette stack, j'ai transformé une administration réactive en une **gestion proactive**. Si Nginx s'arrête, je reçois une alerte Prometheus et je lis instantanément la cause dans les logs Loki sans même ouvrir un terminal".

**Souhaites-tu que je te rédige une conclusion percutante de 30 secondes pour clore ta présentation en beauté ?**