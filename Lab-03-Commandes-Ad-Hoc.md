# Lab 3 - Commandes Ad-Hoc Ansible

## 🎯 Objectifs

À la fin de ce lab, vous serez capable de :
- Exécuter des commandes ad-hoc efficaces
- Comprendre les modules courants
- Utiliser les options de ligne de commande
- Automatiser des tâches simples sans playbooks
- Gérer les erreurs et les sorties

## 📋 Introduction

Les commandes ad-hoc sont des commandes Ansible exécutées directement depuis la ligne de commande sans créer de playbook. Elles permettent d'exécuter des tâches simples rapidement.

### Syntaxe Générale

```
ansible [hôtes] -i [inventaire] -m [module] -a [arguments] [options]
```

### Schéma d'Exécution

```
┌─────────────────────────────────────────────┐
│         Commande Ad-Hoc                     │
│  ansible app1 -m ping -i inventory.ini      │
└─────────────────────────────────────────────┘
                      │
         ┌────────────┴────────────┐
         ▼                         ▼
    Parsing             Connection SSH
    Arguments           à app1
         │                     │
         └────────────┬────────┘
                      ▼
          Transfert du module Python
                      │
         ┌────────────┴────────────┐
         ▼                         ▼
    Exécution du module    Récupération résultat
         │                         │
         └────────────┬────────────┘
                      ▼
              Affichage du résultat
                 (JSON)
```

## 🔧 Modules Courants

### 1. Module `ping`

Le plus simple pour tester la connectivité.

```bash
# Tester un seul hôte
ansible app1 -i inventory.ini -m ping

# Tester tout un groupe
ansible webservers -i inventory.ini -m ping

# Tester tous les hôtes
ansible all -i inventory.ini -m ping

# Format JSON
ansible app1 -i inventory.ini -m ping -o  # JSON sur une ligne
```

**Output :**
```json
app1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

### 2. Module `command` et `shell`

Exécuter des commandes.

```bash
# Exécuter une commande simple (command)
ansible app1 -i inventory.ini -m command -a "uptime"

# Résultat
# app1 | CHANGED => {
#    "stdout": "13:45:21 up 2:30, 2 users, load average: 0.10, 0.15, 0.08"
# }

# Exécuter avec pipes et redirections (shell)
ansible app1 -i inventory.ini -m shell -a "cat /etc/passwd | head -1"

# Différence:
# - command: ne supporte pas pipes (|), redirections (>, <)
# - shell: supporte pipes, redirections, variables d'environnement
```

**Quand utiliser :**
- `command` : plus sûr, plus rapide (pas d'interpréteur)
- `shell` : quand vous avez besoin de pipes/redirections

### 3. Module `setup`

Récupérer les variables système (facts).

```bash
# Toutes les variables (long!)
ansible app1 -i inventory.ini -m setup

# Filtrer par pattern
ansible app1 -i inventory.ini -m setup -a "filter=ansible_os_family"

# Informations réseau
ansible app1 -i inventory.ini -m setup -a "filter=ansible_default_ipv4"

# Mémoire
ansible app1 -i inventory.ini -m setup -a "filter=ansible_memory_mb"

# Distribution
ansible app1 -i inventory.ini -m setup -a "filter=ansible_distribution*"
```

**Facts utiles :**
```bash
# Hostname
ansible_hostname

# OS
ansible_os_family
ansible_distribution
ansible_distribution_version

# Réseau
ansible_default_ipv4.address
ansible_interfaces

# Matériel
ansible_processor_vcpus
ansible_memtotal_mb
ansible_swaptotal_mb
```

### 4. Module `file`

Gérer les fichiers et répertoires.

```bash
# Créer un répertoire
ansible app1 -i inventory.ini -m file -a "path=/tmp/test state=directory"

# Créer un fichier
ansible app1 -i inventory.ini -m file -a "path=/tmp/test.txt state=touch"

# Définir les permissions
ansible app1 -i inventory.ini -m file -a "path=/tmp/test.txt mode=0644"

# Changer le propriétaire
ansible app1 -i inventory.ini -m file \
  -a "path=/tmp/test.txt owner=vagrant group=vagrant"

# Supprimer un fichier
ansible app1 -i inventory.ini -m file -a "path=/tmp/test.txt state=absent"

# Créer un lien symbolique
ansible app1 -i inventory.ini -m file \
  -a "src=/var/www dest=/tmp/www state=link"
```

### 5. Module `copy`

Copier des fichiers.

```bash
# Copier un fichier local
ansible app1 -i inventory.ini -m copy \
  -a "src=./myfile.txt dest=/tmp/myfile.txt"

# Avec contenu inline
ansible app1 -i inventory.ini -m copy \
  -a "content='Hello World' dest=/tmp/hello.txt"

