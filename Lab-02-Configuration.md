# Lab 2 - Configuration d'Ansible

## 🎯 Objectifs

À la fin de ce lab, vous serez capable de :
- Comprendre la structure de configuration d'Ansible
- Configurer le fichier `ansible.cfg`
- Gérer les inventaires complexes
- Optimiser les performances d'Ansible
- Configurer les variables globales

## 📋 Introduction

Ansible offre plusieurs niveaux de configuration pour personnaliser son comportement. La priorité suit cet ordre :
1. Variables d'environnement
2. Fichier `ansible.cfg` (répertoire courant)
3. Fichier `~/.ansible.cfg` (répertoire utilisateur)
4. Fichier `/etc/ansible/ans
ible.cfg` (système)

## 🏗️ Structure du Projet Ansible

```
ansible-project/
├── ansible.cfg              # Configuration Ansible
├── inventory/
│   ├── production.ini       # Inventaire production
│   ├── staging.ini          # Inventaire staging
│   └── development.ini      # Inventaire développement
├── group_vars/
│   ├── all.yml              # Variables pour tous les groupes
│   ├── webservers.yml       # Variables pour webservers
│   └── databases.yml        # Variables pour databases
├── host_vars/
│   ├── app1.yml             # Variables spécifiques à app1
│   └── db.yml               # Variables spécifiques à db
├── roles/
│   ├── webserver/
│   ├── database/
│   └── common/
├── playbooks/
│   ├── site.yml             # Playbook principal
│   ├── webservers.yml
│   └── databases.yml
└── templates/
    └── config.j2
```

## ⚙️ Configuration du Fichier ansible.cfg

### Exemple Complet Optimisé

Créer le fichier `ansible.cfg` dans le répertoire du projet :

```ini
[defaults]
# Inventaire par défaut
inventory = ./inventory/production.ini

# Utilisateur par défaut pour SSH
remote_user = vagrant

# Port SSH par défaut
remote_port = 22

# Clé SSH privée
private_key_file = ~/.ssh/id_rsa

# Désactiver la vérification de la clé d'hôte SSH
host_key_checking = False

# Nombre de forks (connexions parallèles)
# Augmenter pour de meilleures performances
forks = 5

# Répertoire temporaire sur les nœuds gérés
remote_tmp = /tmp/.ansible

# Archiver les logs
log_path = /var/log/ansible.log

# Emplacement du module Python
library = ./library

# Emplacement des rôles
roles_path = ./roles

# Timeout pour les connexions SSH (secondes)
timeout = 10

# Afficher les tâches en cours
display_skipped_hosts = False

# Format de sortie
force_color = True
force_handlers = False

# Limite les hôtes concernés (par défaut: tous)
# limit = webservers

# Emplacement du fichier de verrouillage
lock_path = /tmp/ansible.lock

# Stratégie pour les boucles
strategy = linear

# Activer les statistiques
force_valid_group_names = True

# Ignorer les erreurs de syntaxe de groupe
invalid_comment_lines = # ,

[inventory]
# Format d'inventaire
enable_plugins = ini, yaml, aws_ec2, azure_rm, google

# Traiter les symboles de remplacement comme des patterns
unparsed_is_failed = False

[privilege_escalation]
# Utiliser sudo pour passer en root
become = False
become_method = sudo
become_user = root
become_ask_pass = False
become_ask_vault_pass = False

[ssh_connection]
# Utiliser pipelining pour réduire les connexions SSH
pipelining = True

# Nombre de vérifications avant connexion
retries = 3

# Type de contrôle de flux
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
```

### Étapes de Configuration

```bash
# 1. Créer la structure du projet
mkdir -p ansible-project/{inventory,group_vars,host_vars,roles,playbooks,templates}
cd ansible-project

# 2. Créer ansible.cfg
cat > ansible.cfg << 'EOF'
[defaults]
inventory = ./inventory/production.ini
remote_user = vagrant
private_key_file = ~/.ssh/id_rsa
host_key_checking = False
forks = 5
force_color = True
log_path = /tmp/ansible.log

[ssh_connection]
pipelining = True
EOF

# 3. Vérifier la configuration
ansible-config view
ansible-config list
```

