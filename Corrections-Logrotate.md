# Corrections — Fils Rouges Ansible (Lab 4 à Lab 8)

Ce document regroupe les corrections complètes de deux fils rouges pédagogiques :

- **Série B** : Rotation de logs applicatifs (Logrotate)
- **Série C** : Gestion centralisée des utilisateurs et accès SSH

---

# Série B — Rotation de Logs Applicatifs

## Correction — Lab 4

```yaml
---
- name: Mise en place de la rotation des logs
  hosts: webservers
  become: yes
  tasks:
    - name: Installer logrotate
      apt:
        name: logrotate
        state: present

    - name: Créer le répertoire de logs applicatifs
      file:
        path: /var/log/myapp
        state: directory
        owner: www-data
        group: www-data
        mode: '0755'

    - name: Créer un fichier de log de test
      copy:
        content: "log de démarrage - {{ ansible_date_time.iso8601 }}\n"
        dest: /var/log/myapp/app.log
        mode: '0644'

    - name: Déployer la configuration logrotate
      copy:
        content: |
          /var/log/myapp/*.log {
              daily
              rotate 7
              compress
              missingok
              notifempty
          }
        dest: /etc/logrotate.d/myapp
        mode: '0644'
```

**Vérification :**
```bash
ansible-playbook log-rotation.yml
ansible webservers -a "logrotate --debug /etc/logrotate.d/myapp"
```

---

## Correction — Lab 5

`group_vars/webservers.yml` :
```yaml
log_app_name: myapp
log_retention_days: 7
log_compress: true
log_env: dev
```

`host_vars/app2.yml` :
```yaml
log_retention_days: 30
log_env: prod
```

`log-rotation.yml` (mis à jour) :
```yaml
---
- name: Rotation des logs (avec variables)
  hosts: webservers
  become: yes
  vars_files:
    - group_vars/webservers.yml
  tasks:
    - name: Installer logrotate
      apt:
        name: logrotate
        state: present

    - name: Créer le répertoire de logs applicatifs
      file:
        path: "/var/log/{{ log_app_name }}"
        state: directory
        owner: www-data
        group: www-data
        mode: '0755'

    - name: Déployer la configuration logrotate
      copy:
        content: |
          /var/log/{{ log_app_name }}/*.log {
              daily
              rotate {{ log_retention_days }}
              {{ 'compress' if log_compress else '' }}
              missingok
              notifempty
          }
        dest: "/etc/logrotate.d/{{ log_app_name }}"
```

**Vérification :**
```bash
ansible app1 -a "cat /etc/logrotate.d/myapp"   # rotate 7
ansible app2 -a "cat /etc/logrotate.d/myapp"   # rotate 30
```

---

## Correction — Lab 6

`group_vars/webservers.yml` (mis à jour) :
```yaml
log_compress: true
log_env: dev

apps:
  - name: myapp
    retention: 7
  - name: api-service
    retention: 14
  - name: worker
    retention: 3
```

`log-rotation.yml` :
```yaml
---
- name: Rotation des logs (multi-applications)
  hosts: webservers
  become: yes
  vars_files:
    - group_vars/webservers.yml
  tasks:
    - name: Installer logrotate
      apt:
        name: logrotate
        state: present

    - name: Créer les répertoires de logs
      file:
        path: "/var/log/{{ item.name }}"
        state: directory
        owner: www-data
        mode: '0755'
      loop: "{{ apps }}"

    - name: Déployer les configurations logrotate
      copy:
        content: |
          /var/log/{{ item.name }}/*.log {
              daily
              rotate {{ item.retention }}
              compress
              missingok
              notifempty
          }
        dest: "/etc/logrotate.d/{{ item.name }}"
      loop: "{{ apps }}"
      notify: tester logrotate

    - name: Alerte rétention faible en production
      debug:
        msg: "⚠️ Rétention de {{ item.retention }} jours jugée faible pour {{ item.name }} en prod"
      loop: "{{ apps }}"
      when: log_env == "prod" and item.retention < 14

  handlers:
    - name: tester logrotate
      command: "logrotate --debug /etc/logrotate.d/{{ item.item.name }}"
      loop: "{{ apps }}"
      loop_control:
        loop_var: item
      changed_when: false
```

> ⚠️ Point d'attention pour les apprenants : dans un handler déclenché par une
> tâche en boucle, `item` n'est plus automatiquement disponible — il faut soit
> boucler explicitement sur `apps` dans le handler (comme ci-dessus), soit
> utiliser `listen` avec un nom générique et ne pas dépendre de `item`.

**Vérification idempotence :**
```bash
ansible-playbook log-rotation.yml   # 1ère exécution : changed
ansible-playbook log-rotation.yml   # 2ème exécution : ok partout
```

---

## Correction — Lab 7

`templates/logrotate.conf.j2` :
```jinja2
/var/log/{{ item.name }}/*.log {
    daily
    rotate {{ item.retention }}
    {% if log_compress %}compress
    delaycompress{% endif %}
    missingok
    notifempty
   
}
```

`log-rotation.yml` (tâche mise à jour) :
```yaml
    - name: Générer les configurations logrotate depuis le template
      template:
        src: logrotate.conf.j2
        dest: "/etc/logrotate.d/{{ item.name }}"
      loop: "{{ apps }}"
      notify: tester logrotate
```

**Vérification :**
```bash
ansible webservers -a "logrotate --force /etc/logrotate.d/myapp"
ansible webservers -a "cat /var/log/myapp/rotation.history"
```

---

## Correction — Lab 8

Structure du rôle :
```
roles/log-rotation/
├── defaults/main.yml
├── handlers/main.yml
├── tasks/
│   ├── main.yml
│   ├── install.yml
│   └── configure.yml
└── templates/
    └── logrotate.conf.j2
```

`roles/log-rotation/defaults/main.yml` :
```yaml
log_compress: true
log_env: dev
apps:
  - name: myapp
    retention: 7
  - name: api-service
    retention: 14
  - name: worker
    retention: 3
```

`roles/log-rotation/handlers/main.yml` :
```yaml
---
- name: tester logrotate
  command: "logrotate --debug /etc/logrotate.d/{{ item.name }}"
  loop: "{{ apps }}"
  changed_when: false
```

`roles/log-rotation/tasks/main.yml` :
```yaml
---
- import_tasks: install.yml
- import_tasks: configure.yml
```

`roles/log-rotation/tasks/install.yml` :
```yaml
---
- name: Installer logrotate
  apt:
    name: logrotate
    state: present

- name: Créer les répertoires de logs
  file:
    path: "/var/log/{{ item.name }}"
    state: directory
    owner: www-data
    mode: '0755'
  loop: "{{ apps }}"
```

`roles/log-rotation/tasks/configure.yml` :
```yaml
---
- name: Générer les configurations logrotate depuis le template
  template:
    src: logrotate.conf.j2
    dest: "/etc/logrotate.d/{{ item.name }}"
  loop: "{{ apps }}"
  notify: tester logrotate

- name: Alerte rétention faible en production
  debug:
    msg: "⚠️ Rétention de {{ item.retention }} jours jugée faible pour {{ item.name }} en prod"
  loop: "{{ apps }}"
  when: log_env == "prod" and item.retention < 14
```

Playbook final `site_logrotate.yml` :
```yaml
---
- name: Rotation des logs via rôle
  hosts: webservers
  become: yes
  roles:
    - log-rotation
```

**Vérification finale :**
```bash
ansible-playbook site_logrotate.yml -v
ansible webservers -a "ls /etc/logrotate.d/"
```

