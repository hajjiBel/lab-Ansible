# Lab 5 – Variables Ansible

## Table des matières

- [Introduction](#introduction)
- [Gestion des Variables](#gestion-des-variables)
  - [Définir des variables dans les playbooks](#définir-des-variables-dans-les-playbooks)
- [Variables d'hôte et de groupe](#variables-dhôte-et-de-groupe)
  - [Créer les fichiers variables](#créer-les-fichiers-variables)
  - [Créer des fichiers index.html](#créer-des-fichiers-indexhtml)
  - [Créer le playbook](#créer-le-playbook)
  - [Tester le résultat](#tester-le-résultat)
- [Challenge Lab: Utilisation des variables d'inventaire](#challenge-lab-utilisation-des-variables-dinventaire)
- [Variables enregistrées](#variables-enregistrées)
- [Ansible Facts](#ansible-facts)
  - [Challenge Lab: Facts](#challenge-lab-facts)
  - [Utilisation des facts dans les playbooks](#utilisation-des-facts-dans-les-playbooks)
- [Application 1](#application-1)

---

## Introduction

Ansible prend en charge les **variables** permettant de stocker des valeurs pouvant être utilisées dans les Playbooks. Les variables peuvent être définies à divers endroits et ont une priorité claire. Ansible remplace la variable par sa valeur lorsqu'une tâche est exécutée.

### Syntaxe des variables

Les variables sont référencées dans les Playbooks en plaçant le nom de la variable entre deux accolades :

```yaml
{{ variable1 }}
```

### Pratique recommandée

La pratique recommandée est de définir des variables dans des fichiers situés dans deux répertoires nommés `host_vars` et `group_vars` :

- **Variables de groupe** : Pour définir des variables pour le groupe `servers`, créez un fichier YAML nommé `group_vars/servers` avec les définitions de variables.

- **Variables d'hôte** : Pour définir des variables spécifiquement pour l'hôte `host1.example.com`, créez le fichier `host_vars/host1.example.com` avec les définitions de variables.

> **Note** : Les variables hôtes ont priorité sur les variables de groupe.

---

## Gestion des Variables

Nous pouvons définir des variables à différents endroits dans un projet Ansible et avec différentes portées.

### Portées des variables

| Portée | Description |
|--------|-------------|
| **Global Scope** | Variables définies à partir de l'interface de ligne de commande ou de la configuration Ansible (`ansible.cfg`) |
| **Play Scope** | Les variables sont définies dans le playbook |
| **Host Scope** | Variables définies sur des groupes d'hôtes ou des hôtes individuelles par l'inventaire, fact gathering, etc. |

---

### Définir des variables dans les playbooks

Les administrateurs peuvent définir leurs propres variables dans les Playbooks et les utiliser dans des tâches.

#### Méthodes de définition

Les variables de playbooks peuvent être définies de plusieurs manières :

1. En les plaçant directement dans un bloc `vars` au début d'un Playbook
2. En définissant les variables de Playbook dans des fichiers externes

#### Exercice pratique : Créer un utilisateur système

Créez un playbook `users.yml` pour créer un utilisateur système ayant les informations suivantes :

- **user** : admin
- **group** : wheel
- **home directory** : /dev/null
- **shell** : /bin/false

**Solution avec variables intégrées :**

```yaml
---
- name: Create system user
  hosts: all
  become: yes
  vars:
    user: admin
    group: wheel
    homedir: /dev/null
    shell: /bin/false
  tasks:
    - name: Add user
      user:
        name: "{{ user }}"
        group: "{{ group }}"
        home: "{{ homedir }}"
        shell: "{{ shell }}"
```

#### Déplacer les variables dans un fichier externe

**Playbook principal :**

```yaml
---
- name: Create system user
  hosts: all
  become: yes
  vars_files:
    - users.yml
  tasks:
    - name: Add user
      user:
        name: "{{ user }}"
        group: "{{ group }}"
        home: "{{ homedir }}"
        shell: "{{ shell }}"
```

**Fichier `users.yml` avec les variables :**

```yaml
---
user: admin
group: wheel
homedir: /dev/null
shell: /bin/false
```

---

## Variables d'hôte et de groupe

### Objectif

Nous souhaitons déployer des fichiers `index.html` différents selon le type du serveur (**dev** / **prod**) sur lequel il est déployé.

### Étape 1 : Créer les répertoires

Sur **master** en tant qu'utilisateur **vagrant**, créez les répertoires contenant les définitions de variable `host` et `group` :

```bash
mkdir {host_vars,group_vars}
```

---

### Créer les fichiers variables

Créez maintenant deux fichiers contenant des définitions de variables qui pointent vers un environnement.

#### Fichier `group_vars/webservers`

```yaml
---
stage: dev

```

#### Fichier `host_vars/app2`

```yaml
---
stage: prod

```

---

### Créer des fichiers index.html

Créez deux fichiers HTML pour les différents environnements.

#### Fichier `prod_index.html`

```html
<body>
  <h1>This is a production webserver, take care!</h1>
</body>
```

#### Fichier `dev_index.html`

```html
<body>
  <h1>This is a development webserver, have fun!</h1>
</body>
```

---

### Créer le playbook

Vous avez maintenant besoin d'un Playbook qui copie le fichier `prod` ou `dev_index.html` en fonction de la variable **stage**.

Créez un nouveau Playbook appelé `deploy_index_html.yml`.

> **Note** : Notez comment la variable **stage** est utilisée dans le nom du fichier à copier.

```yaml
---
- name: Copy index.html
  hosts: webservers
  become: yes
  tasks:
    - name: Copy index.html
      copy:
        src: "{{ stage }}_index.html"
        dest: /tmp/index.html
```

### Exécutez le Playbook

```bash
ansible-playbook deploy_index_html.yml
```

---

### Tester le résultat

Le Playbook doit copier différents fichiers sous le nom `index.html` sur les hôtes. Utilisez `curl` pour le tester :

**Test sur app1 (dev) :**

```bash
[master]$ curl 192.168.60.4
```

**Sortie attendue :**

```html
<body>
  <h1>This is a development webserver, have fun!</h1>
</body>
```

**Test sur app2 (prod) :**

```bash
[master]$ curl 192.168.60.5
```

**Sortie attendue :**

```html
<body>
  <h1>This is a production webserver, take care!</h1>
</body>
```

> **Remarque** : Peut-être vous pensez maintenant : il doit y avoir un moyen plus intelligent de modifier le contenu des fichiers... vous avez absolument raison. Ce lab a été conçu pour introduire les variables. Vous êtes sur le point de connaître les modèles (templates) dans l'un des prochains labs.

---

## Challenge Lab: Utilisation des variables d'inventaire

### Objectif

Vous avez maintenant toutes les informations pour effectuer les tâches suivantes :

### Tâches

1. Dans le fichier de variables du groupe **webservers**, définissez une variable **service** par `sshd`.

2. Dans le fichier de variable hôte de l'hôte **app1**, définissez une variable **service** par `httpd`.

3. Créez un playbook `check_service.yml` pour redémarrer un service dont le nom est défini par la variable **service**. Rendez-le applicable à tous les hôtes.

4. Exécutez le Playbook avec `-v` pour voir Ansible vérifie différents services en fonction des variables.

---

### Solution

#### Fichier `host_vars/app1`

```yaml
---
stage: prod
service: apache2
```

#### Fichier `group_vars/webservers`

```yaml
---
stage: dev
service: sshd
```

#### Playbook `check_service.yml`

```yaml
---
- name: Check if service is enabled and started
  hosts: webservers
  become: yes
  tasks:
    - name: Check service is enabled and started
      service:
        name: "{{ service }}"
        enabled: true
        state: started
```

#### Exécution

```bash
ansible-playbook check_service.yml -v
```

---

## Variables enregistrées

Les administrateurs peuvent capturer le résultat d'une commande à l'aide de l'instruction `register`. La sortie est enregistrée dans une variable qui pourra être utilisée ultérieurement, soit à des fins de débogage, soit pour réaliser autre chose, telle qu'une configuration particulière basée sur la sortie d'une commande.

### Exemple pratique

Le Playbook suivant montre cette utilisation. Créez-le en tant que `register_var.yml` et exécutez-le :

```yaml
---
- name: Installs a package and prints the result
  hosts: dbservers
  become: true
  tasks:
    - name: Install the package
      yum:
        name: mariadb
        state: present
        update_cache: yes
      register: install_result

    - name: Display installation result
      debug:
        msg: "{{ install_result }}"
```

### Fonctionnement

Quand ce Playbook est lancé :

- Le module de débogage est utilisé pour vider la valeur de la variable enregistrée `install_result` dans le terminal.
- Cela peut être très utile lors du débogage.
- Dans ce cas, le paquet était déjà installé et  reflété par la sortie du module capturée dans la variable.

### Affiner la sortie

Modifiez le task `debug` pour afficher seulement le message retourné :

```yaml
- name: Display only results
  debug:
    var: install_result.results
```

---

## Ansible Facts

### Qu'est-ce qu'un Fact ?

**Ansible Facts** sont des variables automatiquement découvertes par Ansible à partir d'une hôte gérée. **Facts** sont extraits par le module `setup` et contiennent des informations utiles stockées dans des variables que les administrateurs peuvent réutiliser.

### Explorer les Facts

Pour avoir une idée des **facts** recueillis par Ansible par défaut, sur **master** en tant qu'utilisateur `vagrant`, exécutez :

```bash
ansible app1 -m setup
```

### Utiliser des filtres

C'est peut-être un peu trop. Vous pouvez utiliser des **filtres** pour limiter la sortie à certains faits. L'expression est un joker de style shell :

**Filtrer les informations réseau :**

```bash
ansible app1 -m setup -a 'filter=ansible_eth0'
```

**Rechercher les faits liés à la mémoire :**

```bash
ansible all -m setup -a 'filter=ansible_*_mb'
```

---

### Challenge Lab: Facts

#### Objectif

Essayez de trouver et d'imprimer la distribution (CentOS) de vos hôtes gérées sur une seule ligne. Utilisez `grep` pour trouver le **Fact**, puis appliquez un filtre pour n'imprimer que ce fact.

#### Solution

**Étape 1 : Rechercher le fact**

```bash
ansible app1 -m setup | grep distribution
```

**Étape 2 : Filtrer et afficher**

```bash
ansible all -m setup -a 'filter=ansible_distribution' -o
```

---

### Utilisation des facts dans les playbooks

Les **facts** peuvent être utilisés dans un Playbook comme des variables, en utilisant le nom approprié. Créez le Playbook `facts.yml` :

```yaml
---
- name: Output facts within a playbook
  hosts: webservers
  tasks:
    - name: Prints Ansible facts
      debug:
        msg: "The default IPv4 address of {{ ansible_fqdn }} is {{ ansible_eth1.ipv4.address }}"
```

### Exécution

```bash
ansible-playbook facts.yml
```

### Sortie attendue

```
PLAY [Output facts within a playbook] *********************************

TASK [Gathering Facts] ************************************************
ok: [app1]
ok: [app2]

TASK [Prints Ansible facts] *******************************************
ok: [app1] => {
    "msg": "The default IPv4 address of app1 is 192.168.60.4"
}
ok: [app2] => {
    "msg": "The default IPv4 address of app2 is 192.168.60.5"
}

PLAY RECAP ************************************************************
app1 : ok=2 changed=0 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
app2 : ok=2 changed=0 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
```

> **Note** : Notez comment le module **setup** a été appelé implicitement par Ansible sous le nom de tâche "Gathering Facts".

---

Déploiement Docker + Apache

## Application 1

### Objectifs

- Créer un dossier `webapp` qui va contenir tous les fichiers du projet
- Créer un fichier d'inventaire appelé `prod.yml` (au format YAML) contenant un groupe `prod` avec comme seul membre notre client
- Créer un dossier `group_vars` qui va contenir un fichier nommé `prod` contenant les informations de connexion à utiliser par Ansible (login et mot de passe)
- Créer un playbook nommé `deploy.yml` permettant de déployer Apache à l'aide de Docker sur le client (l'image à utiliser est `httpd` et le port à exposer à l'extérieur est le 80)
- Installer tout prérequis nécessaire à l'aide du module approprié
- Vérifier la syntaxe du playbook avec la commande `ansible-lint` (installer si nécessaire)
- Vérifier qu'après l'exécution du playbook, le site par défaut d'Apache est bien disponible sur le port 80

---

## Solution : Déploiement Docker + Apache

### 1. Arborescence du projet

Dans votre répertoire de travail sur le nœud de contrôle Ansible :

```bash
mkdir -p webapp/group_vars
cd webapp
```

**Arborescence attendue :**

```
webapp/
  ├─ prod.yml          # inventaire au format YAML
  ├─ group_vars/
  │    └─ prod         # variables du groupe prod
  └─ deploy.yml        # playbook de déploiement Docker+Apache
```

---

### 2. Fichier d'inventaire : prod.yml

Créez le fichier `prod.yml` à la racine du dossier `webapp` :

```yaml
---
all:
  children:
    prod:
      hosts:
        app1:
          ansible_host: 192.168.1.100  # Adresse IP de votre client
```

**Remarque :** Remplacez `192.168.1.100` par l'adresse IP réelle de votre machine cliente.

---

### 3. Variables de groupe : group_vars/prod

Dans `group_vars/prod` (sans extension), placez les informations de connexion à utiliser par Ansible (login + mot de passe, plus éventuellement sudo) :

```yaml
# group_vars/prod
ansible_user: devops                    # utilisateur SSH sur le client
ansible_password: "MotDePasseSSH"       # mot de passe SSH
ansible_connection: ssh

ansible_become: true
ansible_become_method: sudo
ansible_become_password: "MotDePasseSudo"
```

**Remarques :**
- Remplacez `devops` par votre nom d'utilisateur réel
- Remplacez `MotDePasseSSH` et `MotDePasseSudo` par vos mots de passe réels
- Pour plus de sécurité en production, utilisez `ansible-vault` pour chiffrer ce fichier

---

### 4. Playbook : deploy.yml

Créez le fichier `deploy.yml` à la racine du dossier `webapp` :

```yaml
---
- name: Déployer Apache via Docker sur le client
  hosts: prod
  become: true

  vars:
    httpd_image: "httpd:latest"
    httpd_container_name: "webapp_httpd"
    httpd_host_port: 80
    httpd_container_port: 80

  tasks:
    - name: Installer Docker et dépendances Python pour Docker
      apt:
        name:
          - docker.io
          - python3-pip
          - python3-docker
        state: present
        update_cache: yes

    - name: Corriger la pile Python Docker (bug http+docker)
      ansible.builtin.pip:
        name:
          - "requests<2.32.0"
          - "docker>=6.1.0"
        executable: pip3
      when: ansible_os_family == "Debian"

    - name: S'assurer que le service Docker est démarré et activé
      service:
        name: docker
        state: started
        enabled: true

    - name: Déployer le conteneur Apache httpd
      community.docker.docker_container:
        name: "{{ httpd_container_name }}"
        image: "{{ httpd_image }}"
        state: started
        restart_policy: always
        ports:
          - "{{ httpd_host_port }}:{{ httpd_container_port }}"
```

---

### 5. Installation d'ansible-lint

Sur le nœud de contrôle Ansible :

```bash
pip3 install ansible-lint
```

---

### 6. Vérification de la syntaxe

Vérifiez la syntaxe du playbook avec `ansible-lint` :

```bash
ansible-lint deploy.yml
```

Si des avertissements ou erreurs apparaissent, corrigez-les avant de continuer.

---

### 7. Installation de la collection Docker

Le module `community.docker.docker_container` nécessite la collection Docker :

```bash
ansible-galaxy collection install community.docker
```

---

### 8. Exécution du playbook

Exécutez le playbook pour déployer Apache via Docker :

```bash
ansible-playbook -i prod.yml deploy.yml
```

---

### 9. Vérification du déploiement

#### Sur le client (nœud app1)

Vérifiez que le conteneur Docker est en cours d'exécution :

```bash
sudo docker ps
```

Vous devriez voir un conteneur nommé `webapp_httpd` basé sur l'image `httpd:latest`.

#### Depuis le nœud de contrôle ou n'importe quelle machine

Testez l'accès au site Apache par défaut :

```bash
curl http://192.168.1.100:80
```

Ou ouvrez un navigateur et accédez à : `http://192.168.1.100`

Vous devriez voir la page par défaut d'Apache : **"It works!"**
 
**Avantages de cette approche :**

1. **Réutilisabilité** : Le même playbook peut être utilisé pour différentes applications
2. **Maintenabilité** : Les valeurs sont centralisées en un seul endroit
3. **Flexibilité** : Facile de changer les valeurs sans modifier la logique
4. **Clarté** : Les variables documentent l'intention du code

---

## Résumé

Dans ce lab, vous avez appris à :

- ✅ Définir et utiliser des variables dans les playbooks Ansible
- ✅ Organiser les variables avec `host_vars` et `group_vars`
- ✅ Comprendre la priorité des variables
- ✅ Capturer des résultats avec `register`
- ✅ Utiliser les **Ansible Facts** pour obtenir des informations système
- ✅ Filtrer et utiliser les facts dans vos playbooks
- ✅ Améliorer la modularité de vos playbooks avec des variables

---

## Navigation

**[← Lab 4](./Lab%204.md)** | **[Lab 6 - Contrôle d'Exécution →](./Lab%206%20-%20Contrôle%20d%27Exécution.md)**

---
