## 🏗️ Introduction : Pourquoi Rocky Linux ?

* **Standard Entreprise** : Rocky Linux est la suite logique de CentOS, offrant une compatibilité totale avec RHEL (Red Hat Enterprise Linux).
* **Stabilité et Sécurité** : Il inclut nativement **SELinux** et des outils de gestion réseau robustes indispensables en production.

---

## 🌐 1. SRV-DNS : Le Cerveau du Réseau (192.168.142.10)

**Rôle** : Centraliser la résolution de noms pour éviter de gérer des adresses IP complexes.

| Action | Commande (sur **SRV-DNS**) | Pourquoi / Bénéfice |
| --- | --- | --- |
| **Vérifier BIND** | `sudo systemctl status named` | Utilisation du standard mondial pour la résolution de noms. |
| **Test Client** | (Sur **CLI-TEST**) : `nslookup web.monlabo.lan` | **Bénéfice** : Facilité de maintenance. Si l'IP change, on ne modifie qu'un seul fichier zone. |
| **Forwarding** | (Sur **CLI-TEST**) : `ping google.com` | **Bénéfice** : Les serveurs isolés peuvent se mettre à jour via le DNS sans exposition directe. |

---

## 🚀 2. SRV-APPS : Le Cœur des Services (192.168.142.11)

**Rôle** : Héberger les applications web, le partage de fichiers et la pile d'observabilité.

| Action | Commande (sur **SRV-APPS**) | Pourquoi / Bénéfice |
| --- | --- | --- |
| **Web HTTPS** | `curl -kI https://web.monlabo.lan` | **Sécurité** : Chiffrement des flux avec certificats auto-signés via OpenSSL. |
| **Conteneurs** | `podman-compose ps` | **Sécurité** : Podman est "Rootless", limitant l'impact en cas de compromission. |
| **LXC** | `sudo lxc-ls -f` | **Efficience** : Isolation de type système (OS complet) sans la lourdeur d'une VM classique. |

---

## 🛡️ 3. SRV-BACKUP : La Résilience (192.168.142.12)

**Rôle** : Stockage sécurisé et isolé pour la survie des données.

| Action | Commande (sur **SRV-APPS**) | Pourquoi / Bénéfice |
| --- | --- | --- |
| **Sauvegarde** | `rsync -az --delete /srv/samba/projet/ [USER]@192.168.142.12:/mnt/sauvegardes/` | **Performance** : Rsync ne transfère que les différences, économisant la bande passante. |
| **Sécurité SSH** | (Sur **SRV-BACKUP**) : `ss -tuln` | **Isolation** : Seul le port 22 (SSH) est ouvert, réduisant la surface d'attaque. |

---

## 📊 4. Observabilité Totale (Prometheus & Loki)

**Rôle** : Surveiller la santé (Métriques) et comprendre les incidents (Logs).

| Action | Commande / Interface | Pourquoi / Bénéfice |
| --- | --- | --- |
| **Métriques** | `http://192.168.142.11:9090` | **Proactivité** : Surveillance des ressources (CPU/RAM) en temps réel avec Prometheus. |
| **Logs (Loki)** | **Grafana Explore** : `{job="varlogs"}` | **Diagnostic rapide** : Centralisation des journaux système pour identifier les causes racines. |
| **Incident** | `sudo systemctl stop nginx` | **Démo** : Montre l'alerte de service et la ligne "Stopped Nginx" dans Loki instantanément. |

---

## 🏁 Conclusion de la présentation

> "En conclusion, cette architecture sur **Rocky Linux** démontre une infrastructure moderne et cloisonnée. Grâce à l'intégration de **Prometheus** pour les métriques et de **Loki** pour les logs, nous avons une visibilité complète. Nous ne nous contentons pas de savoir *quand* un service tombe, nous comprenons *pourquoi*, tout en garantissant la sécurité via **SELinux** et l'isolation des sauvegardes."

---
