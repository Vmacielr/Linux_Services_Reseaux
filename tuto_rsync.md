## 💾 Script de Déploiement : `deploy_backup.sh`

Ce script va installer les paquets, préparer les dossiers, créer le script de sauvegarde final et planifier la tâche Cron.

Créez le fichier :

```bash
sudo vi deploy_backup.sh
```

Collez le contenu suivant :

```bash
#!/bin/bash

# ===============================================
# SCRIPT DE DÉPLOIEMENT DU SYSTÈME DE SAUVEGARDE RSYNC
# Projet: Stratégie de Sauvegarde 7 jours sur Rocky Linux 9
# Auteur: Votre Nom
# Date: $(date +%Y-%m-%d)
# ===============================================

echo "🚀 Début du déploiement du système de sauvegarde Rsync..."

# --- 1. INSTALLATION DES PRÉREQUIS ---
echo "1/4 - Installation des paquets nécessaires (rsync, cronie, s-nail, postfix)..."
# Installation groupée pour Rsync, Cron (cronie), Mail (s-nail) et SMTP local (postfix)
sudo dnf install rsync cronie s-nail postfix -y

if [ $? -ne 0 ]; then
    echo "❌ Erreur critique : L'installation des paquets a échoué. Arrêt du script."
    exit 1
fi

# --- 2. DÉMARRAGE DES SERVICES ---
echo "2/4 - Activation et démarrage des services crond et postfix..."
sudo systemctl enable --now crond
sudo systemctl enable --now postfix
sudo systemctl is-active crond && echo "  -> crond (Cron) est actif."
sudo systemctl is-active postfix && echo "  -> postfix (Mail) est actif."

# --- 3. PRÉPARATION DES DOSSIERS ---
echo "3/4 - Préparation des dossiers de sauvegarde et des sources..."
# Création du dossier de destination des sauvegardes
sudo mkdir -p /backup/daily

# Création du dossier /var/www/html (pour éviter les erreurs rsync si le service n'est pas installé)
sudo mkdir -p /var/www/html

# --- 4. CRÉATION DU SCRIPT DE SAUVEGARDE FINAL ---
BACKUP_SCRIPT="/usr/local/bin/backup_script.sh"
LOG_FILE="/var/log/backup_rsync.log"
EMAIL_ADMIN="root@localhost" # Email par défaut pour les alertes locales

echo "4/4 - Création et configuration du script de sauvegarde: $BACKUP_SCRIPT"

# Utilisation de la méthode 'tee' pour écrire dans un fichier système avec sudo
sudo sh -c "cat > $BACKUP_SCRIPT" << EOF
#!/bin/bash

# --- CONFIGURATION ---
SOURCES="/home /var/www"
DEST_BASE="/backup/daily"
LOG_FILE="$LOG_FILE"
EMAIL_ADMIN="$EMAIL_ADMIN"

# Récupération du jour (rotation 7 jours)
DAY=\$(date +%A)
DEST_FINAL="\$DEST_BASE/\$DAY"

# --- SAUVEGARDE ---
echo "---------------------------------" >> \$LOG_FILE
echo "Début sauvegarde : \$(date) vers \$DEST_FINAL" >> \$LOG_FILE

# Création du dossier du jour
mkdir -p "\$DEST_FINAL"

# Commande Rsync
# -a: archive, -v: verbose, --delete: miroir, --exclude: exclusions
rsync -av --delete \\
    --exclude '*.log' \\
    --exclude 'cache/' \\
    \$SOURCES "\$DEST_FINAL" >> \$LOG_FILE 2>&1

# --- GESTION ERREUR ---
STATUS=\$?
if [ \$STATUS -ne 0 ]; then
    MSG="ERREUR de sauvegarde sur \$(hostname) (Code: \$STATUS). Voir \$LOG_FILE"
    echo "\$MSG" >> \$LOG_FILE
    echo "\$MSG" | mail -s "ALERTE BACKUP - ECHEC" \$EMAIL_ADMIN
else
    echo "Sauvegarde terminée avec SUCCÈS." >> \$LOG_FILE
fi
EOF

# Rendre le script exécutable
sudo chmod +x $BACKUP_SCRIPT

# --- 5. PLANIFICATION CRON ---
echo "5/5 - Planification de la tâche Cron (exécution à 03h00 du matin)..."

# Ajout de la tâche à la crontab de l'utilisateur root
(sudo crontab -l 2>/dev/null; echo "0 3 * * * $BACKUP_SCRIPT") | sudo crontab -

echo "✅ Déploiement terminé avec succès."
echo "Le script de sauvegarde ($BACKUP_SCRIPT) a été créé et planifié."
echo "Veuillez maintenant procéder aux tests de validation (voir instructions séparées)."

# Afficher la crontab pour confirmation
sudo crontab -l

```

### 0\. Lancer le script de déploiement

Exécutez-le une seule fois :

```bash
chmod +x deploy_backup.sh
./deploy_backup.sh
```

-----

## 🧪 Guide de Tests et Validation

Une fois le script de déploiement exécuté, utilisez ces étapes pour valider le bon fonctionnement de la stratégie de sauvegarde.

### Test 1 : Validation de l'exécution manuelle

Vérifiez que le script de sauvegarde s'exécute correctement et écrit un log de succès.

1.  **Lancez la sauvegarde manuellement :**

    ```bash
    sudo /usr/local/bin/backup_script.sh
    ```

2.  **Vérifiez le journal (log) :**

    ```bash
    cat /var/log/backup_rsync.log
    # La dernière ligne doit indiquer : "Sauvegarde terminée avec SUCCÈS."
    ```

3.  **Vérifiez le dossier de destination :**

    ```bash
    ls /backup/daily/$(date +%A)/
    # Vous devriez voir les dossiers 'home' et 'var'
    ```

### Test 2 : Validation de la Restauration (Réponse à "Testez la restauration d'un fichier supprimé")

Simulez une perte de données pour prouver que le processus de restauration est fonctionnel.

1.  **Créez un fichier précieux :**

    ```bash
    echo "Document a recuperer absolument" > /home/[votre_utilisateur]/fichier_precieux.txt
    ```

2.  **Lancez une sauvegarde (pour copier le fichier) :**

    ```bash
    sudo /usr/local/bin/backup_script.sh
    ```

3.  **Simulez la perte (suppression) :**

    ```bash
    rm /home/[votre_utilisateur]/fichier_precieux.txt
    ```

4.  **Restaurez le fichier depuis la sauvegarde :**
    *Le jour actuel est stocké dans la variable `$(date +%A)`.*

    ```bash
    sudo cp /backup/daily/$(date +%A)/home/[votre_utilisateur]/fichier_precieux.txt /home/[votre_utilisateur]/
    ```

5.  **Vérifiez le contenu du fichier restauré :**

    ```bash
    cat /home/[votre_utilisateur]/fichier_precieux.txt
    # Doit afficher : "Document a recuperer absolument"
    ```

Votre projet est maintenant entièrement déployé et validé \!