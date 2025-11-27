# Projet d'Automatisation pour PC Industriels HAI-P200-4G

Ce projet fournit des outils pour **automatiser l'installation, la sauvegarde et la restauration** des applications de supervision (Node-RED, Grafana, PostgreSQL et **Ignition Edge**) sur les PC industriels **HAI-P200-4G** de notre société, Hexa-ai.

> **Note importante :** Cet outil agit uniquement sur les applications listées ci-dessus et leurs données. Il ne modifie pas la configuration de base du système d'exploitation du PC industriel.

---

## Clause de non-responsabilité

Cet outil est fourni gratuitement et en tant que logiciel open-source. Il est de la **responsabilité de l'utilisateur** de le tester et de le valider de manière approfondie avant toute utilisation dans un environnement de production.

L'auteur et les contributeurs ne sauraient être tenus responsables de toute perte de données, interruption de service, ou tout autre dommage direct ou indirect résultant de l'utilisation de ce logiciel. Vous êtes libre de l'adapter et de le modifier selon vos besoins, à vos propres risques.

---

## À qui s'adresse ce projet ?

Vous êtes automaticien, constructeur de machine ou intégrateur ? Ce projet est fait pour vous.

**Vos bénéfices :**
-   **Gagner du temps :** Déployez une configuration complète en une seule commande et sur plusieurs équipements.
-   **Fiabilité :** Évitez les erreurs de configuration manuelle et assurez une installation identique sur tous les PC.
-   **Maintenance simplifiée :** Sauvegardez et restaurez l'état complet des **applications de supervision** d'un PC en quelques minutes, idéal pour la maintenance ou le remplacement de matériel.

---

## Prérequis

Pour utiliser ces outils, vous avez besoin d'un environnement Linux. Si vous êtes sur Windows, WSL (Sous-système Windows pour Linux) est la solution recommandée.

Une fois dans votre terminal Linux (ou WSL), assurez-vous qu'Ansible est installé :
```bash
# Commande d'installation pour les systèmes basés sur Debian/Ubuntu
sudo apt update && sudo apt install ansible -y
```

---

## Cas d'Usage Principaux

### 1. Déployer une configuration sur un nouveau PC

Ce scénario installe et configure toutes les applications avec une configuration standard.

1.  **Ajoutez le PC à l'inventaire :** Ouvrez le fichier `inventory.ini` et ajoutez l'adresse IP et les identifiants du PC cible.

    ```ini
    [industrial_pcs]
    192.168.1.16 ansible_user=admin ansible_password='VOTRE_MOT_DE_PASSE' ansible_python_interpreter=/usr/bin/python3.11
    ```

2.  **Lancez le déploiement :** Exécutez la commande suivante. Elle appliquera la configuration qui se trouve dans le dossier `deploy/default/`.

    ```bash
    ANSIBLE_CONFIG=./ansible.cfg ansible-playbook -i inventory.ini deploy_playbook.yml
    ```
    > **Note :** La partie `ANSIBLE_CONFIG=./ansible.cfg` est nécessaire si vous travaillez depuis WSL pour que la configuration du projet soit bien prise en compte.

### 2. Déployer une configuration personnalisée

Imaginez que vous avez une machine spéciale qui nécessite des flux Node-RED, des projets Ignition Edge ou des dashboards Grafana différents.

1.  **Créez un "profil" de déploiement :** Copiez le dossier `deploy/default` et renommez-le, par exemple en `deploy/machine_speciale`.

2.  **Modifiez la configuration :** Adaptez les fichiers dans votre nouveau dossier `deploy/machine_speciale/` selon vos besoins.

3.  **Associez le profil au PC :** Dans `inventory.ini`, ajoutez la variable `deployment_profile` à la ligne du PC concerné.

    ```ini
    # Ce PC utilisera la configuration du dossier deploy/machine_speciale/
    192.168.1.17 ansible_user=admin ... deployment_profile=machine_speciale
    ```

4.  **Lancez le déploiement** avec la même commande que précédemment. Ansible détectera et appliquera automatiquement le bon profil.

### 3. Sauvegarder un PC

Cette opération crée une archive `.zip` complète contenant les configurations, les données et les volumes des applications du PC distant.

1.  **Lancez la sauvegarde :** Exécutez la commande suivante, en ciblant le PC à sauvegarder avec `--limit`.

    ```bash
    ANSIBLE_CONFIG=./ansible.cfg ansible-playbook -i inventory.ini backup_playbook.yml --limit 192.168.1.16
    ```

2.  **Récupérez l'archive :** La sauvegarde est automatiquement téléchargée depuis le PC et stockée dans le dossier `backups/` de ce projet.

#### Option pour PostgreSQL : Sauvegarder les données