# Préserver les permissions et propriétaire
ansible app1 -i inventory.ini -m copy \
  -a "src=./config.ini dest=/etc/myapp/config.ini \
      owner=root group=root mode=0644"

# Copier à tous les webservers
ansible webservers -i inventory.ini -m copy \
  -a "src=./app/www dest=/var/www"
```

### 6. Module `yum` / `apt`

Gérer les paquets.

```bash
# Installer un paquet (ubuntu)
ansible app1 -i inventory.ini -m yum -a "name=nginx state=present"

# Sur tous les webservers
ansible webservers -i inventory.ini -m yum \
  -a "name=nginx,php,curl state=present"

# Désinstaller
ansible app1 -i inventory.ini -m apt -a "name=nginx state=absent"

# Mettre à jour tous les paquets
ansible app1 -i inventory.ini -m apt -a "name=* state=latest"

# Installer depuis apt (Debian/Ubuntu)
ansible app1 -i inventory.ini -m apt -a "name=nginx state=present update_cache=yes"
```

### 7. Module `service`

Gérer les services.

```bash
# Démarrer un service
ansible app1 -i inventory.ini -m service \
  -a "name=nginx state=started"

# Arrêter
ansible app1 -i inventory.ini -m service \
  -a "name=nginx state=stopped"

# Redémarrer
ansible app1 -i inventory.ini -m service \
  -a "name=nginx state=restarted"

# Recharger la configuration
ansible app1 -i inventory.ini -m service \
  -a "name=nginx state=reloaded"

# Activer au démarrage
ansible app1 -i inventory.ini -m service \
  -a "name=nginx enabled=yes"

# Tous ensemble
ansible app1 -i inventory.ini -m service \
  -a "name=nginx state=started enabled=yes"
```

### 8. Module `user`

Gérer les utilisateurs.

```bash
# Créer un utilisateur
ansible app1 -i inventory.ini -m user \
  -a "name=appuser shell=/bin/bash home=/home/appuser"

# Avec groupe supplémentaire
ansible app1 -i inventory.ini -m user \
  -a "name=appuser groups=wheel append=yes"

# Modifier le shell
ansible app1 -i inventory.ini -m user \
  -a "name=appuser shell=/bin/bash"

# Supprimer un utilisateur
ansible app1 -i inventory.ini -m user \
  -a "name=appuser state=absent"

# Supprimer le home directory aussi
ansible app1 -i inventory.ini -m user \
  -a "name=appuser state=absent remove=yes"
```

### 9. Module `lineinfile`

Modifier des fichiers ligne par ligne.

```bash
# Ajouter une ligne à la fin du fichier
ansible app1 -i inventory.ini -m lineinfile \
  -a "path=/etc/hosts line='192.168.60.4 app1'"

# Remplacer une ligne existante
ansible app1 -i inventory.ini -m lineinfile \
  -a "path=/etc/fstab \
      regexp='UUID=.*' \
      line='UUID=1234567890 /data ext4 defaults 0 2'"

# Remplacer avec backreference
logout

# Ajouter une ligne avant une autre
ansible app1 -i inventory.ini -m lineinfile \
  -a "path=/etc/sudoers \
      insertbefore='%wheel ALL' \
      line='%sudo ALL=(ALL) NOPASSWD:ALL'"

# S'assurer qu'une ligne N'existe PAS
ansible app1 -i inventory.ini -m lineinfile \
  -a "path=/etc/ssh/sshd_config \
      regexp='^PermitRootLogin' \
      state=absent"
```

## 🎯 Cas d'Usage Courants

### Recueillir des Informations

```bash
# Vérifier la distribution
ansible all -i inventory.ini -m setup \
  -a "filter=ansible_distribution*"

# Vérifier l'espace disque
ansible all -i inventory.ini -m shell -a "df -h"

# Lister les processus
ansible all -i inventory.ini -m shell -a "ps aux | grep apache"

# Vérifier l'uptime
ansible all -i inventory.ini -a "uptime"

# Vérifier la configuration réseau
ansible all -i inventory.ini -m setup \
  -a "filter=ansible_interfaces"
```

### Installer et Configurer

```bash
# Installation complète de serveur web
ansible webservers -i inventory.ini -m apt \
  -a "name=nginx,php-fpm state=present"

ansible webservers -i inventory.ini -m service \
  -a "name=nginx state=started enabled=yes"

# Créer un utilisateur d'application
ansible app1 -i inventory.ini -m user \
  -a "name=webapp shell=/bin/bash groups=wheel"

# Créer les répertoires d'application
ansible app1 -i inventory.ini -m file \
  -a "path=/opt/webapp state=directory owner=webapp"
```

### Gérer la Configuration

```bash
# Sauvegarder et restaurer fichiers config
ansible app1 -i inventory.ini -m copy \
  -a "src=/etc/nginx/nginx.conf dest=./nginx.conf.backup"

