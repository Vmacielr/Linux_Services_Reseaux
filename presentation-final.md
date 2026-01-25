## 🟢 Introduction : Pourquoi Rocky Linux ?

Avant de toucher au clavier, explique ton choix d'OS :

* **Stabilité Entreprise :** Rocky Linux est le successeur spirituel de CentOS. Il est 100% compatible avec RHEL (Red Hat Enterprise Linux).
* **Sécurité Native :** Il intègre nativement des outils de sécurité avancés comme **SELinux** et **Firewalld**, indispensables en milieu professionnel.
* **Professionnalisme :** Apprendre sur Rocky Linux, c'est se préparer aux environnements de production réels (banques, serveurs d'État, grandes entreprises).

---

## 🧠 1. VM 1 : SRV-DNS (Le Cerveau)

**Rôle :** Traduire les noms en adresses IP et gérer les accès internet.

| Commande (sur **SRV-DNS**) | Pourquoi cette technologie ? | En quoi c'est bien ? |
| --- | --- | --- |
| `sudo systemctl status named` | Utilisation de **BIND9**, le standard mondial du DNS. | **Fiabilité :** C'est l'outil le plus documenté et le plus robuste au monde. |
| **Passer sur VM 4 (CLI-TEST) :** `dig srv-apps.monlabo.lan` | Utilisation d'une **Zone de recherche directe**. | **Scalabilité :** Plus besoin de retenir des IPs. On ajoute un serveur ? On met à jour le DNS, et tout le monde le trouve par son nom. |

---

## 🚀 2. VM 2 : SRV-APPS (Le Cœur des Services)

**Rôle :** Héberger les sites web, les fichiers et les conteneurs.

| Commande (sur **SRV-APPS**) | Pourquoi cette technologie ? | En quoi c'est bien ? |
| --- | --- | --- |
| `podman-compose ps` | **Podman** (au lieu de Docker). | **Sécurité :** Podman est "rootless" (ne tourne pas en admin). Si le conteneur est piraté, l'attaquant reste bloqué sans droits sur la VM. |
| `sudo lxc-ls -f` | **LXC** (Conteneurs systèmes). | **Densité :** On simule un OS complet (Ubuntu) sans la lourdeur d'une VM. On peut faire tourner 50 conteneurs LXC là où on ferait tourner 5 VMs. |
| `sudo getsebool -a | grep samba` | **SELinux** activé sur Samba. |

---

## 💾 3. VM 3 : SRV-BACKUP (Le Coffre-fort)

**Rôle :** Stocker les données de manière isolée.

| Commande (sur **SRV-APPS**) | Pourquoi cette technologie ? | En quoi c'est bien ? |
| --- | --- | --- |
| `sudo rsync -az --delete /srv/samba/projet/ ...` | **Rsync via SSH**. | **Efficience :** Rsync ne transfère que les parties de fichiers qui ont changé (incrémental). C'est rapide et ça ne sature pas le réseau. |
| **Sur VM 3 (SRV-BACKUP) :** `ls -lh /mnt/sauvegardes/srv-apps/` | **Stockage déporté**. | **Résilience :** En cas d'incendie ou de crash total de SRV-APPS, l'entreprise peut redémarrer en quelques minutes grâce à cette copie isolée. |

---

## 📊 4. Supervision (Observabilité)

**Outils :** Prometheus & Grafana sur **SRV-APPS**.

| Commande (sur **SRV-APPS**) | Pourquoi cette technologie ? | En quoi c'est bien ? |
| --- | --- | --- |
| `stress-ng --cpu 4 --timeout 20s` | Simulation de charge CPU. | **Proactivité :** On ne répare pas quand ça casse, on surveille pour intervenir *avant* que ça ne casse. |
| **Sur ton navigateur :** Montre les alertes Prometheus. | **Alerting automatique**. | **Tranquillité :** L'administrateur reçoit une alerte dès qu'un seuil critique (95% disque) est atteint. |

---

## 🏁 Conclusion de la présentation

Pour terminer en beauté, adresse-toi au jury ainsi :

> "Pour conclure, ce projet démontre une infrastructure **moderne et souveraine**.
> * **Moderne** par l'utilisation de la conteneurisation (Podman, LXC) et du monitoring en temps réel.
> * **Souveraine** car basée sur des technologies Open Source éprouvées, sans dépendance logicielle coûteuse.
> 
> 
> La force de cette architecture réside dans son **cloisonnement** : chaque service a son rôle, chaque accès est sécurisé par un pare-feu et SELinux, et chaque donnée est protégée par une sauvegarde automatisée. C'est une base solide, prête à évoluer vers le Cloud ou le Multi-site."

---
