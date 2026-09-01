
** Lab 6 – Contrôle d’Exécution
===============================
# Les boucles

Souvent, vous voudrez faire plusieurs choses en une seule tâche, comme
-   Créer beaucoup d'utilisateurs
-   Répéter une requête jusqu'à ce qu'un certain résultat soit atteint.

Ansible prend en charge les boucles pour itérer sur un ensemble de
valeurs et évite aux administrateurs d'écrire des tâches répétitives
utilisant les mêmes modules.

Ansible prend en charge trois principaux types de boucles: les boucles
simples, la liste de hachages et les boucles imbriquées. Dans cet
atelier, nous examinerons rapidement les deux premiers types.



### Schéma de Contrôle d'Exécution

```
┌─────────────────────────┐
│   Tâche Ansible        │
├─────────────────────────┤
│  1. Vérifier condition  │ ─── when
│     (si OK, continuer)  │
├─────────────────────────┤
│  2. Boucler ou Exécuter │ ─── loop/until
├─────────────────────────┤
│  3. Exécuter le module  │
├─────────────────────────┤
│  4. Récupérer résultat  │ ─── register
├─────────────────────────┤
│  5. Vérifier statut     │ ─── failed_when
├─────────────────────────┤
│  6. Appeler handlers    │ ─── notify/handlers
└─────────────────────────┘
```

## 🎯 Conditions avec When

### Syntaxe Basique

```yaml
---
- name: Conditions simples
  hosts: webservers
  tasks:
    - name: Installer nginx sur RedHat
      yum:
        name: nginx
        state: present
      when: ansible_os_family == "RedHat"
    
    - name: Installer nginx sur Debian
      apt:
        name: nginx
        state: present
      when: ansible_os_family == "Debian"
    
    - name: Tâche si variable existe
      debug:
        msg: "Variable définie"
      when: my_variable is defined
    
    - name: Tâche si variable n'existe pas
      debug:
        msg: "Variable non définie"
      when: my_variable is not defined
```

### Conditions Complexes

```yaml
tasks:
    - name: Condition avec et (and)
      debug:
        msg: "Condition vraie"
      when: 
        - ansible_os_family == "RedHat"
        - ansible_processor_vcpus >= 2
    
    - name: Condition avec ou (or)
      debug:
        msg: "L'un ou l'autre est vrai"
      when: 
        - ansible_os_family == "RedHat" or ansible_os_family == "Debian"
    
    - name: Condition avec parenthèses
      debug:
        msg: "Condition complexe"
      when: (ansible_distribution == "CentOS" and ansible_processor_vcpus >= 2) or environment == "production"
    
    - name: Condition avec in
      debug:
        msg: "Distribution supportée"
      when: ansible_distribution in ["CentOS", "Ubuntu", "Debian"]
    
    - name: Condition avec tests de fichier
      debug:
        msg: "Fichier existe"
      when: ansible_facts.stat.exists 
```

### Tests Jinja2 Courants

```yaml
tasks:
    - name: Tests de type
      debug:
        msg: |
          String: {{ my_var is string }}
          Nombre: {{ my_var is number }}
          Liste: {{ my_var is iterable }}
          Booléen: {{ my_var is boolean }}
    
    - name: Tests de contenu
      debug:
        msg: |
          Vide: {{ my_var is empty }}
          Différent: {{ my_var is not empty }}
          Pair: {{ 4 is even }}
          Impair: {{ 5 is odd }}
          Réseau: {{ ip_address is network }}
    
    - name: Tests de récapitulatif
      debug:
        msg: "Skipped: {{ item is skipped }}"
      loop: "{{ some_result.results }}"
```

## 🔄 Boucles

### Loop Basique

```yaml
---
- name: Démonstration des boucles
  hosts: app1
  tasks:
    - name: Boucle simple sur une liste
      debug:
        msg: "Package: {{ item }}"
      loop:
        - nginx
        - php
        - curl
        - wget
    
    - name: Boucle sur une variable
      debug:
        msg: "Nombre: {{ item }}"
      loop: "{{ [1, 2, 3, 4, 5] }}"
    
    - name: Boucle avec index
      debug:
        msg: "Index {{ ansible_loop.index }} : {{ item }}"
      loop: "{{ packages }}"
      loop_control:
        extended: yes
```


