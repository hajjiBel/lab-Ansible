
**Lab 8 – Ansible Roles
======================
## 🎯 Objectifs

À la fin de ce lab, vous serez capable de :
- Comprendre la structure des rôles Ansible
- Créer des rôles réutilisables
- Utiliser les dépendances entre rôles
- Implémenter des rôles complexes
- Partager les rôles avec la communauté

## 📋 Introduction

Un rôle Ansible est une unité réutilisable de code qui encapsule les tâches, variables, templates et fichiers liés à une fonctionnalité spécifique. Les rôles permettent d'organiser le code et de le partager facilement.

### Structure d'un Rôle

```
roles/
├── common/
│   ├── defaults/
│   │   └── main.yml              # Variables par défaut
│   ├── vars/
│   │   └── main.yml              # Variables du rôle
│   ├── files/
│   │   └── config.ini            # Fichiers statiques
│   ├── templates/
│   │   └── sysctl.conf.j2        # Templates Jinja2
│   ├── tasks/
│   │   └── main.yml              # Tâches principales
│   ├── handlers/
│   │   └── main.yml              # Handlers du rôle
│   ├── meta/
│   │   └── main.yml              # Métadonnées et dépendances
│   └── README.md                 # Documentation
├── webserver/
│   ├── defaults/
│   ├── tasks/
│   ├── templates/
│   └── ...
└── database/
    ├── defaults/
    ├── tasks/
    ├── templates/
    └── ...
```
# Créer un rôle Ansible

Pour créer un rôle dans Ansible , utilisez simplement la syntaxe.
```
$ ansible-galaxy init <role_name>
```
Plusieurs répertoires et fichiers seront créés dans votre répertoire de
travail actuel. Dans ce cas, on va créer un rôle dans le répertoire de
travail**.**

Créons un rôle appelé **apache**.
```
$ ansible-galaxy init apache
```
Utilisez la commande **tree** pour avoir un aperçu de la structure de
répertoires du rôle.
```
[master]$ tree apache/

apache/
├── defaults
│ └── main.yml
├── files
├── handlers
│ └── main.yml
├── meta
│ └── main.yml
├── README.md
├── tasks
│ └── main.yml
├── templates
├── tests
│ ├── inventory
│ └── test.yml
└── vars
└── main.yml
```
Comme vous pouvez le voir, plusieurs répertoires ont été créés,
cependant, ils ne seront pas tous utilisés dans le playbook.

Maintenant, pour utiliser votre nouveau rôle dans un playbook,
Vous trouverez ci-dessous un exemple de codes de playbook pour déployer le serveur Web Apache.
```
---
- hosts: all
  tasks:
  - name: Install apache2 Package
    yum: 
      name: apache2
      state: latest
	

  - name: Copy apache2 configuration file
    copy:
      src: apache2.conf
      dest: /etc/apache2/apache2.conf
  notify: restart apache2

  - name: Copy index.html
    copy:
      src: files/index.html
      dest: /var/www/html/
    notify: restart apache2
	
  handlers:
  - name: restart apache
    service: 
      name: httpd 
      state: restarted
```

## Exemple de Playbook avec Rôle Apache2
```
---
- hosts: all
  become: yes
  roles:
    - apache2
```

Pour créer un rôle Ansible, utilisez ansible-galaxy init <role_name>. Cela génère une structure standardisée.

Créons un rôle apache2
```
 ansible-galaxy init apache2
 tree apache2
```
Accédez au répertoire `apache` et éditez votre rôle:

Configuration du Rôle Apache2

1. Tasks (tasks/main.yml)
```
---
- import_tasks: install.yml
- import_tasks: configure.yml  
- import_tasks: service.yml

```

tasks/install.yml :

```
---
- name: Install apache2 Package
  apt:
    name: apache2
    state: latest
  notify: restart apache2
```

tasks/configure.yml :

```
---
- name: Copy apache2 configuration file
  copy:
    src: apache2.conf
    dest: /etc/apache2/apache2.conf
  notify: restart apache2

- name: Copy index.html
  copy:
    src: files/index.html
    dest: /var/www/html/
  notify: restart apache2
```

tasks/service.yml :

```
---
- name: Start and Enable apache2 service
  systemd:
    name: apache2
    state: started
    enabled: yes
```

2. Handlers (handlers/main.yml)

```
---
- name: restart apache2
  systemd:
    name: apache2
    state: restarted
```

3. Fichiers Requis

Copiez apache2.conf et index.html dans apache2/files/.



### Installer un rôle depuis Ansible Galaxy

Ansible Roles jouent un rôle crucial dans le partage de code avec
d'autres utilisateurs de la communauté Ansible à l'aide de la
plate-forme Ansible Galaxy. Dans Ansible Galaxy , vous obtenez des
milliers de rôles effectuant différentes tâches telles que
l'installation de serveurs Web et de bases de données, d'outils de
monitoring , etc.

