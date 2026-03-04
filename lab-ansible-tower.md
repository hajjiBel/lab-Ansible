# Lab Pratique : Déploiement et Gestion avec Ansible Tower (AWX)

## 📋 Table des matières

1. [Introduction et Objectifs](#introduction-et-objectifs)
2. [Prérequis](#prérequis)
3. [Architecture du Lab](#architecture-du-lab)
4. [Module 1 : Comprendre Ansible Tower](#module-1--comprendre-ansible-tower)
5. [Module 2 : Installation et Déploiement de Tower](#module-2--installation-et-déploiement-de-tower)
6. [Module 3 : Configuration des composants Tower](#module-3--configuration-des-composants-tower)
7. [Module 4 : Déploiement d'une application avec Tower](#module-4--déploiement-dune-application-avec-tower)
8. [Exercices pratiques](#exercices-pratiques)
9. [Troubleshooting](#troubleshooting)
10. [Ressources supplémentaires](#ressources-supplémentaires)

---

## Introduction et Objectifs

### 🎯 Objectifs pédagogiques

À l'issue de ce lab, vous serez capable de :

- ✅ Comprendre les avantages d'Ansible Tower par rapport à Ansible CLI
- ✅ Déployer une instance Ansible Tower (AWX) avec Docker Compose
- ✅ Configurer les composants essentiels (Projects, Inventory, Credentials)
- ✅ Créer et exécuter des Job Templates
- ✅ Gérer la traçabilité et la sécurité des déploiements Ansible
- ✅ Planifier et automatiser l'exécution de playbooks



### 🎓 Niveau

Intermédiaire à Avancé (connaissance préalable d'Ansible requise)

---

## Prérequis

### Connaissances requises

- Maîtrise des concepts Ansible (playbooks, rôles, inventaires, commandes ad-hoc)
- Compréhension de base de Docker et Docker Compose
- Notions de Git et GitHub
- Administration Linux de base

### Environnement technique

#### Configuration minimale recommandée

| Composant | Spécification minimale |
|-----------|------------------------|
| CPU | 4 cores |
| RAM | 8 GB |
| Disque | 20 GB disponibles |
| OS | Ubuntu 20.04+ / CentOS 8+ / Debian 11+ |
| Docker | Version 20.10+ |
| Docker Compose | Version 1.29+ |

#### Logiciels nécessaires

```bash
# Vérifier les versions installées
docker --version
docker-compose --version
git --version
```

#### Accès réseau requis

- Connexion Internet pour télécharger les images Docker
- Accès à GitHub pour cloner les repositories
- Ports à ouvrir : **80** (interface Tower), **443** (optionnel HTTPS)

---

## Architecture du Lab

### Schéma d'infrastructure

```
┌─────────────────────────────────────────────────────┐
│                    GitHub Repository                 │
│              (Playbooks + Inventory)                 │
└────────────────────┬────────────────────────────────┘
                     │
                     │ git clone / sync
                     ▼
┌─────────────────────────────────────────────────────┐
│              Ansible Tower (AWX)                     │
│  ┌─────────────────────────────────────────────┐   │
│  │  Container 1: AWX Web (Interface Web)       │   │
│  │  Container 2: AWX Task (Exécution)          │   │
│  │  Container 3: PostgreSQL (Base de données)  │   │
│  │  Container 4: Redis (Cache/Queue)           │   │
│  │  Container 5: Receptor (Communication)      │   │
│  └─────────────────────────────────────────────┘   │
│                    Port 80                           │
└────────────────────┬────────────────────────────────┘
                     │
                     │ SSH / Ansible Execution
                     ▼
┌─────────────────────────────────────────────────────┐
│              Machine Cliente (Target)                │
│         (Application déployée via Tower)             │
└─────────────────────────────────────────────────────┘
```

### Composants

1. **Serveur Tower** : Héberge les 5 conteneurs Docker nécessaires au fonctionnement d'AWX
2. **Machine Cliente** : Serveur cible sur lequel les playbooks seront exécutés
3. **Repository GitHub** : Contient les playbooks, inventaires et variables

---

## Module 1 : Comprendre Ansible Tower

### 1.1 Problématiques résolues par Tower

#### Sans Tower (Ansible CLI)

**Limitations identifiées :**

```
❌ Pas de traçabilité : Qui a exécuté quel playbook ? Quand ?
❌ Pas de planification : Impossible de scheduler des exécutions
❌ Pas de gestion des droits : Tout le monde peut tout faire
❌ Pas de centralisation : Chacun travaille sur sa machine
❌ Pas de validation : Risque d'erreurs et de dérives
❌ Pas de pipeline : Impossible de chaîner des étapes
```

#### Avec Tower (Solution centralisée)

**Avantages apportés :**

```
✅ Traçabilité complète : Historique de toutes les exécutions
✅ Planification (scheduling) : Exécutions automatiques programmées
✅ Gestion des accès (RBAC) : Contrôle fin des permissions
✅ Centralisation : Interface web unique pour toute l'équipe
✅ Validation et workflows : Chaînage de jobs et étapes de validation
✅ Analyse du code : Tower comprend et organise vos playbooks
```

### 1.2 Comparaison avec d'autres outils

| Outil | Avantages | Limites pour Ansible |
|-------|-----------|---------------------|
| **Jenkins** | CI/CD généraliste, flexible | Ne comprend pas nativement Ansible |
| **GitLab CI** | Intégration Git native | Pas d'analyse spécifique Ansible |
| **Ansible Tower** | **Conçu pour Ansible**, analyse du code, lecture d'inventaire | Dédié exclusivement à Ansible |

### 1.3 Fonctionnalités clés de Tower

1. **Dashboard centralisé** : Vue d'ensemble des jobs, hosts, projets
2. **Gestion d'inventaire dynamique** : Support Cloud (AWS, Azure, GCP)
3. **Credential Management** : Stockage sécurisé des mots de passe et clés
4. **Job Templates** : Modèles réutilisables pour vos playbooks
5. **Workflows** : Chaînage de plusieurs jobs
6. **Notifications** : Intégration Slack, email, webhooks
7. **API REST** : Automatisation complète via API
8. **RBAC** : Role-Based Access Control
9. **Intégration LDAP/AD** : Authentification centralisée

### 1.4 Versions disponibles

| Version | Description | Licence | Support |
|---------|-------------|---------|---------|
| **AWX** | Version communautaire open-source | Gratuite | Communauté |
| **Ansible Tower** | Version Enterprise Red Hat | Payante | Red Hat Support |

> 💡 **Note** : Dans ce lab, nous utilisons AWX (version communautaire)

---

## Module 2 : Installation et Déploiement de Tower

### 2.1 Préparation de l'environnement



**Option C : Cloud (AWS/Azure/GCP)**

```bash
# Tower Server : t3.large (AWS) ou Standard_D4s_v3 (Azure)
# Client : t3.small (AWS) ou Standard_B2s (Azure)
```

#### Étape 2 : Préparation du serveur Tower

Connectez-vous au serveur Tower et installez les prérequis :

```bash
# Mise à jour du système
sudo apt update && sudo apt upgrade -y

# Installation de Docker (si non présent)
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Installation de Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/download/v2.20.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Vérification
docker --version
docker-compose --version
```

### 2.2 Déploiement de Tower

#### Étape 1 : Récupération du projet

```bash
# Cloner le repository contenant les scripts d'installation
git clone https://github.com/diranetafen/cursus-devops.git

# Accéder au dossier Tower
cd cursus-devops/ansible/tower/
```

#### Étape 2 : Extraction et préparation

```bash
# Décompresser l'archive Tower
tar -xzf awx.tar.gz -C ~/

# Accéder au répertoire d'installation
cd ~/awx/

# Vérifier le contenu
ls -la
# Vous devriez voir : docker-compose.yml et les fichiers de configuration
```

#### Étape 3 : Analyse du docker-compose.yml

```yaml
# Structure des conteneurs AWX (aperçu)
services:
  awx_web:        # Interface web
  awx_task:       # Exécution des jobs
  postgres:       # Base de données
  redis:          # Cache et file d'attente
  receptor:       # Communication entre nodes
```

#### Étape 4 : Lancement de Tower

```bash
# Démarrer tous les conteneurs en arrière-plan
docker-compose up -d

# Suivre les logs en temps réel (optionnel)
docker-compose logs -f
```

**⏳ Temps de démarrage estimé : 3-5 minutes**

#### Étape 5 : Vérification du déploiement

```bash
# Vérifier que tous les conteneurs sont actifs
docker ps

# Vous devriez voir 5 conteneurs en état "Up"
# CONTAINER ID   IMAGE                COMMAND                  STATUS
# xxxxxxxxxxxx   awx_web:latest       "/usr/bin/supervisord"   Up 2 minutes
# xxxxxxxxxxxx   awx_task:latest      "/usr/bin/supervisord"   Up 2 minutes
# xxxxxxxxxxxx   postgres:12          "docker-entrypoint..."   Up 2 minutes
# xxxxxxxxxxxx   redis:latest         "docker-entrypoint..."   Up 2 minutes
# xxxxxxxxxxxx   receptor:latest      "/usr/bin/receptor..."   Up 2 minutes
```

### 2.3 Accès à l'interface Tower

#### Étape 1 : Ouverture du port

**Si vous utilisez un firewall :**

```bash
# UFW (Ubuntu/Debian)
sudo ufw allow 80/tcp

# Firewalld (CentOS/RHEL)
sudo firewall-cmd --permanent --add-port=80/tcp
sudo firewall-cmd --reload
```

#### Étape 2 : Connexion à Tower

1. Ouvrez votre navigateur
2. Accédez à : `http://<IP_SERVEUR_TOWER>`
3. Utilisez les identifiants par défaut :
   - **Username** : `admin`
   - **Password** : `password`

#### Étape 3 : Découverte de l'interface

**Composants du Dashboard :**

```
┌─────────────────────────────────────────────────┐
│  🏠 Dashboard                                    │
├─────────────────────────────────────────────────┤
│  📊 Jobs (historique des exécutions)            │
│  🖥️  Hosts (machines gérées)                    │
│  📂 Projects (liens vers les repos Git)         │
│  📋 Inventories (liste des hôtes)               │
│  📝 Templates (modèles de jobs)                 │
│  🔑 Credentials (identifiants sécurisés)        │
│  👥 Organizations & Users (gestion des accès)   │
│  ⚙️  Settings (configuration globale)           │
└─────────────────────────────────────────────────┘
```

### 2.4 Configuration de la machine cliente

#### Sur la machine cliente

```bash
# Installer Python (requis par Ansible)
sudo apt update
sudo apt install -y python3 python3-pip

# Installer Docker (pour le déploiement d'application)
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Configurer l'accès SSH (si nécessaire)
sudo systemctl enable ssh
sudo systemctl start ssh
```

#### Sur le serveur Tower

```bash
# Générer une paire de clés SSH (si non existante)
ssh-keygen -t rsa -b 4096 -C "tower@ansible.local"

# Copier la clé publique vers la machine cliente
ssh-copy-id user@<IP_CLIENT>

# Tester la connexion
ssh user@<IP_CLIENT>
```

---

## Module 3 : Configuration des composants Tower

### 3.1 Vue d'ensemble du workflow

**Les 4 étapes pour créer un Job :**

```
1️⃣ PROJECT       → Lien avec le code source (GitHub)
2️⃣ INVENTORY     → Liste des hôtes cibles
3️⃣ CREDENTIALS   → Identifiants SSH, Sudo, Vault
4️⃣ JOB TEMPLATE  → Configuration d'exécution
```

### 3.2 Création d'un Project

#### Définition

Un **Project** établit la connexion entre Tower et votre repository Git contenant vos playbooks.

#### Étape 1 : Accès au menu Projects

1. Dans Tower, cliquez sur **Projects** (menu latéral gauche)
2. Cliquez sur le bouton **+** (Add)

#### Étape 2 : Configuration du Project

| Champ | Valeur | Description |
|-------|--------|-------------|
| **Name** | `deploy-web-app-project` | Nom descriptif du projet |
| **Organization** | `Default` | Organisation par défaut |
| **SCM Type** | `Git` | Type de gestionnaire de code source |
| **SCM URL** | `https://github.com/diranetafen/cursus-devops.git` | URL du repository |
| **SCM Branch/Tag/Commit** | `master` | Branche à utiliser |

#### Options de mise à jour du SCM

- ☑️ **Clean** : Supprime l'ancienne version avant de récupérer
- ☑️ **Delete on Update** : Supprime les modifications locales
- ☑️ **Update Revision on Launch** : Met à jour le code à chaque lancement

#### Étape 3 : Enregistrement et synchronisation

1. Cliquez sur **Save**
2. Tower lance automatiquement une synchronisation
3. Attendez que le statut passe à **Successful** (icône verte)

#### Vérification

```bash
# Tower exécute en arrière-plan l'équivalent de :
git clone https://github.com/diranetafen/cursus-devops.git
git checkout master
```

**📌 Note importante** : Tower utilise un playbook Ansible pour récupérer votre code Git !

### 3.3 Création d'un Inventory

#### Définition

Un **Inventory** définit la liste des machines cibles sur lesquelles les playbooks seront exécutés.

#### Étape 1 : Création de l'inventaire principal

1. Cliquez sur **Inventories** → **+** (Add) → **Inventory**
2. Remplissez :
   - **Name** : `prod-inventory`
   - **Organization** : `Default`
3. Cliquez sur **Save**

#### Étape 2 : Ajout d'une source d'inventaire

1. Dans l'inventaire créé, allez dans l'onglet **Sources**
2. Cliquez sur **+** (Add Source)
3. Configurez :

| Champ | Valeur | Description |
|-------|--------|-------------|
| **Name** | `client-source` | Nom de la source |
| **Source** | `Sourced from a Project` | Provenance |
| **Project** | `deploy-web-app-project` | Projet créé précédemment |
| **Inventory File** | `ansible/tower/host.yml` | Chemin du fichier d'inventaire |

#### Étape 3 : Synchronisation de l'inventaire

1. Cliquez sur **Save**
2. Cliquez sur l'icône de synchronisation (🔄)
3. Attendez la fin de la synchronisation (statut **Successful**)

#### Étape 4 : Vérification des hôtes

1. Retournez dans l'inventaire `prod-inventory`
2. Allez dans l'onglet **Hosts**
3. Vous devriez voir votre machine cliente avec son groupe

**Exemple de fichier host.yml (référence) :**

```yaml
all:
  children:
    prod:
      hosts:
        client:
          ansible_host: 192.168.1.100
          ansible_user: ubuntu
          ansible_python_interpreter: /usr/bin/python3
```

### 3.4 Création des Credentials

#### Types de credentials nécessaires

1. **Machine Credential** : Connexion SSH et élévation sudo
2. **Vault Credential** : Déchiffrement des secrets Ansible Vault

#### 3.4.1 Credential Machine (SSH + Sudo)

1. Allez dans **Credentials** → **+** (Add)
2. Configurez :

| Champ | Valeur |
|-------|--------|
| **Name** | `ssh-sudo-credential` |
| **Credential Type** | `Machine` |
| **Username** | `ubuntu` (ou votre user SSH) |
| **SSH Private Key** | Collez votre clé privée SSH |
| **Privilege Escalation Method** | `sudo` |
| **Privilege Escalation Username** | `root` |
| **Privilege Escalation Password** | Votre mot de passe sudo |

3. Cliquez sur **Save**

#### 3.4.2 Credential Vault

1. Cliquez à nouveau sur **+** (Add)
2. Configurez :

| Champ | Valeur |
|-------|--------|
| **Name** | `vault-password-credential` |
| **Credential Type** | `Vault` |
| **Vault Password** | `devops` (selon le repo) |

3. Cliquez sur **Save**

**📌 Pourquoi deux credentials ?**

```
SSH Credential    → Connexion à la machine cliente
Vault Credential  → Déchiffrer les variables sensibles dans les playbooks
```

### 3.5 Récapitulatif de configuration

**✅ Checklist avant de passer au Module 4 :**

- [ ] Project créé et synchronisé (statut vert)
- [ ] Inventory créé avec source synchronisée
- [ ] Hôtes visibles dans l'inventaire
- [ ] Credential SSH/Sudo créé
- [ ] Credential Vault créé

---

## Module 4 : Déploiement d'une application avec Tower

### 4.1 Création d'un Job Template

#### Définition

Un **Job Template** est un modèle d'exécution qui combine Project, Inventory, Credentials et Playbook.

#### Étape 1 : Accès aux Templates

1. Cliquez sur **Templates** → **+** (Add) → **Job Template**

> ⚠️ **Attention** : Ne pas confondre avec **Workflow Template** (chaînage de jobs)

#### Étape 2 : Configuration du Job Template

| Champ | Valeur | Description |
|-------|--------|-------------|
| **Name** | `deploy-web-app` | Nom descriptif du job |
| **Job Type** | `Run` | Exécution réelle (vs Check) |
| **Inventory** | `prod-inventory` | Inventaire cible |
| **Project** | `deploy-web-app-project` | Projet contenant le code |
| **Playbook** | `ansible/tower/deploy.yml` | Playbook à exécuter |
| **Credentials** | `ssh-sudo-credential` + `vault-password-credential` | Les deux credentials |

#### Options avancées

**Privilege Escalation :**
- ☑️ **Enable Privilege Escalation** : Permet l'utilisation de sudo

**Verbosity :**
- 🔘 **0 (Normal)** : Sortie standard
- 🔘 **1 (Verbose)** : -v
- 🔘 **2 (More Verbose)** : -vv
- 🔘 **3 (Debug)** : -vvv

**Options supplémentaires disponibles :**
- **Limit** : Restreindre l'exécution à certains hôtes
- **Job Tags** : Exécuter uniquement certaines tâches
- **Skip Tags** : Ignorer certaines tâches
- **Extra Variables** : Surcharger les variables au dernier moment

#### Étape 3 : Enregistrement

1. Vérifiez tous les champs
2. Cliquez sur **Save**

### 4.2 Exécution du Job

#### Lancement manuel

1. Sur la page du template `deploy-web-app`
2. Cliquez sur l'icône **🚀 Launch** (fusée)
3. Tower ouvre la console d'exécution en temps réel

#### Suivi de l'exécution

**Vue de la console :**

```
PLAY [Deploy Web Application] **************************************************

TASK [Gathering Facts] *********************************************************
ok: [client]

TASK [Install Docker] **********************************************************
changed: [client]

TASK [Install Python pip] ******************************************************
changed: [client]

TASK [Deploy Apache container] *************************************************
changed: [client]

PLAY RECAP *********************************************************************
client : ok=4  changed=3  unreachable=0  failed=0  skipped=0  rescued=0  ignored=0
```

#### Informations affichées

- **Output** : Sortie console en temps réel
- **Status** : Successful / Failed / Running
- **Revision** : Commit Git utilisé
- **Launched By** : Utilisateur ayant lancé le job
- **Started** : Date/heure de début
- **Finished** : Date/heure de fin
- **Elapsed** : Durée d'exécution

### 4.3 Vérification du déploiement

#### Test de l'application

1. Ouvrez votre navigateur
2. Accédez à : `http://<IP_CLIENT>:80`
3. Vous devriez voir : **"It's working!"**

#### Vérification sur le client

```bash
# Connexion au client
ssh ubuntu@<IP_CLIENT>

# Vérifier les conteneurs Docker
docker ps

# Devrait afficher le conteneur Apache
# CONTAINER ID   IMAGE          COMMAND              STATUS         PORTS
# xxxxxxxxxxxx   httpd:latest   "httpd-foreground"   Up 2 minutes   0.0.0.0:80->80/tcp
```

### 4.4 Analyse du Dashboard

Retournez sur le Dashboard Tower :

```
📊 Jobs History
├─ ✅ deploy-web-app (Successful) - 2 min ago
├─ ✅ prod-inventory - Sync (Successful) - 15 min ago
└─ ✅ deploy-web-app-project - Update (Successful) - 20 min ago

🖥️ Hosts
└─ client (Active)

📂 Projects
└─ deploy-web-app-project (Successful, Last synced: 20 min ago)
```

**Indicateurs clés :**
- **Successful Jobs** : Nombre de jobs réussis
- **Failed Jobs** : Nombre d'échecs
- **Hosts** : Nombre de machines gérées
- **Failed Hosts** : Machines inaccessibles

### 4.5 Fonctionnalités avancées

#### 4.5.1 Workflows

Les **Workflows** permettent de chaîner plusieurs jobs.

**Exemple de workflow :**

```
Job 1: Backup Database
      ↓ (on success)
Job 2: Deploy Application
      ↓ (on success)
Job 3: Run Tests
      ↓ (on failure)
Job 4: Rollback
```

**Création d'un workflow :**

1. **Templates** → **+** → **Workflow Template**
2. Configurez le nom et l'inventaire
3. Dans le **Workflow Visualizer**, ajoutez des nodes
4. Définissez les conditions (success, failure, always)

#### 4.5.2 Planification (Scheduling)

**Planifier un job pour exécution automatique :**

1. Ouvrez le job template
2. Allez dans l'onglet **Schedules**
3. Cliquez sur **+** (Add)
4. Configurez :
   - **Name** : `daily-backup`
   - **Start Date/Time** : Date et heure
   - **Repeat Frequency** : Daily, Weekly, Monthly
   - **Timezone** : Votre fuseau horaire

**Exemple** : Backup tous les jours à 2h du matin

#### 4.5.3 Notifications

**Configurer des notifications Slack/Email :**

1. **Settings** → **Notifications**
2. Cliquez sur **+** (Add)
3. Choisissez le type : Slack, Email, Webhook
4. Configurez les paramètres (URL webhook, email, etc.)
5. Associez aux templates concernés

#### 4.5.4 Surveys

Les **Surveys** permettent de demander des variables à l'utilisateur au lancement.

**Exemple** : Demander la version de l'application à déployer

1. Ouvrez le job template
2. Allez dans l'onglet **Survey**
3. Cliquez sur **Add**
4. Définissez :
   - **Prompt** : "Version de l'application ?"
   - **Variable** : `app_version`
   - **Type** : Text / Multiple Choice
   - **Default** : `1.0.0`

### 4.6 Gestion des utilisateurs et des droits

#### Création d'un utilisateur

1. **Users** → **+** (Add)
2. Configurez :
   - **Username** : `developer1`
   - **Email** : `dev1@example.com`
   - **Password** : (généré)
   - **User Type** : Normal User

#### Attribution de permissions

**Exemple** : Permettre à `developer1` d'exécuter uniquement le job `deploy-web-app`

1. Allez dans le **Template** `deploy-web-app`
2. Onglet **Permissions** → **+** (Add)
3. Sélectionnez l'utilisateur `developer1`
4. Choisissez le rôle : **Execute** (uniquement lancer)

**Rôles disponibles :**
- **Admin** : Toutes les permissions
- **Execute** : Lancer le job uniquement
- **Read** : Voir le job (pas d'exécution)

---

## Exercices pratiques

### Exercice 1 : Déploiement d'une application Nginx

**Objectif** : Créer un nouveau job template pour déployer Nginx à la place d'Apache

**Instructions :**

1. Créez un nouveau playbook `deploy-nginx.yml` dans votre repo Git
2. Créez un nouveau Project dans Tower
3. Créez un Job Template utilisant ce nouveau playbook
4. Exécutez et vérifiez l'accès à Nginx

**Contenu du playbook (référence) :**

```yaml
---
- name: Deploy Nginx
  hosts: all
  become: yes
  tasks:
    - name: Install Docker
      apt:
        name: docker.io
        state: present
        update_cache: yes

    - name: Deploy Nginx container
      docker_container:
        name: nginx-app
        image: nginx:latest
        state: started
        ports:
          - "8080:80"
```

**Critères de réussite :**
- ✅ Job Template créé et configuré
- ✅ Exécution réussie
- ✅ Nginx accessible sur `http://<IP_CLIENT>:8080`

---

### Exercice 2 : Workflow avec backup et déploiement

**Objectif** : Créer un workflow qui sauvegarde puis déploie

**Instructions :**

1. Créez un playbook `backup.yml` qui sauvegarde `/etc/` dans `/tmp/backup/`
2. Créez un Job Template pour ce playbook
3. Créez un Workflow Template avec :
   - **Node 1** : Job backup
   - **Node 2** : Job deploy-web-app (si backup réussi)
4. Testez le workflow

**Critères de réussite :**
- ✅ Workflow créé avec 2 nodes
- ✅ Node 2 exécuté uniquement si Node 1 réussit
- ✅ Backup créé dans `/tmp/backup/`

---

### Exercice 3 : Survey pour choisir l'environnement

**Objectif** : Permettre à l'utilisateur de choisir l'environnement (dev/prod)

**Instructions :**

1. Ajoutez un Survey au Job Template `deploy-web-app`
2. Créez une variable `environment` avec choix : `dev` ou `prod`
3. Modifiez le playbook pour utiliser cette variable (ex: port différent)
4. Testez en lançant le job

**Exemple de modification playbook :**

```yaml
- name: Deploy on correct port
  docker_container:
    name: web-app
    image: httpd:latest
    ports:
      - "{{ '8080:80' if environment == 'dev' else '80:80' }}"
```

**Critères de réussite :**
- ✅ Survey configuré
- ✅ Choix de l'environnement proposé au lancement
- ✅ Application déployée sur le bon port

---

### Exercice 4 : Notification Slack

**Objectif** : Envoyer une notification Slack à chaque exécution de job

**Instructions :**

1. Créez un webhook Slack (https://api.slack.com/messaging/webhooks)
2. Configurez une Notification dans Tower avec ce webhook
3. Associez la notification au Job Template
4. Testez l'exécution et vérifiez la réception dans Slack

**Critères de réussite :**
- ✅ Notification créée dans Tower
- ✅ Associée au job template
- ✅ Message reçu dans Slack après exécution

---

### Exercice 5 : Gestion des droits utilisateurs

**Objectif** : Créer deux utilisateurs avec des permissions différentes

**Instructions :**

1. Créez deux utilisateurs :
   - `developer` : Peut uniquement exécuter les jobs
   - `admin-ops` : Peut créer et modifier les jobs
2. Testez en vous connectant avec chaque compte
3. Vérifiez les limitations

**Critères de réussite :**
- ✅ Utilisateur `developer` peut exécuter mais pas modifier
- ✅ Utilisateur `admin-ops` peut créer de nouveaux templates
- ✅ Permissions respectées

---

## Troubleshooting

### Problèmes courants et solutions

#### 1. Tower ne démarre pas

**Symptômes :**
```bash
docker ps
# Aucun conteneur ou conteneurs en état "Restarting"
```

**Solutions :**

```bash
# Vérifier les logs
docker-compose logs

# Problème courant : manque de ressources
# Solution : augmenter RAM ou arrêter d'autres services

# Recréer les conteneurs
docker-compose down
docker-compose up -d
```

#### 2. Job échoue avec "Unreachable Host"

**Symptômes :**
```
fatal: [client]: UNREACHABLE! => {"changed": false, "msg": "Failed to connect"}
```

**Solutions :**

```bash
# Tester la connectivité SSH manuellement
ssh -i /path/to/key user@<IP_CLIENT>

# Vérifier les credentials dans Tower
# Credentials → ssh-sudo-credential → Vérifier la clé SSH

# Vérifier le firewall du client
sudo ufw status
sudo ufw allow 22/tcp
```

#### 3. Synchronisation Project échoue

**Symptômes :**
```
ERROR! Repository not found or access denied
```

**Solutions :**

1. Vérifiez l'URL du repository (pas d'erreur de frappe)
2. Si repo privé, ajoutez un Credential de type **Source Control**
3. Vérifiez la branche spécifiée (existe-t-elle ?)

```bash
# Tester manuellement
git clone <URL_REPO>
git checkout <BRANCHE>
```

#### 4. Problème de mot de passe Vault

**Symptômes :**
```
ERROR! Vault password incorrect
```

**Solutions :**

1. Vérifiez le mot de passe dans le Credential Vault
2. Testez manuellement :

```bash
ansible-vault view group_vars/vault.yml
# Entrez le mot de passe
```

#### 5. Interface Tower lente

**Symptômes :**
- Chargement très lent
- Timeouts

**Solutions :**

```bash
# Vérifier l'utilisation des ressources
docker stats

# Si conteneurs surchargés, augmenter RAM
# Ou diminuer le nombre de workers dans Tower settings
```

#### 6. Job bloqué en "Pending"

**Symptômes :**
- Job ne démarre jamais
- Statut "Pending" indéfiniment

**Solutions :**

```bash
# Vérifier les conteneurs Tower
docker ps | grep awx

# Redémarrer le conteneur task
docker restart <CONTAINER_ID_AWX_TASK>

# Vérifier la capacité d'exécution
# Settings → Jobs → Vérifier la capacité disponible
```

---

## Ressources supplémentaires

### Documentation officielle

- **AWX Documentation** : https://ansible.readthedocs.io/projects/awx/
- **Ansible Tower Documentation** : https://docs.ansible.com/ansible-tower/
- **Ansible Documentation** : https://docs.ansible.com/

### Tutoriels et guides

- **AWX GitHub Repository** : https://github.com/ansible/awx
- **Ansible Galaxy** (Rôles communautaires) : https://galaxy.ansible.com/
- **Red Hat Learning** : https://www.redhat.com/en/services/training

### Communauté

- **Ansible Forum** : https://forum.ansible.com/
- **Reddit /r/ansible** : https://reddit.com/r/ansible
- **Stack Overflow** : Tag `ansible-tower` ou `ansible-awx`

### Outils complémentaires

- **Ansible Lint** : Validation de playbooks
  ```bash
  pip install ansible-lint
  ansible-lint playbook.yml
  ```

- **Ansible Semaphore** : Alternative UI à AWX
  - https://github.com/ansible-semaphore/semaphore

### Certifications

- **Red Hat Certified Specialist in Ansible Automation** (EX407)
- **Red Hat Certified Engineer in Ansible Automation** (EX294)

---

## Conclusion

### Ce que vous avez appris

✅ **Comprendre l'intérêt d'Ansible Tower** pour la centralisation et la traçabilité  
✅ **Déployer Tower (AWX)** avec Docker Compose  
✅ **Configurer les composants essentiels** : Projects, Inventory, Credentials, Job Templates  
✅ **Exécuter des playbooks** via l'interface web Tower  
✅ **Utiliser les fonctionnalités avancées** : Workflows, Scheduling, Notifications  
✅ **Gérer les utilisateurs et les permissions** (RBAC)  

### Prochaines étapes

1. **Intégrer Tower avec votre CI/CD** (Jenkins, GitLab CI)
2. **Utiliser l'API Tower** pour automatiser la création de jobs
3. **Configurer des inventaires dynamiques** pour le Cloud (AWS, Azure)
4. **Créer des workflows complexes** avec conditions et validations
5. **Mettre en place des notifications** Slack/Teams pour votre équipe
6. **Explorer Ansible Collections** pour étendre les fonctionnalités

### Commandes utiles de référence

```bash
# Gestion des conteneurs Tower
docker-compose up -d          # Démarrer Tower
docker-compose down           # Arrêter Tower
docker-compose logs -f        # Suivre les logs
docker-compose restart        # Redémarrer tous les conteneurs

# Vérification
docker ps                     # Lister les conteneurs actifs
docker stats                  # Utilisation des ressources

# Nettoyage
docker-compose down -v        # Arrêter et supprimer les volumes
docker system prune -a        # Nettoyer tout Docker
```

---

**🎓 Félicitations ! Vous maîtrisez maintenant les bases d'Ansible Tower/AWX.**

**📧 Support** : Pour toute question, n'hésitez pas à contacter votre formateur ou à consulter la documentation officielle.

**⭐ Bonne chance dans vos projets d'automatisation !**

---

## Annexes

### Annexe A : Exemple de playbook deploy.yml

```yaml
---
- name: Deploy Web Application
  hosts: all
  become: yes
  vars_files:
    - group_vars/vault.yml

  tasks:
    - name: Install Docker
      apt:
        name: docker.io
        state: present
        update_cache: yes

    - name: Install Python pip
      apt:
        name: python3-pip
        state: present

    - name: Install Docker Python module
      pip:
        name: docker
        state: present

    - name: Deploy Apache container
      docker_container:
        name: webapp
        image: httpd:latest
        state: started
        restart_policy: always
        ports:
          - "80:80"
```

### Annexe B : Exemple de fichier d'inventaire host.yml

```yaml
all:
  children:
    prod:
      hosts:
        client:
          ansible_host: 192.168.1.100
          ansible_user: ubuntu
          ansible_ssh_private_key_file: /path/to/key
          ansible_python_interpreter: /usr/bin/python3
    
    dev:
      hosts:
        dev-server:
          ansible_host: 192.168.1.101
          ansible_user: ubuntu
          ansible_python_interpreter: /usr/bin/python3
```

### Annexe C : Commandes Ansible de référence

```bash
# Test de connectivité
ansible all -i inventory.yml -m ping

# Exécution ad-hoc
ansible all -i inventory.yml -m shell -a "uptime"

# Exécution d'un playbook
ansible-playbook -i inventory.yml playbook.yml

# Mode check (dry-run)
ansible-playbook -i inventory.yml playbook.yml --check

# Avec verbosité
ansible-playbook -i inventory.yml playbook.yml -vvv

# Utiliser Vault
ansible-playbook -i inventory.yml playbook.yml --ask-vault-pass
```

---

**Document généré pour la formation Ansible Tower/AWX**  
**Version** : 1.0  
**Date** : Mars 2026  
**Auteur** : Formation DevOps