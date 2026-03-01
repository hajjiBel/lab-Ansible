# Lab 1 - Installation d'Ansible

## 🎯 Objectifs

À la fin de ce lab, vous serez capable de :
- Installer Ansible sur la machine de contrôle
- Configurer l'authentification SSH sans mot de passe
- Vérifier la connectivité avec les nœuds gérés
- Comprendre les prérequis Python

## 📋 Introduction

Ansible est un outil d'orchestration et d'automatisation décentralisé. Contrairement à d'autres outils, il n'a besoin d'aucun agent sur les nœuds gérés - il communique en SSH et Python.

### Architecture d'Ansible

```
┌──────────────────────────────────────────────────────────────┐
│              MACHINE DE CONTRÔLE (master)                    │
│  ┌────────────────────────────────────────────────────────┐  │
│  │         Ansible Engine                                 │  │
│  │  • CLI (ansible, ansible-playbook)                    │  │
│  │  • Modules                                             │  │
│  │  • Plugins                                             │  │
│  │  • Python 3.9+                                         │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
                              │
                    SSH + Python (pas d'agent)
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
   ┌────▼─────┐         ┌─────▼────┐         ┌─────▼────┐
   │   app1    │         │   app2    │         │   db     │
   │ 192.168   │         │ 192.168   │         │ 192.168  │
   │ .60.4     │         │ .60.5     │         │ .60.6    │
   │ Python 2.7│         │ Python 2.7│         │ Python 2.7
   │ ou 3.5+   │         │ ou 3.5+   │         │ ou 3.5+  │
   └───────────┘         └───────────┘         └──────────┘
```

## 🔐 Configuration SSH

### Prérequis : Authentification sans mot de passe

Ansible utilise SSH pour la communication. Pour éviter de saisir un mot de passe à chaque fois, on configure l'authentification par clé.

#### Étape 1 : Générer les clés SSH sur master

```bash
# Connexion à la machine master
vagrant ssh master

# Se positionner en tant qu'utilisateur vagrant
cd ~/.ssh

# Générer la paire de clés RSA (4096 bits pour meilleure sécurité)
ssh-keygen -t rsa -b 4096 -f id_rsa -N ""

# Vérifier la création
ls -la
# Résultat attendu:
# -rw-r--r--  1 vagrant vagrant  3243 id_rsa
# -rw-r--r--  1 vagrant vagrant   762 id_rsa.pub
```

**Contenu de la clé publique :**
```bash
cat ~/.ssh/id_rsa.pub
# Copier la clé affichée pour la suite
```

#### Étape 2 : Copier la clé publique sur chaque nœud

```bash
# Pour chaque nœud (app1, app2, db)

  # Se connecter au nœud
  vagrant ssh $node
  
  # Créer le répertoire .ssh s'il n'existe pas
  mkdir -p ~/.ssh
  
  # Créer le fichier authorized_keys
  cat >> ~/.ssh/authorized_keys << 'EOF'
# Coller la clé publique ici (résultat de cat ~/.ssh/id_rsa.pub)
EOF
  
  # Configurer les permissions
  chmod 700 ~/.ssh
  chmod 600 ~/.ssh/authorized_keys
  
  # Quitter le nœud
  exit

```

#### Étape 3 : Tester l'accès SSH sans mot de passe

```bash
# Depuis master
ssh vagrant@192.168.60.4 "hostname"
# Devrait afficher: app1.local

ssh vagrant@192.168.60.5 "hostname"
# Devrait afficher: app2.local

ssh vagrant@192.168.60.6 "hostname"
# Devrait afficher: db.local
```

## 📦 Installation d'Ansible


```bash


# 1. Installer Ansible
sudo apt install ansible

# 2. Vérifier l'installation
ansible --version
```

**Output attendu :**
```
ansible 2.9.27
  config file = /etc/ansible/ansible.cfg
  configured module search path = ['/root/.ansible/plugins/modules', ...]
  ansible python module location = /usr/lib/python2.7/site-packages/ansible
  executable location = /usr/bin/ansible
  python version = 2.7.5
```

## ✅ Vérification des Prérequis

### Sur la machine de contrôle (master)

```bash
# Python
python3 --version
# Résultat: Python 3.6+

# SSH
ssh -V
# Résultat: OpenSSH 7.4+

# Ansible
ansible --version
```


## 📝 Configuration de base

### Créer un répertoire de travail

```bash
# Sur master, créer un répertoire pour les playbooks
mkdir -p ~/ansible-workspace
cd ~/ansible-workspace
```

## 🧪 Premières Commandes de Test

### Test de connectivité basique

```bash
# Depuis master
cd ~/ansible-workspace

# Tester la connexion ping
ansible all -i "192.168.60.4,192.168.60.5,192.168.60.6," -m ping

# Ou avec des noms d'hôte
ansible all -i "app1,app2,db," -m ping
```

**Output attendu :**
```
192.168.60.4 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}
```

### Informations système

```bash
# Récupérer les informations système
ansible all -i "192.168.60.4," -m setup -a "filter=ansible_distribution*"

# Voir les informations mémoire
ansible all -i "192.168.60.4," -m setup -a "filter=ansible_memtotal_mb"
```

## 📋 Exercices Pratiques


### Exercice 1 : Créer un inventaire simple

Créer un fichier `inventory.ini` :

```ini
[all_servers]
app1 ansible_host=192.168.60.4
app2 ansible_host=192.168.60.5
db   ansible_host=192.168.60.6

[webservers]
app1
app2

[databases]
db

[all_servers:vars]
ansible_user=vagrant
ansible_ssh_private_key_file=~/.ssh/id_rsa
```

Tester :
```bash
ansible-inventory -i inventory.ini --graph
ansible all -i inventory.ini -m ping
```

### Exercice 3 : Exécuter des commandes ad-hoc

```bash
# Afficher la date sur tous les serveurs
ansible all -i inventory.ini -a "date"

# Vérifier l'espace disque
ansible all -i inventory.ini -a "df -h"

# Lister les utilisateurs
ansible all -i inventory.ini -a "cat /etc/passwd | head -5"
```

## 🐛 Dépannage

### Erreur : "Permission denied (publickey,password)"

```bash
# Vérifier que la clé est en place sur le serveur
ssh-keyscan 192.168.60.4 >> ~/.ssh/known_hosts

# Tester la connexion SSH
ssh -vvv vagrant@192.168.60.4

# Vérifier les permissions sur le serveur
ssh vagrant@192.168.60.4 "ls -la ~/.ssh && cat ~/.ssh/authorized_keys"
```

### Erreur : "ansible: command not found"

```bash
# Vérifier l'installation
pip3 list | grep ansible

# Réinstaller
pip3 install --force-reinstall ansible
```

### Erreur : "Python not found"

```bash
# Spécifier le chemin Python dans le playbook ou l'inventaire
# Dans inventory.ini:
[all_servers:vars]
ansible_python_interpreter=/usr/bin/python3
```

## 📚 Ressources Additionnelles

- Documentation Ansible: https://docs.ansible.com/
- Modules Ansible: https://docs.ansible.com/ansible/latest/collections/
- Best Practices: https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html

---

**Étape suivante** → [Lab 2 - Configuration d'Ansible](./Lab-02-Configuration.md)