## 📑 Gestion des Inventaires

### Inventaire Simple (INI)

Créer `inventory/production.ini` :

```ini
# Groupes d'hôtes
[webservers]
app1  ansible_host=192.168.60.4
app2  ansible_host=192.168.60.5

[databases]
db    ansible_host=192.168.60.6

# Groupe parent (tous les serveurs)
[all_servers:children]
webservers
databases

# Variables globales
[all_servers:vars]
ansible_user=vagrant
ansible_port=22
ansible_connection=ssh
environment=production
```

### Inventaire YAML (Recommandé)

Créer `inventory/production.yaml` :

```yaml
---
all:
  children:
    webservers:
      hosts:
        app1:
          ansible_host: 192.168.60.4
          ansible_port: 22        
          
        app2:
          ansible_host: 192.168.60.5          
      vars:
        ansible_user: vagrant
        http_port: 80       
    
    databases:
      hosts:
        db:
          ansible_host: 192.168.60.6
      vars:
        ansible_user: vagrant
        db_port: 3306
       

  vars:
    ansible_connection: ssh
    ssh_port: 22
    environment: production
   
```

### Inventaire Dynamique

Pour les environnements cloud, créer `inventory/aws_ec2.yaml` :

```yaml
---
plugin: aws_ec2
aws_access_key: '{{ lookup("env", "AWS_ACCESS_KEY_ID") }}'
aws_secret_key: '{{ lookup("env", "AWS_SECRET_ACCESS_KEY") }}'
regions:
  - us-east-1
  - eu-west-1
keyed_groups:
  - key: placement.region
    parent_group: aws_region
  - key: tags.Name
    parent_group: application
hostnames:
  - dns-name
  - private-ip-address
compose:
  ansible_host: private_ip_address
```


## 🧪 Vérification de la Configuration

### Tester l'inventaire

```bash
# Afficher l'inventaire parsé
ansible-inventory -i inventory/production.ini --list

# Afficher les hôtes d'un groupe
ansible-inventory -i inventory/production.ini --host app1

# Afficher les groupes
ansible-inventory -i inventory/production.ini --graph

# Format JSON pour vérification
ansible-inventory -i inventory/production.ini --list --yaml
```

### Tester les connexions

```bash
# Tester ping sur tous les hôtes
ansible all -i inventory/production.ini -m ping

# Tester sur un groupe spécifique
ansible webservers -i inventory/production.ini -m ping

# Tester sur un hôte spécifique
ansible app1 -i inventory/production.ini -m ping -vvv
```

### Vérifier les variables

```bash
# Afficher les variables d'un hôte
ansible app1 -i inventory/production.ini -m setup

# Filtre les variables
ansible app1 -i inventory/production.ini -m setup -a "filter=ansible_*"


```

## 🔍 Commandes Utiles de Configuration

```bash
# Afficher la configuration actuelle utilisée
ansible-config view

# Lister toutes les options de configuration
ansible-config list

# Valider la syntaxe d'ansible.cfg
ansible-config validate

# Voir quel fichier ansible.cfg est utilisé
ansible-config list | grep CONFIG_FILE

# Voir le chemin de recherche des inventaires
ansible all -i inventory/ --list-hosts
```

## 🐛 Dépannage

### Erreur : "Unable to parse inventory file"

```bash
# Vérifier la syntaxe du fichier d'inventaire
ansible-inventory -i inventory/production.ini --list

# Avec verbosité
ansible-inventory -i inventory/production.ini --list -vvv
```

### Erreur de connexion SSH

```bash
# Tester SSH directement
ssh -vvv vagrant@192.168.60.4

# Vérifier les paramètres SSH dans ansible.cfg
grep -A 5 "\[ssh_connection\]" ansible.cfg

# Tester avec verbosité
ansible all -i inventory/ -m ping -vvv
```



## 📚 Ressources Additionnelles

- Documentation Ansible Configuration: https://docs.ansible.com/ansible/latest/reference_appendices/config.html
- Variables Guide: https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html
- Inventaire Advanced: https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html

---

**Étape suivante** → [Lab 3 - Commandes Ad-Hoc](./Lab-03-Commandes-Ad-Hoc.md)