## Boucles simples
Les boucles simples sont une liste d'éléments sur lesquels Ansible
itère. Ils sont définis en fournissant une liste d'éléments au mot-clé
**loop**. Créez le Playbook suivant `loop1.yml` et exécutez-le:
```yaml
---
- name: Loop demo
  hosts: app1
  tasks:
  - name: Check if service is started
    service:
      name: "{{ item }}"
      state: "{{ etat }}"
    loop:
    - httpd
    - sshd
```
La liste des éléments à parcourir peut également être fournie sous forme
de liste dans la section **vars** ou dans un fichier. Dans cet exemple, le
tableau est appelé check_services. Créez ce Playbook en tant que
`loop2.yml` et exécutez-le, la sortie doit être la même:
```yaml
---
- name: Loop demo
  hosts: app1
  vars:
    check_services:
    - httpd
    - sshd
  tasks:
  - name: Check if service is started
    service:
      name: "{{ item }}"
      state: started
    loop: "{{ check_services }}"
```

### Les Hashs
Vous pouvez également parcourir une liste de **hashs** pour des données plus
complexes. Le Playbook suivant montre comment une liste de **hashs** avec
des paires clé-valeur est transmise au module **user**. Créez
`loop3.yml` et exécutez-le:
```yaml
---
- name: Hash demo
  hosts: app1
  become: yes
  tasks:
  - name: Create users from hash
    user:
      name: "{{ joe }}"
      state: present
      groups: "{{ root }}"
    loop:
    - { name: 'jane', groups: 'wheel' }
    - { name: 'joe', groups: 'root' }
	
```

# Conditionnels Ansible

Ansible peut utiliser des conditions pour exécuter des tâches lorsque
certaines conditions sont remplies.

Pour implémenter une condition, l'instruction **when** doit être utilisée,
suivie de la condition à tester. La condition est exprimée en utilisant
l'un des opérateurs disponibles comme par exemple pour comparaison:

  `==`      Compare deux objets pour l'égalité.
  `!=`     Compare deux objets pour l'inégalité.
  `>`       True si le côté gauche est plus grand que le côté droit.
  `>=`     True si le côté gauche est supérieur ou égal au côté droit.
  `<`       True si le côté gauche est plus bas que le côté droit.
  `<=`     True si le côté gauche est inférieur ou égal au côté droit.

### Utiliser des conditions

Par exemple, vous souhaitez installer un serveur FTP, mais uniquement
sur des hôtes qui ne sont pas en production car le protocole n'est pas
sécurisé.

En tant qu'utilisateur, créez le Playbook `ftpserver.yml`, exécutez-le et
examinez la sortie:
```yaml
---
- name: Install insecure FTP server as long as not in production
  hosts: webservers
  become: yes
  tasks:
  - name: Install FTP server if not in production
    yum:
      name: vsftpd
      state: latest
    when: stage == "prod"
```
*** ansible-playbook ftpserver.yml -e "stage=dev"
Résultat attendu: la tâche est ignorée sur *app1* car la variable *stage*
est définie sur prod:

> [...]
> TASK [Install FTP server if not in production]
> \************************
> skipping: [app1]
> changed: [app2]
> [...]

### Challenge Lab: Fact en conditionnel

Essayons autre chose. Vous avez probablement remarqué *app1* et *db* ont différentes 
quantités de RAM.
Sinon, regardez à nouveau les facts:

> $ ansible all -m setup -a 'filter=ansible_*_mb'

Écrivez un Playbook `mariadb.yml` qui installe *MariaDB* mais seulement si
l'hôte a plus de, disons, ***400Mo*** de RAM.

