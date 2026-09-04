---

# Série C — Gestion Centralisée des Utilisateurs et Accès SSH

## Génération préalable des clés SSH

```bash
mkdir -p keys && cd keys
for user in deploy alice bob carla; do
  ssh-keygen -t ed25519 -f ${user}_key -N "" -C "${user}@formation"
done
mv keys/*.pub .
```

- `<user>_key` = clé **privée** (à conserver, jamais commitée)
- `<user>_key.pub` = clé **publique** (utilisée par le playbook)

---

## Correction — Lab 4

`users.yml` :
```yaml
---
- name: Gestion des utilisateurs et accès SSH
  hosts: webservers
  become: yes
  tasks:
    - name: Créer l'utilisateur deploy
      user:
        name: deploy
        shell: /bin/bash
        state: present

    - name: Ajouter la clé publique SSH de deploy
      authorized_key:
        user: deploy
        state: present
        key: "{{ lookup('file', 'deploy_key.pub') }}"

    - name: Créer le répertoire de travail de deploy
      file:
        path: /opt/deploy
        state: directory
        owner: deploy
        group: deploy
        mode: '0750'
```

**Vérification :**
```bash
ssh -i deploy_key deploy@192.168.60.4 "whoami && ls -la /opt/deploy"
```

---

## Correction — Lab 5

`group_vars/webservers.yml` :
```yaml
deploy_user: deploy
deploy_home: /opt/deploy
deploy_shell: /bin/bash
```

`host_vars/app2.yml` :
```yaml
deploy_user: deploy-app2
```

`users.yml` (mis à jour) :
```yaml
---
- name: Gestion des utilisateurs et accès SSH (avec variables)
  hosts: webservers
  become: yes
  vars_files:
    - group_vars/webservers.yml
  tasks:
    - name: Créer l'utilisateur
      user:
        name: "{{ deploy_user }}"
        shell: "{{ deploy_shell }}"
        state: present

    - name: Ajouter la clé publique SSH
      authorized_key:
        user: "{{ deploy_user }}"
        state: present
        key: "{{ lookup('file', 'deploy_key.pub') }}"

    - name: Créer le répertoire de travail
      file:
        path: "{{ deploy_home }}"
        state: directory
        owner: "{{ deploy_user }}"
        group: "{{ deploy_user }}"
        mode: '0750'
```

**Vérification :**
```bash
ansible app1 -a "id deploy"
ansible app2 -a "id deploy-app2"
```

---

## Correction — Lab 6

`group_vars/webservers.yml` (mis à jour) :
```yaml
deploy_home: /opt/deploy
deploy_shell: /bin/bash

users:
  - name: alice
    role: admin
  - name: bob
    role: dev
  - name: carla
    role: admin
```

`users.yml` :
```yaml
---
- name: Gestion des utilisateurs (boucles, conditions, handler)
  hosts: webservers
  become: yes
  vars_files:
    - group_vars/webservers.yml
  tasks:
    - name: Créer les utilisateurs
      user:
        name: "{{ item.name }}"
        shell: "{{ deploy_shell }}"
        state: present
      loop: "{{ users }}"

    - name: Déployer les clés SSH des utilisateurs
      authorized_key:
        user: "{{ item.name }}"
        state: present
        key: "{{ lookup('file', item.name + '_key.pub') }}"
      loop: "{{ users }}"

    - name: Ajouter les admins au groupe sudo
      user:
        name: "{{ item.name }}"
        groups: sudo
        append: yes
      loop: "{{ users }}"
      when: item.role == "admin"

    - name: Interdire le login root par mot de passe
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?PermitRootLogin'
        line: 'PermitRootLogin prohibit-password'
      notify: redémarrer sshd

  handlers:
    - name: redémarrer sshd
      service:
        name: sshd
        state: restarted
```

**Vérification idempotence :**
```bash
ansible-playbook users.yml   # 1ère exécution : changed
ansible-playbook users.yml   # 2ème exécution : ok partout, handler non déclenché
```