# Mettre à jour une configuration
ansible app1 -i inventory.ini -m lineinfile \
  -a "path=/etc/nginx/nginx.conf \
      regexp='worker_processes' \
      line='    worker_processes auto;'"

# Recharger le service après modification
ansible app1 -i inventory.ini -m service \
  -a "name=nginx state=reloaded"
```

## 📊 Options de Ligne de Commande Utiles

```bash
# Verbose (affiche plus de détails)
ansible all -i inventory.ini -m ping -v      # -v
ansible all -i inventory.ini -m ping -vv     # -vv
ansible all -i inventory.ini -m ping -vvv    # -vvv
ansible all -i inventory.ini -m ping -vvvv   # -vvvv

# Format JSON sur une ligne
ansible all -i inventory.ini -m ping -o

# Limite les hôtes
ansible all -i inventory.ini -m ping --limit=app1
ansible all -i inventory.ini -m ping --limit=webservers

# Exécuter sur un seul hôte
ansible app1 -i inventory.ini -m ping --limit=1

# Ignorer certains hôtes
ansible all -i inventory.ini -m ping --limit="!db"
ansible all -i inventory.ini -m ping --limit="webservers:!app2"

# Afficher les hôtes concernés sans exécuter
ansible all -i inventory.ini --list-hosts

# Syntaxe vérification
ansible-playbook playbook.yml --syntax-check

# Mode dry-run (check mode)
ansible app1 -i inventory.ini -m apt -a "name=nginx" --check

# Donner les détails des changements
ansible app1 -i inventory.ini -m apt -a "name=nginx" --diff

# Demander une confirmation avant exécution
ansible all -i inventory.ini -m ping --ask-become-pass

# Spécifier un inventaire particulier
ansible all -i /path/to/inventory.ini -m ping

# Paralléliser sur plusieurs forks
ansible all -i inventory.ini -m ping -f 10

# Timeout personnalisé
ansible all -i inventory.ini -m ping --timeout=5
```

## 🧪 Exercices Pratiques

### Exercice 1 : Tester la Connectivité

1. Ping sur tous les hôtes
2. Afficher le type d'OS
3. Afficher la version du kernel

```bash
# Solution
ansible all -i inventory.ini -m ping
ansible all -i inventory.ini -m setup -a "filter=ansible_os_family"
ansible all -i inventory.ini -m setup -a "filter=ansible_kernel_version"
```

### Exercice 2 : Installer et Configurer Nginx

1. Installer nginx sur webservers
2. Démarrer et activer le service
3. Vérifier qu'il est actif

```bash
# Solution
ansible webservers -i inventory.ini -m yum -a "name=nginx state=present"
ansible webservers -i inventory.ini -m service -a "name=nginx state=started enabled=yes"
ansible webservers -i inventory.ini -m shell -a "systemctl status nginx"
```

### Exercice 3 : Gérer des Fichiers

1. Créer un répertoire `/opt/app` sur app1
2. Copier un fichier de configuration
3. Vérifier les permissions

```bash
# Solution
ansible app1 -i inventory.ini -m file -a "path=/opt/app state=directory mode=0755"
ansible app1 -i inventory.ini -m copy -a "content='app_config' dest=/opt/app/config.txt"
ansible app1 -i inventory.ini -m shell -a "ls -la /opt/app"
```

### Exercice 4 : Générer un Rapport

Créer un script pour générer un rapport système :

```bash
#!/bin/bash
# rapport_system.sh

echo "=== RAPPORT SYSTÈME ===" > rapport.txt
echo "" >> rapport.txt

echo "Distribution:" >> rapport.txt
ansible all -i inventory.ini -m setup -a "filter=ansible_distribution*" >> rapport.txt

echo "" >> rapport.txt
echo "Mémoire:" >> rapport.txt
ansible all -i inventory.ini -m setup -a "filter=ansible_memtotal_mb" >> rapport.txt

echo "" >> rapport.txt
echo "CPU:" >> rapport.txt
ansible all -i inventory.ini -m setup -a "filter=ansible_processor_vcpus" >> rapport.txt

echo "" >> rapport.txt
echo "Uptime:" >> rapport.txt
ansible all -i inventory.ini -a "uptime" >> rapport.txt