Par défaut, la sauvegarde de PostgreSQL n'inclut que la **structure** des bases de données (le "schéma"), mais **pas les données** qu'elles contiennent. C'est une opération rapide et légère, idéale pour recréer un environnement vide.

Pour effectuer une **sauvegarde complète** qui inclut toutes les données de PostgreSQL, ajoutez l'option `-e "backup_postgres_volume=true"` à la commande :
```bash
ANSIBLE_CONFIG=./ansible.cfg ansible-playbook -i inventory.ini backup_playbook.yml --limit 192.168.1.16 -e "backup_postgres_volume=true"
```
Cette sauvegarde sera plus volumineuse mais permettra une restauration complète de l'état de la base de données.

### 4. Restaurer un PC

Utilisez une sauvegarde pour remettre un PC dans un état antérieur ou pour cloner une configuration sur un nouveau matériel.

1.  **Identifiez le fichier de sauvegarde** dans le dossier `backups/` (par ex: `backup-192.168.1.16-20251126T153000.zip`).

2.  **Lancez la restauration :** La commande nécessite deux informations cruciales :
    -   `-e "backup_file=..."` pour spécifier le nom du fichier de sauvegarde.
    -   `--limit ...` pour vous assurer de restaurer sur le bon PC.

    ```bash
    ANSIBLE_CONFIG=./ansible.cfg ansible-playbook -i inventory.ini restore_playbook.yml -e "backup_file=backup-192.168.1.16-20251126T153000.zip" --limit 192.168.1.16
    ```
    > **Attention :** La restauration efface les données et configurations actuelles du PC cible pour les remplacer par celles de la sauvegarde.

---
## Tutoriel : Préparer un Profil de Déploiement Personnalisé

Le cas d'usage le plus puissant de cet outil est de pouvoir répliquer une configuration complexe et validée sur de nouveaux PC en une seule commande. Ce guide vous montre comment transformer un PC configuré manuellement en un "profil de déploiement" réutilisable.

Le principe est le suivant :
1.  Vous configurez un PC "modèle" avec vos flux Node-RED, vos dashboards Grafana et vos bases de données PostgreSQL.
2.  Vous utilisez le playbook de **sauvegarde** pour extraire toute la configuration.
3.  Vous organisez les fichiers extraits dans un nouveau dossier de profil (ex: `deploy/ma_super_machine/`).
4.  Vous ajustez les playbooks pour automatiser la création des bases de données et des connexions.

### Étape 1 : Développer et configurer votre PC modèle

Travaillez directement sur un PC industriel HAI-P200-4G.
-   **Node-RED** : Créez vos flux, installez les palettes nécessaires depuis l'interface.
-   **Grafana** : Créez vos dashboards et, surtout, configurez manuellement la source de données (datasource) pour vous connecter à votre base PostgreSQL.
-   **PostgreSQL** : Créez vos bases de données et vos tables via pgAdmin ou un autre outil.

### Étape 2 : Sauvegarder la configuration

Une fois que tout fonctionne comme vous le souhaitez, lancez une sauvegarde complète de ce PC. Assurez-vous d'inclure les données PostgreSQL avec l'option `-e "backup_postgres_volume=true"`.

```bash
ANSIBLE_CONFIG=./ansible.cfg ansible-playbook -i inventory.ini backup_playbook.yml --limit <IP_DU_PC_MODELE> -e "backup_postgres_volume=true"
```

Vous obtiendrez une archive `backup-<IP>-<DATE>.zip` dans votre dossier `backups/`.

### Étape 3 : Créer et remplir le nouveau profil de déploiement

