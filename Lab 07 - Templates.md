
** Lab 7 – Ansible Templates
===========================
Ansible utilise Jinja2, un moteur de création de modèles pour les
frameworks Python, utilisé pour générer du contenu ou des expressions
dynamiques. La création de modèles est extrêmement utile lors de la
création de fichiers de configuration personnalisés pour plusieurs
serveurs, mais uniques pour chacun d'entre eux.

### Syntaxe Jinja2

```
{{ variable }}          # Variables
{# commentaire #}       # Commentaires
{% if condition %}      # Conditions
{% for item in list %}  # Boucles
{{ variable | filter }} # Filters
```

Exemple 1
---------

Rappelez-vous du lab des variables, où nous avons distribué des fichiers
index.html selon la variable *stage.* Le même objectif peut être atteint
avec les templates J2.

Cette fois, nous allons distribuer le même fichier index.html, qui doit
changer de contenu selon le système d’exploitation.

Créons donc un playbook *test_server.yml* comme indiqué:
```yaml
---
- hosts: webservers
  gather_facts: yes
  become: yes
  tasks:
  - name: Install index.html
    template:
      src: index.html.j2
      dest: /var/www/html/index.html
      mode: 0644
```
Notre template Jinja2 de fichier `index.html` est **index.html.j2** qui
sera pushé vers le fichier `index.html` sur chaque serveur Web. N'oubliez
jamais de mettre l'extension ***.j2*** à la fin pour indiquer qu'il s'agit
d'un fichier *Jinja2*.

Créons maintenant le fichier modèle `index.html.j2`.
```Jinja2
<html>
  <center>
    <h1> The hostname of this webserver is {{ ansible_fqdn }}</h1>
    <h3> It is running on {{ ansible_os_family }} system </h3>
    <p> this is a {{ stage }} server </p>
  </center>
</html>
```

Ce modèle est un fichier HTML simple dans lequel *ansible_hostname* et
*ansible_os_family* sont des variables qui seront remplacées par les
noms d'hôte et les systèmes d'exploitation respectifs des serveurs Web.
```
$ ansible-playbook test_server.yml
```
Rechargeons maintenant les pages Web pour les serveurs Web.

### Filtres

Parfois, vous pouvez décider de remplacer la valeur d'une variable par
une chaîne qui apparaît d'une certaine manière.

#### Exemple 1: faire apparaître les chaînes en majuscules / minuscules

Par exemple, dans l'exemple précédent, nous pouvons décider de faire
apparaître les variables Ansible en majuscules. Pour ce faire, ajoutez
la valeur upper à la variable. De cette façon, la valeur de la variable
est convertie au format majuscule.

##### {{ ansible_fqdn | upper }} => APP1.DEV

##### {{ ansible_os_family | upper }} => REDHAT
```html
<html>
  <center>
    <h1> The hostname of this webserver is {{ ansible_fqdn | upper }}</h1>
    <h3> It is running on {{ ansible_os_family | upper }} system </h3>
    <p> this is a {{ stage | upper }} server </p>
  </center>
</html>
```

### Exemple 2: remplacer une chaîne par une autre

En outre, vous pouvez remplacer une chaîne par une autre.

Par exemple:

Le titre du film est {{ movie_name }} => Le titre du film est Ring.

Pour remplacer la sortie par une autre chaîne, utilisez l'argument
replace comme indiqué:

Le titre du film est {{ movie_name | replace (“Ring“,”Heist”) }} =>
Le titre du film est Heist .

Changez le modèle J2 pour remplacer prod par Production et dev par Development.
```
this is a {{ stage | replace (“prod“,”Production”) | replace (“dev“,”Development”) }} server
```


### Exemple 2

Créer `templates/nginx.conf.j2` :

```nginx
# Configuration Nginx générée par Ansible
# Host: {{ ansible_hostname }}
# Date: {{ ansible_date_time.iso8601 }}

user {{ nginx_user }};
worker_processes {{ nginx_worker_processes }};
error_log /var/log/nginx/error.log;
pid /var/run/nginx.pid;

events {
    worker_connections {{ nginx_worker_connections }};
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;

    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout {{ nginx_keepalive_timeout }};

    gzip on;
    gzip_types text/plain text/css application/json;

    server {
        listen {{ http_port }} default_server;
        listen [::]:{{ http_port }} default_server;
        server_name {{ server_name }};

        root {{ web_root }};

        location / {
            try_files $uri $uri/ =404;
        }
    }
}
```

### Utiliser le Template dans un Playbook

```yaml
---
- name: Générer configuration Nginx
  hosts: webservers
  
  vars:
    nginx_user: nginx
    nginx_worker_processes: 4
    nginx_worker_connections: 1024
    nginx_keepalive_timeout: 65
    http_port: 80
    server_name: "{{ inventory_hostname }}"
    web_root: /var/www/html
  
  tasks:
    - name: Générer le fichier nginx.conf
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
        owner: root
        group: root
        mode: '0644'
      notify: reload nginx
  
  handlers:
    - name: reload nginx
      service:
        name: nginx
        state: reloaded
```

---
#Application 1
Créez un nouveau dossier nommé templates et préparez-y un modèle 
qui sera utilisé plus tard pour générer un fichier hosts pour chaque 
nœud à partir de l'inventaire. 
L'idée générale est décrite ci-dessous :
Nommez le fichier hosts.j2 :
```
127.0.0.1 localhost <nom_court_local> <fqdn_local>
127.0.1.1 localhost
<ip_address_host1> <nom_court_host1> <fqdn_host1>
<ip_address_host2> <nom_court_host2> <fqdn_host2>
```


Créez un playbook nommé hosts.yml qui respecte les exigences suivantes :

*Il s'exécute sur tous les hôtes.

*Il utilise un modèle pour remplir le fichier hosts.j2 créé précédemment pour tous les hôtes de l'inventaire.

*Après l'exécution du playbook, il devrait être possible d'atteindre 

*un autre nœud depuis n'importe quel nœud en utilisant l'IP, le nom court ou le FQDN.
