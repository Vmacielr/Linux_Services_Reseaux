## 🚀 Script de Déploiement : `deploy_borg_setup.sh`

Ce script installe BorgBackup, prépare l'environnement, et crée un script de sauvegarde quotidien qui gère la rotation et la maintenance du dépôt.

Créez le fichier :

```bash
sudo vi deploy_borg_setup.sh
```

Collez le contenu suivant :

```bash
#!/bin/bash

# ===============================================
# SCRIPT DE DÉPLOIEMENT BORG
# Projet: BorgBackup chiffré et dédupliqué
# ===============================================

echo "🚀 Début du déploiement de BorgBackup..."

# --- 1. INSTALLATION DES PRÉREQUIS ---
echo "1/3 - Installation de BorgBackup..."
# Sur Rocky 9, Borg est généralement disponible via DNF.
sudo dnf install borgbackup -y

if [ $? -ne 0 ]; then
    echo "❌ Erreur critique : L'installation de BorgBackup a échoué. Arrêt du script."
    exit 1
fi

# --- 2. PRÉPARATION DES DOSSIERS ---
SOURCE_DIR="/home"
REPO_DIR="/mnt/borg_repo"
BACKUP_SCRIPT="/usr/local/bin/backup_borg.sh"

echo "2/3 - Préparation du dépôt Borg et du dossier source..."
# Création du dossier du dépôt Borg (Simule un point de montage externe)
sudo mkdir -p $REPO_DIR

# --- 3. CRÉATION ET INITIALISATION DU DÉPÔT ---
echo "  -> Création et initialisation du dépôt chiffré Borg. (Entrez un mot de passe fort)"
# Initialisation du dépôt Borg avec chiffrement 'repokey-blake2'
# La méthode 'repokey' stocke la clé dans le dépôt chiffré par un mot de passe.
sudo borg init --encryption=repokey-blake2 $REPO_DIR

# --- 4. CRÉATION DU SCRIPT DE SAUVEGARDE QUOTIDIEN ---
echo "3/3 - Création du script de sauvegarde quotidien : $BACKUP_SCRIPT"

sudo sh -c "cat > $BACKUP_SCRIPT" << EOF
#!/bin/bash

# --- CONFIGURATION BORG ---
REPO_DIR="$REPO_DIR"
SOURCE_DIR="$SOURCE_DIR"
LOG_FILE="/var/log/borg_backup.log"

# Nom de l'archive basée sur la date (ex: 2025-12-04_1200)
ARCHIVE_NAME="::\$(hostname)-\$(date +\%Y-\%m-\%d_\%H\%M)"

echo "---------------------------------" >> \$LOG_FILE
echo "Début de la sauvegarde Borg : \$(date)" >> \$LOG_FILE

# 1. Sauvegarde (Le mot de passe Borg sera demandé manuellement la première fois, ou via un fichier secret)
# Note: Pour une automatisation complète via cron, vous devriez utiliser la variable BORG_PASSCOMMAND ou BORG_PASSPHRASE.
# Pour l'exercice, nous utiliserons BORG_PASSPHRASE pour le test manuel, mais il doit être retiré pour Cron
# ou géré via une méthode sécurisée (Keyfile).

echo "Sauvegarde de \$SOURCE_DIR vers \$REPO_DIR avec l'archive \$ARCHIVE_NAME..." >> \$LOG_FILE
sudo borg create --stats --progress \$ARCHIVE_NAME \$SOURCE_DIR >> \$LOG_FILE 2>&1
BORG_STATUS=\$?

if [ \$BORG_STATUS -ne 0 ]; then
    MSG="ERREUR Borg: Sauvegarde échouée (Code: \$BORG_STATUS). Voir \$LOG_FILE"
    echo "\$MSG" | mail -s "ALERTE BORG - ÉCHEC" root@localhost
else
    echo "Sauvegarde Borg terminée avec succès." >> \$LOG_FILE
fi

# 2. Maintenance et Rotation
# Conserve 7 archives quotidiennes et 4 archives hebdomadaires
echo "Nettoyage des anciennes archives..." >> \$LOG_FILE
sudo borg prune -v --list \$REPO_DIR --keep-daily 7 --keep-weekly 4 >> \$LOG_FILE 2>&1
EOF

# Rendre le script exécutable
sudo chmod +x $BACKUP_SCRIPT

# --- 5. PLANIFICATION CRON (Exécution quotidienne à 01h00) ---
echo "5/5 - Planification de la tâche Cron (exécution à 01h00 du matin)..."
# Nécessite que cronie soit déjà installé (fait dans Exercice 2)

(sudo crontab -l 2>/dev/null; echo "0 1 * * * /usr/local/bin/backup_borg.sh") | sudo crontab -

echo "✅ Déploiement Borg terminé."
echo "Procédez à l'initialisation manuelle (étape suivante) puis aux tests de déduplication."
```

