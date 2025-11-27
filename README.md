# Projet d'Automatisation Ansible pour hai-pxxx-ai

Ce projet contient un ensemble de playbooks Ansible conçus pour déployer, gérer, sauvegarder et restaurer l'environnement applicatif `hai-pxxx-ai`, qui s'appuie sur des conteneurs Podman.

## Prérequis

- **Ansible** installé sur la machine de contrôle.
- **Podman** et **systemd** installés et configurés sur les machines cibles (`industrial_pcs`).
- Accès SSH avec élévation de privilèges (become) configuré entre la machine de contrôle et les cibles.

## Structure du Projet

- `inventory.ini`: Fichier d'inventaire définissant les hôtes cibles et les variables associées (comme `deployment_profile`).
- `deploy_playbook.yml`: Playbook principal pour le déploiement des applications.
- `backup_playbook.yml`: Playbook pour effectuer des sauvegardes complètes des données.
- `restore_playbook.yml`: Playbook pour restaurer les données à partir d'une sauvegarde.
- `roles/`: Contient les rôles Ansible pour structurer la logique de déploiement.
- `deploy/`: Contient les profils de déploiement. Chaque sous-dossier (ex: `default/`) contient les fichiers de configuration spécifiques à ce profil.

---

## 1. Playbook de Déploiement (`deploy_playbook.yml`)

Ce playbook installe et configure les services (Node-RED, Grafana, Ignition, PostgreSQL) en tant que services systemd gérant des conteneurs Podman.

### Utilisation

Pour lancer un déploiement, exécutez la commande suivante :

```bash
ansible-playbook -i inventory.ini deploy_playbook.yml
```

### Logique des Profils de Déploiement

Le déploiement est basé sur un système de profils. Vous pouvez définir un profil pour un hôte dans le fichier `inventory.ini` en utilisant la variable `deployment_profile`.

```ini
[industrial_pcs]
192.168.1.100 deployment_profile=usine_A
192.168.1.101 deployment_profile=usine_B
```

Si aucun profil n'est spécifié, le profil `default` est utilisé. Le playbook ira chercher les fichiers de configuration dans le dossier correspondant (ex: `deploy/usine_A/`).

### Gestion des Données des Volumes

Le playbook de déploiement offre des options pour nettoyer les volumes de données de certains services avant l'exécution.

#### Données Grafana

Par défaut, le volume de données de Grafana (`v_grafana`) est **nettoyé** à chaque déploiement. Cette approche garantit un déploiement "propre" où l'état de Grafana est entièrement défini par les fichiers de provisioning.

Pour conserver les données existantes (tableaux de bord, etc.), désactivez ce comportement avec l'option `clear_grafana_volume` :

```bash
ansible-playbook -i inventory.ini deploy_playbook.yml -e "clear_grafana_volume=false"
```

#### Données PostgreSQL

De la même manière, le volume de données de PostgreSQL (`v_postgres`) est nettoyé par défaut à chaque déploiement. Cela signifie que toutes les données des bases seront perdues.

Pour conserver les données de la base de données lors d'un redéploiement, désactivez ce comportement avec l'option `clear_postgres_volume` : 

```bash
ansible-playbook -i inventory.ini deploy_playbook.yml -e "clear_postgres_volume=false"
```

---

## 2. Playbook de Sauvegarde (`backup_playbook.yml`)

Ce playbook arrête les services, sauvegarde les volumes de données des conteneurs et les schémas de base de données, puis redémarre les services.

### Utilisation

```bash
ansible-playbook -i inventory.ini backup_playbook.yml
```

Les sauvegardes sont stockées localement dans le dossier `backups/`.

### Options

- **Sauvegarde complète du volume PostgreSQL** : Pour sauvegarder le volume complet de PostgreSQL au lieu d'un simple export des schémas, utilisez l'option `-e "backup_postgres_volume=true"`.

---

## 3. Playbook de Restauration (`restore_playbook.yml`)

Ce playbook restaure l'état des applications à partir d'un fichier de sauvegarde.

**AVERTISSEMENT :** La restauration écrase les données existantes sur la machine cible.

### Utilisation

Vous devez spécifier le fichier de sauvegarde à utiliser et limiter l'exécution à une seule machine cible, qui doit correspondre à celle de la sauvegarde.

```bash
ansible-playbook -i inventory.ini restore_playbook.yml \
  --limit 192.168.1.100 \
  -e "backup_file=backup-192.168.1.100-20251127T123000.zip"
```

- `backup_file`: Le nom du fichier de sauvegarde (situé dans le dossier `backups/`).
- `--limit`: Essentiel pour s'assurer que la restauration ne s'applique qu'à une seule machine. Le playbook échouera si l'IP de la limite ne correspond pas à l'IP dans le nom du fichier de sauvegarde.