1.  **Créez le dossier du profil** : Copiez `deploy/default` et renommez-le, par exemple en `deploy/ma_super_machine`.
2.  **Extraire et placer les fichiers de configuration** depuis l'archive `.zip` :

    -   **Node-RED** :
        -   Dans l'archive, allez dans le dossier `nodered/`.
        -   Copiez tout son contenu (en particulier `flows.json`, `flows_cred.json`, et `package.json`) dans `deploy/ma_super_machine/nodered/`.

    -   **PostgreSQL (Schéma)** :
        -   Connectez-vous à votre PC modèle avec pgAdmin.
        -   Pour chaque base de données, faites un clic droit -> "Backup...". Choisissez le format "Plain" et dans les options, sélectionnez "Only schema".
        -   Cela génère un fichier `.sql` contenant les instructions `CREATE TABLE ...`. Si vous avez plusieurs bases de données avec des schémas différents, vous devrez consolider manuellement toutes les instructions `CREATE TABLE` dans un seul et unique fichier.
        -   **Important :** Nommez ce fichier `schema.sql` et placez-le dans `deploy/ma_super_machine/postgres/`. Le playbook est configuré pour chercher **exclusivement** ce nom de fichier.

    -   **Grafana (Dashboards & Datasources)** :
        -   **Dashboards** :
            -   Dans l'interface Grafana du PC modèle, ouvrez un dashboard, allez dans les paramètres (roue crantée) > "JSON Model".
            -   Copiez le contenu JSON et collez-le dans un nouveau fichier (ex: `mon_dashboard.json`).
            -   Placez ce fichier dans `deploy/ma_super_machine/grafana/provisioning/dashboards/`.
        -   **Datasources** :
            -   Créez un fichier `datasources.yml` dans `deploy/ma_super_machine/grafana/provisioning/datasources/`.
            -   Remplissez-le comme suit pour que Grafana se connecte automatiquement au conteneur PostgreSQL :
            ```yaml
            # deploy/ma_super_machine/grafana/provisioning/datasources/datasources.yml
            apiVersion: 1
            datasources:
              - name: 'PostgreSQL-Auto' # Nom qui apparaîtra dans Grafana
                type: postgres
                access: proxy
                url: postgres:5432 # 'postgres' est le nom du service interne
                user: postgres
                password: 'hai1@' # Doit correspondre au mot de passe dans deploy_playbook.yml
                database: 'GNSA-OEE-XXXXX' # La base de données par défaut à interroger
                jsonData:
                  sslmode: 'disable'
                  postgresVersion: 1200 # Spécifiez la version de PostgreSQL
            ```

### Étape 4 : Automatiser la création des bases de données

Pour que le déploiement crée automatiquement vos bases de données et y applique votre schéma :

1.  Ouvrez le playbook `deploy_playbook.yml`.
2.  Localisez la variable `postgres_databases`.
3.  Listez ici les noms de toutes les bases de données que vous souhaitez créer.
4.  Lors du déploiement, le playbook va d'abord créer chacune de ces bases, puis il exécutera le contenu du fichier `schema.sql` (que vous avez préparé à l'étape 3) **sur chacune d'entre elles**.

    ```yaml
    # deploy_playbook.yml
    ...
    vars:
      ...
      postgres_databases:
        - "GNSA-OEE-XXXXX"
        - "une_autre_base"
    ...
    ```

### Étape 5 : Déployer !

Votre profil est prêt. Il vous suffit maintenant d'associer ce profil à un nouveau PC dans le fichier `inventory.ini` et de lancer le déploiement.

```ini
# inventory.ini
[industrial_pcs]
192.168.1.20 ansible_user=admin ... deployment_profile=ma_super_machine
```

```bash
ANSIBLE_CONFIG=./ansible.cfg ansible-playbook -i inventory.ini deploy_playbook.yml --limit 192.168.1.20
```

Ansible va maintenant installer et configurer toutes les applications, créer les bases de données, importer leur schéma, et configurer Grafana avec ses dashboards et sa connexion à la base de données, le tout en une seule fois.

## Structure du Projet (Pour les curieux)

-   **Playbooks (`.yml`)**: Fichiers principaux décrivant les actions (`deploy`, `backup`, `restore`).
-   **`inventory.ini`**: Liste des PC à gérer.
-   **`ansible.cfg`**: Fichier de configuration technique pour Ansible.
-   **`deploy/`**: Contient les dossiers de configuration (profils) à déployer.
-   **`roles/`**: Découpage logique des tâches (copie de fichiers, gestion des services, etc.).
-   **`backups/`**: Répertoire où sont stockées les sauvegardes téléchargées.


### Profils Personnalisés

Vous pouvez créer des configurations sur mesure pour des machines spécifiques.

1.  **Créer un nouveau dossier de profil** : Dupliquez le dossier `deploy/default/` et renommez-le (par exemple, `deploy/machine_speciale/`).

2.  **Personnaliser la configuration** : Modifiez les fichiers à l'intérieur de votre nouveau dossier (`deploy/machine_speciale/`) selon vos besoins. Vous pouvez par exemple changer les flux Node-RED (`nodered/flows.json`) ou les dashboards Grafana.

3.  **Assigner le profil dans l'inventaire** : Dans le fichier `inventory.ini`, ajoutez la variable `deployment_profile` à la ligne de la machine concernée pour lui assigner votre nouveau profil.

**Exemple dans `inventory.ini`:**
```ini
# Cette machine utilisera la configuration du dossier deploy/machine_speciale/
192.168.1.17 ansible_user=admin ... deployment_profile=machine_speciale
```

Lors de l'exécution du `deploy_playbook.yml`, Ansible sélectionnera automatiquement le bon dossier de configuration en fonction du profil assigné à chaque machine.