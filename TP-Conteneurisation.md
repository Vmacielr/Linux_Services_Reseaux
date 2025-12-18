# 📔 Journal de Bord : Conteneurisation sur SRV-APPS

Ce document retrace les manipulations effectuées sur la machine **SRV-APPS (192.168.142.11)** pour l'apprentissage de Podman et Docker Compose.

---

## 🟢 TP 1 : Mon Premier Conteneur (Nginx)

**Objectif :** Déployer un serveur web isolé et comprendre le mappage de ports.

### Commandes principales

* **Lancer le conteneur :**
```bash
podman run -d --name web -p 8080:80 docker.io/library/nginx

```


* **Vérifier les conteneurs actifs :**
```bash
podman ps

```


* **Voir tous les conteneurs (même arrêtés) :**
```bash
podman ps -a

```


* **Consulter les logs (accès web, erreurs) :**
```bash
podman logs web

```


* **Entrer dans le conteneur pour modifier des fichiers :**
```bash
podman exec -it web bash
# Une fois dedans : echo "titre" > /usr/share/nginx/html/index.html

```


* **Supprimer le conteneur :**
```bash
podman rm -f web

```



> [!NOTE]
> **Leçon apprise :** Les conteneurs sont éphémères. Si on supprime le conteneur, les modifications faites avec `exec` disparaissent.

---

## 🔵 TP 2 : Construction d'Image (Dockerfile)

**Objectif :** Créer une image personnalisée "immortelle" contenant déjà notre code.

### Préparation des fichiers

1. **`index.html`** : Contenu personnalisé du site.
2. **`Dockerfile`** : La recette de construction.
```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80

```



### Commandes de construction et test

* **Renommer le fichier (si erreur de casse) :**
```bash
mv DockerFile Dockerfile

```


* **Construire l'image :**
```bash
podman build -t mon-site:v1 .

```


* **Lister les images stockées :**
```bash
podman images

```


* **Lancer un conteneur basé sur cette image :**
```bash
podman run -d --name mon-site-v1 -p 8081:80 mon-site:v1

```



---

## 🟡 TP 3 : Orchestration Multi-Services (Compose)

**Objectif :** Déployer une stack complexe (WordPress + MySQL) avec persistance des données.

### Installation de l'outil

```bash
sudo dnf install podman-compose -y

```

### Fichier `docker-compose.yml`

Ce fichier définit deux services (`db` et `wordpress`) et deux volumes nommés (`db_data` et `wp_data`) pour éviter la perte de données lors du `down`.

### Commandes Compose

* **Démarrer toute la stack :**
```bash
podman-compose up -d

```


* **Arrêter et supprimer les conteneurs :**
```bash
podman-compose down

```


* **Vérifier le statut des services :**
```bash
podman-compose ps

```



---

### 🔍 Rappel Sécurité (Firewalld)

Pour accéder à ces services depuis ton PC hôte ou la VM `CLI-TEST`, n'oublie pas d'ouvrir les ports :

```bash
sudo firewall-cmd --permanent --add-port={8080/tcp,8081/tcp,8082/tcp}
sudo firewall-cmd --reload

```

---

C'est une excellente idée pour garder ton journal de bord à jour. Voici la suite de ton README, couvrant les aspects sécurité de Podman et la découverte de LXC.

---

## 🟠 TP 4 : Sécurité et Podman (Rootless & Pods)

**Objectif :** Comprendre l'isolation utilisateur et le regroupement de conteneurs dans un "Pod".

### Commandes principales

* **Vérifier le mappage des UID (Rootless) :**
```bash
podman run --rm alpine cat /proc/self/uid_map

```


* **Créer un Pod (groupe de conteneurs partageant le réseau) :**
```bash
podman pod create --name mon-pod -p 8083:80

```


* **Ajouter des conteneurs dans le Pod :**
```bash
podman run -d --pod mon-pod --name serveur-web docker.io/library/nginx
podman run -d --pod mon-pod --name base-cache docker.io/library/redis

```


* **Gérer le Pod via Systemd (Mode Utilisateur) :**
```bash
mkdir -p ~/.config/systemd/user/
podman generate systemd --name mon-pod --files --new
mv *.service ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable --now pod-mon-pod.service

```



> [!NOTE]
> **Leçon apprise :** Dans un Pod, les conteneurs peuvent se parler via `localhost` (ex: Nginx peut contacter Redis sur `localhost:6379`).

---

## 🔴 TP 5 : Conteneurs Systèmes (LXC)

**Objectif :** Déployer un OS complet (conteneur système) et comprendre la différence avec les conteneurs applicatifs.

### Préparation du réseau (sur Rocky Linux)

* **Création manuelle du pont réseau (bridge) :**
```bash
sudo nmcli con add type bridge ifname lxcbr0 con-name lxcbr0
sudo nmcli con mod lxcbr0 ipv4.addresses 10.0.3.1/24 ipv4.method manual
sudo nmcli con up lxcbr0

```



### Gestion du conteneur

* **Créer un conteneur Ubuntu :**
```bash
sudo lxc-create -t download -n ubuntu-test -- --dist ubuntu --release jammy --arch amd64

```


* **Démarrer et vérifier :**
```bash
sudo lxc-start -n ubuntu-test
sudo lxc-ls -f

```


* **Accéder au shell du conteneur (sans mot de passe) :**
```bash
sudo lxc-attach -n ubuntu-test

```


* **Arrêt forcé (si plantage) :**
```bash
sudo lxc-stop -n ubuntu-test -k

```



> [!NOTE]
> **Leçon apprise :** LXC lance un système complet avec son propre `systemd` et de nombreux processus, contrairement à Podman qui ne lance qu'un seul binaire.

---

## 🟣 TP 6 : La Plomberie (containerd)

**Objectif :** Manipuler le moteur de bas niveau utilisé par Kubernetes et comprendre l'isolation des runtimes.

### Installation (sur Rocky/RHEL)

```bash
# Ajout du dépôt Docker-CE pour obtenir les outils
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install containerd.io -y
sudo systemctl enable --now containerd

```

### Commandes de "bas niveau" (ctr)

* **Télécharger une image (Full Name requis) :**
```bash
sudo ctr images pull docker.io/library/alpine:latest

```


* **Lancer un conteneur et sa tâche associée :**
```bash
sudo ctr run -d docker.io/library/alpine:latest mon-test

```


* **Lister les processus (Tasks) réels :**
```bash
sudo ctr tasks ls

```


* **Supprimer proprement :**
```bash
sudo ctr task kill -s SIGKILL mon-test
sudo ctr task rm mon-test
sudo ctr container rm mon-test

```



> [!IMPORTANT]
> **Leçon apprise :** Podman et containerd ne partagent pas le même stockage. Une image téléchargée avec `ctr` n'est pas visible par `podman images`.

---