### 0\. Lancer le script de déploiement

Exécutez-le une seule fois :

```bash
chmod +x deploy_borg_setup.sh
./deploy_borg_setup.sh
```

-----

## 🔑 Guide d'Initialisation et de Test BorgBackup

Borg nécessite la saisie du mot de passe de chiffrement lors de la première utilisation.

### Étape 1 : Création de la première sauvegarde (J1)

1.  **Créez un fichier test de grande taille** pour observer la déduplication plus tard :

    ```bash
    echo "Premiere version du document." > /home/document_important.txt
    # Crée un fichier de 50 Mo de données aléatoires pour simuler un gros fichier
    dd if=/dev/urandom of=/home/gros_fichier_A.bin bs=1M count=50
    ```

2.  **Lancez le script de sauvegarde (J1)**
    *Lorsqu'il vous le demandera, entrez le mot de passe de chiffrement que vous avez défini lors de l'étape `borg init`.*

    ```bash
    sudo /usr/local/bin/backup_borg.sh
    ```

3.  **Vérifiez l'archive :**

    ```bash
    sudo borg list /mnt/borg_repo
    ```

### Étape 2 : Création de la deuxième sauvegarde (J2 - Déduplication en action)

1.  **Modifiez légèrement le fichier source :**

    ```bash
    echo "Deuxieme version du document, avec une petite modification." >> /home/document_important.txt
    ```

    *Laissez le gros fichier de 50 Mo intact.*

2.  **Lancez le script de sauvegarde (J2) :**

    ```bash
    sudo /usr/local/bin/backup_borg.sh
    ```

3.  **Observez la déduplication :**
    Affichez les statistiques du dépôt.

    ```bash
    sudo borg info /mnt/borg_repo
    ```

      * **Interprétation :** Vous verrez deux lignes importantes :
          * `This archive size:` (Taille de l'archive J2) : sera 50 Mo + la modification.
          * `Total data stored:` (Taille totale unique dans le dépôt) : sera très proche de la taille de J1.
      * **Conclusion :** Borg n'a pas copié à nouveau les 50 Mo de `gros_fichier_A.bin`, il a seulement stocké les *blocs* qui avaient changé dans `document_important.txt`.

### Étape 3 : Restauration d'une version spécifique

1.  **Listez les archives (versions) disponibles :**

    ```bash
    sudo borg list /mnt/borg_repo
    # Identifiez le nom de l'archive du J1 (celle qui a la première version du document).
    ```

2.  **Simulez la suppression du fichier original :**

    ```bash
    rm /home/document_important.txt
    ```

3.  **Restaurez la version spécifique de J1 :**

      * Remplacez `[NOM_ARCHIVE_J1]` par le nom réel de l'archive J1 (ex: `hostname-2025-12-04_1148`).
      * Nous restaurons le fichier dans un dossier temporaire (`/tmp/restore_test`).

    <!-- end list -->

    ```bash
    sudo borg extract /mnt/borg_repo::[NOM_ARCHIVE_J1] home/document_important.txt --to /tmp/restore_test/
    ```

4.  **Vérifiez le contenu de la version restaurée :**

    ```bash
    cat /tmp/restore_test/home/document_important.txt
    ```

      * **Résultat attendu :** Vous verrez uniquement le texte : `"Premiere version du document."`, prouvant que la restauration à la version J1 a fonctionné.

L'exercice est validé si les commandes `borg info` montrent la déduplication et si la restauration ramène l'ancienne version.