Ansible Galaxy est une base de données ou un référentiel de rôles
Ansible que vous pouvez exploiter dans vos playbooks.

Pour rechercher un rôle dans Ansible Galaxy , exécutez simplement la
commande.
```
$ ansible-galaxy search <role>
```
Par exemple, pour rechercher un rôle nommé mysql, exécutez.
```
$ ansible-galaxy search mysql
```


Comme vous pouvez le voir, il existe des centaines de rôles qui
correspondent au mot-clé *mysql* . Cependant, tous les rôles
n'effectuent pas ce que vous souhaitez, il est donc recommandé de lire
attentivement les instructions.

Pour collecter plus d'informations sur un rôle, exécutez simplement la
commande Ansible:
```
$ ansible-galaxy info geerlingguy.mysql
```
Dans notre exemple, nous allons installer le rôle
5KYDEV0P5.skydevops-mysql .
```shell
$ ansible-galaxy install 5KYDEV0P5.skydevops-mysql
```
``` output
- downloading role 'mysql', owned by geerlingguy
- downloading role from https://github.com/geerlingguy/ansible-role-mysql/archive/3.3.2.tar.gz
- extracting geerlingguy.mysql to /home/vagrant/.ansible/roles/geerlingguy.mysql
- geerlingguy.mysql (3.3.2) was installed successfully
``` 

Le rôle est téléchargé et extrait dans le répertoire des rôles par
défaut de l'utilisateur `~/.ansible/roles/`.

Le rôle peut ensuite être appelé dans un playbook, par exemple:
```yaml
---
- name: Install MySQL server
  hosts: dbservers
  gather_facts: yes
  roles:
  - role: 5KYDEV0P5.skydevops-mysql
    become: yes
```
Vous pouvez maintenant exécuter en toute sécurité le playbook Ansible
comme indiqué.
```
$ ansible-playbook install_mysql.yml
```
De plus, vous pouvez visiter Ansible Galaxy via votre navigateur Web et
rechercher manuellement des rôles pour effectuer diverses tâches, comme
indiqué par le tableau de bord.


Par exemple, pour rechercher un rôle tel que elasticsearch, cliquez sur
l'option « Monitoring » et recherchez le rôle comme indiqué.


Ansible Galaxy permet aux utilisateurs d'installer plus facilement les
meilleurs rôles en répertoriant les rôles les plus populaires et les
plus téléchargés. Pour obtenir plus d'informations sur un rôle
spécifique, cliquez simplement dessus.


Vérifiez les informations de rôle sur Ansible Galaxy

Dans un playbook, vous pouvez également spécifier plusieurs rôles, par
exemple.
```yaml
---

- name: Install MySQL server
  hosts: dbservers
  gather_facts: yes
  become: yes
  roles:
  - geerlingguy.mysql
  - geerlingguy.phpmyadmin
```
Pour lister les rôles installés, exécutez simplement.
```
$ ansible-galaxy list
```

## Inclure un Ansible Role dans une tâche
Vous pouvez réutiliser des rôles de manière dynamique n'importe où dans la section 
des tâches d'un play en utilisant `include_role`. 
Alors que les rôles ajoutés dans la section **roles:** s'exécutent avant toute autre tâche
dans un playbook, les rôles inclus s'exécutent dans l'ordre dans lequel ils sont définis. 
S'il existe d'autres tâches avant une tâche `include_role`, 
les autres tâches seront exécutées en premier.

Pour inclure un rôle:

```YAML
---
- hosts: webservers
  tasks:
    - name: Print a message
      debug:
        msg: "this task runs before the example role"

    - name: Include the example role
      include_role:
        name: example

    - name: Print a message
      debug:
        msg: "this task runs after the example role"

```
Vous pouvez inclure conditionnellement un rôle:
```yaml
---
- hosts: webservers
  tasks:
    - name: Include the some_role role
      include_role:
        name: some_role
      when: "ansible_facts['os_family'] == 'RedHat'"
```


##Conclusion

Les rôles facilitent la réutilisation et le partage des playbooks
Ansible. De cette façon, ils font gagner beaucoup de temps à
l'utilisateur en essayant d'écrire beaucoup de code redondant qui aurait
été utilisé dans d'autres tâches d'administration système.


---
## Application
***Décomposition en Rôles : 
Transformez le playbook de l'application flask en une structure de rôles 
Ansible distincts. 
Créez des rôles distincts pour l'installation de paquets, 
le clonage du référentiel Git, la configuration du fichier, 
l'installation des dépendances, la copie du fichier de service systemd, 
et le redémarrage du service.