cat rapport.txt
```


## Application 1
1- En utilisant Ansible en ligne de commande, vous allez réaliser les actions suivantes :
* Afficher la date et l'heure actuelles sur les hôtes cibles
* Vérifier l'espace disque disponible sur les hôtes cibles.
* Installer le logiciel `tree`
* créer un fichier vide `/tmp/test` avec les droits 644
* Créer les groupes `rennes`, `lille` et `paris`
* Créer un utilisateur `john`
* Vérifier que l'utilisateur john a été créé sur les hôtes cibles
* Modifier l'utilisateur `john` pour qu'il ait l'UID 10000
* Modifier l'utilisateur `john` pour qu'il soit dans le groupe `rennes`


2- Écrire des commandes Ansible ad-hoc pour réaliser les actions suivantes :

*Créer l'utilisateur automation sur tous les hôtes de l'inventaire.
*Copier la clé SSH publique générée dans /home/automation/.ssh/authorized_keys sur tous les hôtes.
*Configurer l'utilisateur automation pour qu'il puisse élever ses privilèges sans mot de passe sur tous les hôtes.
*Tester la connexion SSH et les commandes avec élévation de privilèges.


## Application 2
**Configuration de l'inventaire (inventory) :
Assurez-vous que votre fichier d'inventaire est à jour avec les adresses IP 
correctes de vos serveurs.

**a. Installation de dépendances pour une application :

Utilisez la commande ad-hoc pour installer les dépendances nécessaires à une application
fictive sur tous les serveurs d'application.

```
ansible app1 -m yum -a "name=python3 state=present" --become

ansible app1 -m shell -a "yum install -y python3 python3-pip" --become
```

** Installation de git:
Utilisez la commande ad-hoc pour installer git sur tous les serveurs d'application.

**b. Clonage d'un référentiel Git :

Utilisez la commande ad-hoc pour cloner un référentiel Git sur tous les serveurs d'application.
Clone le dépôt GitHub: https://github.com/render-examples/flask-hello-world.git
dans le répertoire: /opt/flask-app

*si des erreurs de certificat ssl retourné, vous pouvez cloner localement le dépot, 
*puis copier dans les noeuds

```ansible webservers -m git -a "repo=https://github.com/render-examples/flask-hello-world.git dest=/opt/flask-app" --become```

**c. Configuration d'un fichier de configuration :

Utilisez la commande ad-hoc pour configurer un fichier de configuration spécifique sur tous les serveurs d'application.

```
ansible webservers -m ini_file -a "dest=/opt/flask-app/config.ini section=production option=debug value=false" --become```

Installer les dépendances depuis le fichier requirements.txt 
avec le gestionnaire de package pip .

``` 
ansible webservers -m shell -a "/usr/bin/python3 -m pip install -r /opt/flask-app/requirements.txt" --become 
 ```
** Créez un fichier de service systemd, par exemple flask-app.service
** puis le copier dans le répertoire /etc/systemd/system/  des serveurs d'application:
[Unit]
Description=Flask App Service
After=network.target

[Service]
ExecStart=/usr/bin/python3 /usr/local/bin/gunicorn app:app
WorkingDirectory=/opt/flask-app
User=vagrant
Group=vagrant
Restart=always

[Install]
WantedBy=multi-user.target

**d. Démarrage d'un service d'application :

Utilisez la commande ad-hoc pour démarrer un service d'application sur tous les serveurs.


```
ansible webservers -m shell -a "systemctl daemon-reload" -b
ansible webservers -m systemd -a "name=flask-app state=started" --become

```

**e. Vérification de l'état du service :

Utilisez la commande ad-hoc pour vérifier l'état du service sur tous les serveurs d'application.

```ansible webservers -m systemd -a "name=flask-app state=started" --become	```			 
---




## 🐛 Dépannage

### Erreur : "No hosts matched"

```bash
# Vérifier l'inventaire
ansible-inventory -i inventory.ini --list

# Vérifier la syntaxe du group
ansible-inventory -i inventory.ini --graph

# Utiliser -v pour voir ce qui se passe
ansible all -i inventory.ini -m ping -v
```

### Erreur : "Permission denied"

```bash
# Vérifier la clé SSH
ansible app1 -i inventory.ini -m ping -vvv

# Utiliser --ask-pass pour saisir le mot de passe
ansible app1 -i inventory.ini -m ping --ask-pass

# Utiliser sudo
ansible app1 -i inventory.ini -m command -a "whoami" --ask-become-pass
```

### Erreur : "Module not found"

```bash
# Vérifier que le module existe
ansible-doc -l | grep -i modulename

# Voir la documentation du module
ansible-doc copy

# Vérifier le chemin des modules
ansible app1 -i inventory.ini -m debug -a "msg={{ ansible_modules_path }}"
```

## 📚 Ressources Additionnelles

- Module Documentation: https://docs.ansible.com/ansible/latest/collections/
- Ad-Hoc Commands: https://docs.ansible.com/ansible/latest/user_guide/basic_concepts.html#ad-hoc-commands
- Best Practices: https://docs.ansible.com/ansible/latest/user_guide/intro_adhoc.html

---

**Étape suivante** → [Lab 4 - Playbooks Ansible](./Lab-04-Playbooks.md)