-   Trouvez le fact pour `memfree` en Mo (regardez la sortie de la
    commande ad-hoc et n'hésitez pas à utiliser "grep").

-   Utilisez ce Playbook comme modèle et créez l'instruction **when** en
    remplaçant les espaces réservés en majuscules :
```
---
- name: MariaDB server installation
  hosts: all
  become: yes
  gather_facts: yes
  tasks:
  - name: Install latest MariaDB server when host RAM greater than 900 MB
    yum:
      name: mariadb-server
      state: latest
    when: FACT > 400
```
Exécutez le Playbook. En conséquence, la tâche d'installation doit être
ignorée sur quelques noeuds qui ne satisfont pas la condition.

#### Solution
```
---
- name: MariaDB server installation
  hosts: all
  gather_facts: yes
  become: yes
  tasks:
  - name: Install latest MariaDB server when host RAM greater than 700 MB
    yum:
      name: mariadb-server
      state: latest
    when: ansible_memfree_mb > 700
```

# Ansible Handlers

Parfois, lorsqu'une tâche modifie le système, une autre tâche devra être
exécutée. Par exemple, une modification du fichier de configuration d'un
service peut alors exiger que le service soit rechargé afin que la configuration
modifiée prenne effet.

Ici les **handlers** de Ansible entrent en jeu. Les **handlers** peuvent
être considérés comme des tâches inactives qui ne sont déclenchées que
lorsqu'elles sont explicitement appelées à l'aide de l'instruction
"***notify***".

À titre d'exemple, écrivons un Playbook qui:

-   gère le fichier de configuration d'Apache ***httpd.conf*** sur toutes
    les hôtes du groupe *webserver*
-   redémarre Apache lorsque le fichier est changé

Nous avons d'abord besoin du fichier que Ansible va déployer:
```
[master]$ ansible -m fetch -a "src=/etc/apache2/ports.conf dest=/home/vagrant/ansible/ flat=yes" all
```
Créons maintenant le Playbook apache2_conf.yml:
```yaml
---
- name: manage ports.conf
  hosts: webservers
  become: yes
  tasks:
  - name: Copy Apache configuration file
    copy:
      src: ports.conf
      dest: /etc/apache2/ports.conf
    notify:
    - restart_apache
  handlers:
  - name: restart_apache
    service:
      name: httpd
      state: restarted
```
Alors, quoi de neuf ici?

-   La section ***notify*** appelle le **handler** uniquement lorsque la
     tâche de copie a modifié le fichier.

-   La section ***handlers*** définit une tâche qui n'est exécutée que
     sur notification.

Exécutez le Playbook. Nous n'avons encore rien changé dans le fichier
donc il ne devrait y avoir aucune ligne *changed* dans la sortie et bien
sûr le handler ne se déclenche pas.

-   Maintenant, changez la ligne ***Listen 80*** dans httpd.conf en:

> Listen 8080

-   Exécutez à nouveau le Playbook. Maintenant, la sortie de Ansible
     devrait être beaucoup plus intéressante:

    -   port.conf doit être copié

    -   Le ***handler*** doit avoir redémarré Apache

Modifier le port dans ports.conf ne suffit pas toujours. 
Vous devez également mettre à jour la déclaration de votre VirtualHost 
(généralement dans /etc/apache2/sites-available/000-default.conf) 
pour qu'elle corresponde :

Exemple : <VirtualHost *:8080>


Apache devrait maintenant écouter sur le port 8080. Assez facile à
vérifier:
```SHELL
[master]$ curl 192.168.60.5
curl: (7) Failed connect to 192.168.60.5:80; Connection refused
[master]$ curl 192.168.60.5:8080
<body>
<h1>This is a production webserver, take care!</h1>
</body>
```

---
## Application 1 :

Stockez un fichier avec le contenu ci-dessus:
users:
- username: adam
  uid: 2000
- username: greg
  uid: 2001
- username: robby
  uid: 3001
- username: xenia
  uid: 3002

Créez un playbook users.yml qui répond aux exigences suivantes :

Crée des utilisateurs dont l'UID commence par 2 dans le groupe d'hôtes webservers.
Crée des utilisateurs dont l'UID commence par 3 dans le groupe d'hôtes dbservers. 
Les utilisateurs doivent faire partie du groupe supplémentaire wheel.
Le shell des utilisateurs doit être réglé sur /bin/bash.

```

---
- name: Gestion des utilisateurs par UID et groupes d'hôtes
  hosts: webservers, dbservers
  become: true
  vars_files:
    - users_list.yml

  tasks:
    # Cas 1 : webservers (UID commençant par 2)
    - name: Créer les utilisateurs (UID 2xxx) sur webservers
      user:
        name: "{{ item.username }}"
        uid: "{{ item.uid }}"
        groups: wheel
        append: yes
        shell: /bin/bash
        state: present
      loop: "{{ users }}"
      when: 
        - "'webservers' in group_names"
        - "item.uid | string | syntax_check_uid('2')"
      # Note : On utilise une logique simple ci-dessous si syntax_check est complexe :
      # when: "'webservers' in group_names and item.uid >= 2000 and item.uid < 3000"

    # Cas 2 : dbservers (UID commençant par 3)
    - name: Créer les utilisateurs (UID 3xxx) sur dbservers
      user:
        name: "{{ item.username }}"
        uid: "{{ item.uid }}"
        groups: wheel
        append: yes
        shell: /bin/bash
        state: present
      loop: "{{ users }}"
      when: 
        - "'dbservers' in group_names"
        - "item.uid >= 3000 and item.uid < 4000"
```
