# Lab 0 - Créer l'Infrastructure Locale avec Vagrant

## 🎯 Objectifs

À la fin de ce lab, vous serez capable de :
- Comprendre le rôle de Vagrant et VirtualBox
- Créer une infrastructure multi-machines avec Vagrant
- Gérer le cycle de vie des machines virtuelles
- Configurer une architecture pour les labs Ansible

## 📋 Introduction

Vagrant est un outil d'approvisionnement de serveur qui permet de créer et de gérer des environnements de développement reproductibles. Combiné à VirtualBox, il offre une solution gratuite et open-source pour émuler une infrastructure depuis votre poste de travail.

### Schéma de l'Infrastructure

```
┌─────────────────────────────────────────────────────────────┐
│                    VOTRE MACHINE HÔTE                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         VirtualBox (Hyperviseur)                      │  │
│  │                                                       │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │  │
│  │  |db (database)│  │    app1     │  │    app2     │  │  │
│  │  │             │  │(Application)│  │(Application)│  │  │
│  │  │ 192.168.    │  │ 192.168.    │  │ 192.168.    │  │  │
│  │  │ 60.6        │  │ 60.4        │  │ 60.5        │  │  │
│  │  │ ubuntu 22.04│  │ ubuntu 22.04│  │ ubuntu 22.04│  │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  │  │
│  │                                                       │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │              master (Contrôle)                  │  │  │
│  │  │               192.168.60.1                      │  │  │
│  │  │               ubuntu 22.04                      │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  │                                                       │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Tableau de Configuration des Machines

| Nom d'hôte | Adresse IP   | Système d'Exploitation     | Rôle                    |
|-----------|---------------|----------------------    --|------------------------|
| master    | 192.168.60.1  | ubuntu 22.04               | Nœud de contrôle       |
| app1      | 192.168.60.4  | ubuntu 22.04               | Serveur d'application  |
| app2      | 192.168.60.5  | ubuntu 22.04               | Serveur d'application  |
| db        | 192.168.60.6  | ubuntu 22.04               | Serveur de base données|

## 📦 Prérequis et Installation

### Logiciels requis

1. **VirtualBox** 
   - Télécharger: https://www.virtualbox.org/wiki/Downloads
   - ⚠️ Installer les Extension Pack pour les meilleures performances

2. **Vagrant** 
   - Télécharger: https://developer.hashicorp.com/vagrant/install


## 🏗️ Construction de l'Infrastructure

### Étape 1 : Préparation du répertoire

```bash
# Créer un répertoire de travail
mkdir -p ~/Vms/ansible-training
cd ~/Vms/ansible-training
```

### Étape 2 : Créer le fichier Vagrantfile

Créer un fichier `Vagrantfile` avec le contenu suivant :

```
#  -*-  mode:  ruby -*-
# vi: set ft=ruby  :

VAGRANTFILE_API_VERSION = "2"
Vagrant.configure(VAGRANTFILE_API_VERSION)  do  |config|
  # General Vagrant VM   configuration.
  config.vm.box  =  "ubuntu/jammy64"
  config.ssh.insert_key  =  false
  config.vm.synced_folder  ".",  "/home/vagrant",  disabled:  true
  config.vm.provider  :virtualbox  do  |v|
  	v.memory  =  1024
  	v.linked_clone  =  true
	end
  # Provisionning commun
  config.vm.provision "shell", inline: <<-SHELL
    # Mise à jour système
    apt-get update -qq
    apt-get upgrade -yqq
    
    # Installer outils Ansible
    apt-get install -yqq python3-pip python3-apt sshpass curl wget htop net-tools tree vim
    
  
    # Nettoyage
    apt-get autoremove -yqq
    apt-get clean
    
    echo "Provisionning terminé pour {{ hostname }}"
  SHELL

  #  ControlMaster
  config.vm.define  "master"  do  |app|
    app.vm.hostname  =  "master.dev"
    app.vm.network  :private_network,  ip:  "192.168.60.1"
  end

	#  Application  server 1.
	config.vm.define  "app1"  do  |app|
  	app.vm.hostname  =  "app1.dev"
  	app.vm.network  :private_network,  ip:  "192.168.60.4"
	end

	#  Application  server 2.
	config.vm.define  "app2"  do  |app|
  	app.vm.hostname  =  "app2.dev"
  	app.vm.network  :private_network,  ip:  "192.168.60.5"
	end

	#  Database  server.
	config.vm.define  "db"  do  |db|
  	db.vm.hostname  =  "db.dev"
  	db.vm.network  :private_network,  ip:  "192.168.60.6"
	end
