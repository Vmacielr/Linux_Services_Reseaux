## 🚀 Script de Déploiement : `deploy_db_backup.sh`

Ce script réalise toute la configuration initiale, y compris l'installation de MariaDB, la création d'une base de test, la création du script de dump, et sa planification.

Créez le fichier :

```bash
sudo vi deploy_db_backup.sh
```

Collez le contenu suivant :

```bash
#!/bin/bash

# ===============================================
# SCRIPT DE DÉPLOIEMENT DE SAUVEGARDE MARIADB
# Projet: Dump complet quotidien avec rotation 30 jours
# ===============================================

echo "🚀 Début du déploiement du système de sauvegarde MariaDB..."

# --- 1. INSTALLATION ET DÉMARRAGE DE MARIADB ---
echo "1/4 - Installation du serveur MariaDB..."
sudo dnf install mariadb-server -y

if [ $? -ne 0 ]; then
    echo "❌ Erreur critique : L'installation de MariaDB a échoué. Arrêt du script."
    exit 1
fi

echo "  -> Démarrage et activation du service MariaDB..."
sudo systemctl enable --now mariadb
sudo systemctl is-active mariadb && echo "  -> MariaDB est actif."

# --- 2. CRÉATION DE LA BASE DE DONNÉES DE TEST ---
DB_NAME="test_db_project"

echo "2/4 - Création de la base de données de test ($DB_NAME)..."
# Exécution de la commande SQL via le shell
# Le "-e" permet d'exécuter la commande directement.
sudo mysql -e "CREATE DATABASE IF NOT EXISTS $DB_NAME;"

# --- 3. PRÉPARATION DU DOSSIER DE SAUVEGARDE ---
DEST_DIR="/backup/db"
echo "3/4 - Création du dossier de destination : $DEST_DIR"
sudo mkdir -p $DEST_DIR

# --- 4. CRÉATION DU SCRIPT DE DUMP ET DE ROTATION ---
BACKUP_SCRIPT="/usr/local/bin/backup_db.sh"

echo "4/4 - Création du script de sauvegarde de base de données : $BACKUP_SCRIPT"

sudo sh -c "cat > $BACKUP_SCRIPT" << EOF
#!/bin/bash

# --- CONFIGURATION DB ---
DB_NAME="$DB_NAME"
DEST_DIR="$DEST_DIR"
LOG_FILE="/var/log/backup_mariadb.log"
DATE_FORMAT=\$(date +%Y%m%d)

echo "---------------------------------" >> \$LOG_FILE
echo "Début du dump MariaDB : \$(date)" >> \$LOG_FILE

# 1. Dump complet et compression GZIP
# Utilisation de 'sudo' pour garantir les droits d'accès à la DB via le socket root
sudo mysqldump \$DB_NAME | gzip > \$DEST_DIR/db_\$DATE_FORMAT.sql.gz
DUMP_STATUS=\$?

# 2. Gestion de la rotation (Rétention 30 jours)
# Recherche et suppression des fichiers plus vieux que 30 jours (-mtime +30)
echo "  -> Rotation : Suppression des dumps vieux de plus de 30 jours..."
find \$DEST_DIR/ -name "*.sql.gz" -mtime +30 -delete

# 3. Gestion des erreurs
if [ \$DUMP_STATUS -ne 0 ]; then
    MSG="ERREUR: Le dump de la base de données a échoué (Code: \$DUMP_STATUS)."
    echo "\$MSG" >> \$LOG_FILE
    echo "\$MSG" | mail -s "ALERTE CRITIQUE - DUMP DB ECHEC" root@localhost
else
    echo "  -> Dump terminé avec succès : \$DEST_DIR/db_\$DATE_FORMAT.sql.gz" >> \$LOG_FILE
    echo "Dump MariaDB terminé avec SUCCÈS." >> \$LOG_FILE
fi
EOF

# Rendre le script exécutable
sudo chmod +x $BACKUP_SCRIPT

# --- 5. PLANIFICATION CRON (À 04H00) ---
echo "5/5 - Planification de la tâche Cron (exécution à 04h00 du matin)..."

# Ajout de la tâche à la crontab de l'utilisateur root
(sudo crontab -l 2>/dev/null; echo "0 4 * * * $BACKUP_SCRIPT") | sudo crontab -

echo "✅ Déploiement du système de sauvegarde DB terminé."
echo "Procédez au guide de test pour la validation et la restauration."
```

### 0\. Lancer le script de déploiement

Exécutez-le une seule fois :

```bash
chmod +x deploy_db_backup.sh
./deploy_db_backup.sh
```

-----

## 🧪 Guide de Tests et Validation (Manuel)

Cette partie est cruciale pour valider votre stratégie de sauvegarde et de restauration.

### Test 1 : Validation du Dump et de la Compression

1.  **Créez des données de test dans la base de données :**

    ```bash
    DB_NAME="test_db_project"
    sudo mysql $DB_NAME -e "
    CREATE TABLE users (id INT AUTO_INCREMENT PRIMARY KEY, name VARCHAR(100), created_at DATETIME DEFAULT CURRENT_TIMESTAMP);
    INSERT INTO users (name) VALUES ('[votre_utilisateur]_Test'), ('Utilisateur_2');
    SELECT * FROM users;
    "
    ```

2.  **Lancez le script de sauvegarde manuellement :**

    ```bash
    sudo /usr/local/bin/backup_db.sh
    ```

3.  **Vérifiez le fichier de dump :**

      * Un fichier compressé doit exister.

    <!-- end list -->

    ```bash
    ls -lh /backup/db/
    # Vous devriez voir un fichier comme 'db_20251204.sql.gz'
    ```

4.  **Vérifiez le contenu compressé (sans le décompresser entièrement) :**

    ```bash
    zcat /backup/db/*.sql.gz | head
    # Vous devriez voir les commandes SQL (CREATE TABLE, INSERT INTO) et le nom de la DB.
    ```

### Test 2 : Validation de la Corruption et Restauration

C'est la partie la plus importante : prouver que la restauration fonctionne.

1.  **Simulez la corruption (supprimez les données critiques) :**

    ```bash
    DB_NAME="test_db_project"
    sudo mysql $DB_NAME -e "DROP TABLE users;"

    # Vérification : La commande suivante doit échouer ou être vide
    sudo mysql $DB_NAME -e "SELECT * FROM users;"
    ```

2.  **Identifiez le dernier fichier de sauvegarde :**

    ```bash
    LAST_DUMP=$(ls -t /backup/db/*.sql.gz | head -1)
    echo "Fichier de restauration : $LAST_DUMP"
    ```

3.  **Restaurez la base de données :**

      * La commande décompresse (`zcat`) et envoie (`|`) le contenu SQL directement au client `mysql`.

    <!-- end list -->

    ```bash
    sudo zcat $LAST_DUMP | sudo mysql $DB_NAME
    echo "Restauration terminée."
    ```

4.  **Vérifiez que les données sont revenues :**

    ```bash
    sudo mysql $DB_NAME -e "SELECT * FROM users;"
    # Les utilisateurs '[votre_utilisateur]_Test' et 'Utilisateur_2' doivent réapparaître.
    ```

Si les utilisateurs sont revenus, la restauration est un succès \!