---

## Correction — Lab 7

`templates/sshd_custom.conf.j2` :
```jinja2
# Généré par Ansible le {{ ansible_date_time.date }} sur {{ ansible_hostname }}
# Utilisateurs autorisés à se connecter :
{% for u in users %}
# - {{ u.name }} ({{ u.role }})
{% endfor %}
AllowUsers {{ users | map(attribute='name') | join(' ') }}
```

`templates/motd.j2` :
```jinja2
Bienvenue sur {{ ansible_hostname }}
Système : {{ ansible_distribution }} {{ ansible_distribution_version }}
Administrateurs de ce serveur :
{% for u in users %}
{% if u.role == "admin" %}
  - {{ u.name }}
{% endif %}
{% endfor %}
```

`users.yml` (tâches ajoutées) :
```yaml
    - name: Générer la configuration SSH personnalisée
      template:
        src: sshd_custom.conf.j2
        dest: /etc/ssh/sshd_config.d/custom.conf
        mode: '0644'
      notify: redémarrer sshd

    - name: Générer le message d'accueil (MOTD)
      template:
        src: motd.j2
        dest: /etc/motd
        mode: '0644'
```

**Vérification :**
```bash
ssh -i alice_key alice@192.168.60.4
# Le MOTD affiché doit lister alice et carla (admins), pas bob
```

---

## Correction — Lab 8

Structure du rôle :
```
roles/user-management/
├── defaults/main.yml
├── handlers/main.yml
├── tasks/
│   ├── main.yml
│   ├── create_users.yml
│   └── configure_ssh.yml
└── templates/
    ├── sshd_custom.conf.j2
    └── motd.j2
```

`roles/user-management/defaults/main.yml` :
```yaml
deploy_shell: /bin/bash
users:
  - name: alice
    role: admin
  - name: bob
    role: dev
  - name: carla
    role: admin
```

`roles/user-management/handlers/main.yml` :
```yaml
---
- name: redémarrer sshd
  service:
    name: sshd
    state: restarted
```

`roles/user-management/tasks/main.yml` :
```yaml
---
- import_tasks: create_users.yml
- import_tasks: configure_ssh.yml
```

`roles/user-management/tasks/create_users.yml` :
```yaml
---
- name: Créer les utilisateurs
  user:
    name: "{{ item.name }}"
    shell: "{{ deploy_shell }}"
    state: present
  loop: "{{ users }}"

- name: Déployer les clés SSH des utilisateurs
  authorized_key:
    user: "{{ item.name }}"
    state: present
    key: "{{ lookup('file', item.name + '_key.pub') }}"
  loop: "{{ users }}"

- name: Ajouter les admins au groupe sudo
  user:
    name: "{{ item.name }}"
    groups: sudo
    append: yes
  loop: "{{ users }}"
  when: item.role == "admin"
```

`roles/user-management/tasks/configure_ssh.yml` :
```yaml
---
- name: Interdire le login root par mot de passe
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^#?PermitRootLogin'
    line: 'PermitRootLogin prohibit-password'
  notify: redémarrer sshd

- name: Générer la configuration SSH personnalisée
  template:
    src: sshd_custom.conf.j2
    dest: /etc/ssh/sshd_config.d/custom.conf
    mode: '0644'
  notify: redémarrer sshd

- name: Générer le message d'accueil (MOTD)
  template:
    src: motd.j2
    dest: /etc/motd
    mode: '0644'
```

Playbook final `site_users.yml` :
```yaml
---
- name: Gestion des utilisateurs via rôle
  hosts: webservers
  become: yes
  roles:
    - user-management
```

**Vérification finale :**
```bash
ansible-playbook site_users.yml -v
ansible webservers -a "id alice"
ssh -i bob_key bob@192.168.60.4  # doit fonctionner mais sans sudo
```