end

```

### Étape 3 : Lancer l'infrastructure

```bash

# Créer et démarrer toutes les machines
vagrant up

# Afficher le statut de toutes les machines
vagrant status
```

**Output attendu :**
```
Current machine states:

master                    running (virtualbox)
app1                      running (virtualbox)
app2                      running (virtualbox)
db                        running (virtualbox)
```

### Étape 4 : Accéder aux machines

```bash
# Accéder à master
vagrant ssh master

# Ou accéder à une autre machine
vagrant ssh app1
vagrant ssh app2
vagrant ssh db

# Depuis une autre fenêtre, accéder directement
vagrant ssh-config  # Affiche la configuration SSH pour toutes les machines
```

## 🔌 Gestion du Cycle de Vie des Machines

### Commandes essentielles

```bash
# Arrêter les machines
vagrant halt                # Arrête toutes les machines
vagrant halt master         # Arrête la machine "master" seulement

# Démarrer les machines
vagrant up                  # Démarre toutes les machines
vagrant up app1 app2        # Démarre app1 et app2

# Redémarrer les machines
vagrant reload
vagrant reload --provision  # Relance les provisionneurs

# Suspendre les machines
vagrant suspend master      # Met en veille
vagrant resume master       # Réveille

# Détruire les machines
vagrant destroy             # Détruit toutes les machines
vagrant destroy app1        # Détruit app1 seulement
vagrant destroy -f          # Force sans confirmation
```

## ✅ Exercices Pratiques

### Exercice 1 : Vérifier la Connectivité

```bash
# Se connecter à chaque machine et vérifier
for machine in master app1 app2 db; do
  echo "=== Vérification de $machine ==="
  vagrant ssh $machine -c "hostname && ip addr show"
  echo ""
done
```

### Exercice 2 : Configurer les Résolutions de Noms

Sur votre machine hôte, ajouter les entrées dans `/etc/hosts` (Linux/Mac):

```
192.168.60.1  master 
192.168.60.4  app1 
192.168.60.5  app2 
192.168.60.6  db 
```

### Exercice 3 : Tester la Connectivité Réseau

```bash
# Depuis la machine master
vagrant ssh master -c "
  echo 'Test de connectivité'
  ping -c 2 192.168.60.4
  ping -c 2 192.168.60.5
  ping -c 2 192.168.60.6
"
```

## 🐛 Dépannage

### Problème : VirtualBox ne trouve pas l'extension pack

```bash
# Télécharger et installer l'extension pack manuellement
# https://www.virtualbox.org/wiki/Downloads

# Vérifier l'installation
VBoxManage list extpacks
```

### Problème : Vagrant n'arrive pas à télécharger la box

```bash
# Essayer une autre source
vagrant box add centos/7 --provider virtualbox

# Ou ajouter manuellement une box
vagrant box add centos/7 ~/Downloads/ubuntu/jammy64.box
```

### Problème : Les machines n'ont pas d'accès réseau

```bash
# Vérifier la carte réseau dans VirtualBox
# Menu Fichier → Gestionnaire de réseau d'hôte

# Redémarrer VirtualBox
vagrant destroy -f
vagrant up

# Si le problème persiste, redémarrer votre machine
```

## 📝 Notes Importantes

- ✅ La première création peut prendre 10-15 minutes selon la vitesse d'Internet
- ✅ L'utilisateur par défaut sur CentOS est `vagrant` (mot de passe: `vagrant`)
- ✅ Les machines n'ont pas d'accès Internet (sauf via le routeur VirtualBox)
- ✅ Les fichiers `/vagrant` sont synchronisés avec le répertoire de l'hôte

---

**Prêt pour le prochain lab ?** → [Lab 1 - Installation d'Ansible](./Lab-01-Installation.md)
