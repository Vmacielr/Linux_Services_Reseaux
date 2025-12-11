Rocky :
Pourquoi ? C'est une distribution Enterprise Linux (clone de Red Hat Enterprise Linux) qui est reconnue pour sa stabilité, sa sécurité, et sa longue durée de support (LTS). Elle est idéale pour héberger des serveurs en production.

BIND :
résolution de nom (DNS) À quoi ça sert ? Cela permet aux utilisateurs de la machine cliente (CLI-TEST) d'accéder aux services par leur nom (ex: web.monlabo.lan ou fichiers.monlabo.lan) plutôt que par leur adresse IP (192.168.142.10). C'est la base de l'utilisabilité d'un réseau.
```
dig web.monlabo.lan
ping -c 3 web.monlabo.lan
```

La partie la plus délicate du projet a été de garantir que le client utilise d'abord mon serveur DNS local (192.168.142.10) et non le DNS d'Internet (via l'adaptateur NAT).
J'ai rendu le fichier /etc/resolv.conf immuable (sudo chattr +i /etc/resolv.conf) pour forcer l'ordre de résolution et empêcher NetworkManager de le modifier. Ceci est crucial pour le bon fonctionnement des services internes.
```
lsattr /etc/resolv.conf
```

Nginx :
À quoi ça sert ? Il assure la diffusion du contenu web. J'ai configuré le service en HTTPS en utilisant un certificat auto-signé.
Sécurité (HTTPS) : L'utilisation d'HTTPS crypte la communication entre le client et le serveur. Cela garantit que les données échangées (même s'il s'agit juste d'une page de test) ne peuvent pas être lues par des tiers sur le réseau.
```
curl -k https://web.monlabo.lan
```

Firewalld :
À quoi ça sert ? Il assure la sécurité du serveur en bloquant par défaut toutes les connexions entrantes non sollicitées. J'ai explicitement ouvert uniquement les ports nécessaires pour le DNS (53), le Web (443/HTTPS), Samba (445), et SSH.


Samba :
À quoi ça sert ? Il permet aux utilisateurs du réseau de monter des dossiers partagés (//web.monlabo.lan/ProjetSecret) pour stocker et accéder à des fichiers de manière centralisée et sécurisée via une authentification par utilisateur (adrien).
```
smbclient //web.monlabo.lan/ProjetSecret -U adrien
put confidentiel.txt
get confidentiel.txt
```

Rsync :
À quoi ça sert ? Il assure la résilience et la récupération du service de partage de fichiers. En cas de corruption ou de suppression accidentelle de données dans le répertoire Samba, nous disposons d'une copie de secours complète et à jour, créée avec un minimum de surcharge réseau.
```
sudo rsync -avh /srv/samba/projet/ /var/backups/samba_data/
ls -l /var/backups/samba_data/
sudo echo "bleh2" >> /srv/samba/projet/confidentiel.txt
sudo rsync -avh /srv/samba/projet/ /var/backups/samba_data/
```