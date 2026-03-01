# 🧪 Lab 4 : Playbooks Ansible - Ubuntu 22.04

## 📚 Vue d'Ensemble

Ce laboratoire vous initie aux **playbooks Ansible**, l'élément central de l'automatisation. Vous apprendrez à transformer vos commandes ad-hoc en playbooks YAML structurés, réutilisables et idempotents.

### 🎯 Objectifs d'Apprentissage

À la fin de ce lab, vous serez capable de :

- ✅ Écrire des playbooks YAML avec syntaxe correcte
- ✅ Structurer des tâches complexes avec modules Ansible
- ✅ Gérer l'escalade de privilèges avec `become`
- ✅ Valider la syntaxe et déboguer les playbooks
- ✅ Comprendre l'idempotence et les états `changed/ok`
- ✅ Créer des déploiements applicatifs complets
- ✅ Utiliser les niveaux de verbosité pour le troubleshooting


### 📦 Prérequis Techniques

**Labs précédents complétés :**
- ✅ [Lab 0 : Infrastructure Vagrant](Lab-0-Creer-l-infrastructure-locale.md)
- ✅ [Lab 1 : Installation Ansible](Lab-1-Installation-de-Ansible.md)
- ✅ [Lab 2 : Configuration Ansible](Lab-2-Ansible-configuration.md)
- ✅ [Lab 3 : Commandes ad-hoc](Lab-3-Commandes-Ad-Hoc.md)

**Vérifications préalables :**

```bash
# Sur le nœud master
vagrant ssh master

# Vérifier Ansible
ansible --version
ansible all -m ping

# Vérifier l'inventaire
cat /etc/ansible/hosts
ansible-inventory --list
```

---

## 🎓 Concepts Fondamentaux

### 📖 Structure Complète d'un Playbook YAML

```yaml
---
- name: Nom descriptif du Play
  hosts: webservers                    # Groupe d'hôtes
  become: yes                         # Escalade privilèges
  gather_facts: yes                   # Collecter facts (défaut)
  tasks:                              # Liste des tâches
    - name: Tâche 1 - Installation
      apt:
        name: nginx
        state: latest
        update_cache: yes

    - name: Tâche 2 - Service
      service:
        name: nginx
        state: started
        enabled: yes

    - name: Tâche 3 - Déploiement
      copy:
        src: index.html
        dest: /var/www/html/index.html
        owner: www-data
        group: www-data
        mode: '0644'
```

### 🔑 Éléments Clés du Playbook

| Élément | Description | Obligatoire | Exemple |
|---------|-------------|-------------|---------|
| `---` | Début YAML | ✅ Oui | `---` |
| `name` | Nom du play | ❌ Non | `Déploiement Nginx` |
| `hosts` | Hôtes cibles | ✅ Oui | `webservers` |
| `become` | Escalade privilèges | ❌ Non | `become: yes` |
| `tasks` | Liste des tâches | ✅ Oui | `tasks:` |
| `name` (tâche) | Description tâche | ❌ Recommandé | `Installer Nginx` |

###  Idempotence - Principe Essentiel

**Définition :** Un playbook idempotent peut être exécuté N fois sans changer l'état du système après la 1ère exécution.

```
🚀 1ère exécution : CHANGED ✅ (modifications appliquées)
✅ 2ème exécution : OK ✅ (état déjà correct)
✅ 3ème exécution : OK ✅ (aucun changement)
```

**Modules Idempotents (Recommandés) :**

```
✅ apt, yum, dnf           # Gestion paquets
✅ service, systemd        # Services
✅ user, group             # Utilisateurs
✅ copy, file, template    # Fichiers
✅ git                     # Dépôts
✅ authorized_key          # SSH
```

**Modules NON Idempotents (À éviter) :**

```
❌ shell, command, raw     # Commandes arbitraires
❌ script                  # Scripts locaux
```

### 🎨 États de Sortie Ansible

| Couleur | État | JSON | Signification |
|---------|------|------|---------------|
| 🟢 Vert | `ok` | `"changed": false` | ✅ État correct |
| 🟡 Jaune | `changed` | `"changed": true` | ⚠️ Modification appliquée |
| 🔴 Rouge | `failed` | `"failed": true` | ❌ Erreur |
| 🟣 Violet | `skipped` | `"skipped": true` | ⏭️ Tâche ignorée |
| 🔵 Bleu | `unreachable` | - | 🌐 Hôte injoignable |

---

## 🚀 Premier Playbook : Déploiement Nginx Complet

### 📋 Contexte Professionnel

**Mission :** Déployer un serveur web Nginx sur webservers (app1, app2) avec page d'accueil personnalisée.

### 📁 Fichiers à créer

#### 1. Page HTML `/home/vagrant/index.html` :

```html
<!DOCTYPE html>
<html>
<head>
    <title>🚀 Serveur Nginx - Ansible</title>
    <meta charset="UTF-8">
    <style>
        body { 
            font-family: 'Segoe UI', Arial, sans-serif; 
            text-align: center; 
            margin-top: 100px; 
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
        }
        .container { max-width: 800px; margin: 0 auto; }
        h1 { font-size: 3em; margin-bottom: 20px; }
        .info { background: rgba(255,255,255,0.2); padding: 20px; border-radius: 10px; margin: 20px; }
    </style>
</head>
<body>
    <div class="container">
        <h1>🎉 Déploiement Nginx Réussi !</h1>
        <div class="info">
            <p><strong>Serveur :</strong> {{ ansible_hostname }}</p>
            <p><strong>IP :</strong> {{ ansible_default_ipv4.address }}</p>
            <p><strong>OS :</strong> {{ ansible_distribution }} {{ ansible_distribution_version }}</p>
            <p><strong>Déployé par :</strong> Ansible Lab 4</p>
            <p><strong>Date :</strong> {{ ansible_date_time.date }} {{ ansible_date_time.time }}</p>
        </div>
    </div>
</body>
</html>
```

#### 2. Playbook complet `/home/vagrant/nginx.yml` :

```yaml
---
- name:  Déploiement Serveur Web Nginx Complet
  hosts: webservers
  become: yes
  tasks:
    - name:  Installer dernière version Nginx
      apt:
        name: nginx
        state: latest
        update_cache: yes

    - name:  Démarrer et activer service Nginx
      service:
        name: nginx
        state: started
        enabled: yes

    - name:  Déployer page d'accueil personnalisée
      copy:
        src: index.html
        dest: /var/www/html/index.html
        owner: www-data
        group: www-data
        mode: '0644'
        backup: yes  # Sauvegarde ancienne version

    - name:  Vérifier port 80 accessible
      uri:
        url: http://localhost
        method: GET
        status_code: 200
      register: nginx_check

    - name:  Confirmer succès déploiement
      debug:
        msg: "Nginx opérationnel sur {{ inventory_hostname }} - Status: {{ nginx_check.status }}"
```

### 🎬 Exécution Progressive

```bash
# 1. Validation syntaxe
ansible-playbook --syntax-check nginx.yml

# 2. Simulation (dry-run)
ansible-playbook nginx.yml --check --diff

# 3. Exécution réelle
ansible-playbook nginx.yml -v

# 4. Vérification 2ème exécution (idempotence)
ansible-playbook nginx.yml

# 5. Tests fonctionnels
curl http://192.168.60.4
curl http://192.168.60.5
```

---

## 🔧 Guide Complet : Niveaux de Verbosité

| Niveau | Commande | Contenu | Usage |
|--------|----------|---------|-------|
| 0 | `ansible-playbook pb.yml` | Résumé tâches | Production |
| 1 | `-v` | + Temps exécution | Debug standard |
| 2 | `-vv` | + Facts Ansible | Analyse facts |
| 3 | `-vvv` | + SSH détaillé | Problèmes connexion |
| 4 | `-vvvv` | + Modules debug | Support technique |


---

## 🏗️ Application 1 : Sauvegarde Système Automatisée

### 🎯 Objectifs Techniques

Créer `/home/vagrant/backup.yml` qui effectue :

1. Crée `/backup` (755, vagrant:vagrant)
2. Archive `/etc` → `/backup/configuration.gz` (640)
3. Récupère l'archive sur master → `/backup/app2-configuration.gz`
4. Idempotent : 2ème exécution = tout ok

### 📄 Playbook Complet

```yaml
---
- name: 💾 Sauvegarde Configuration Système - App2
  hosts: app2
  become: yes
  tasks:
    # 1. Infrastructure backup
    - name:  Créer répertoire /backup
      file:
        path: /backup
        state: directory
        owner: vagrant
        group: vagrant
        mode: '0755'

    # 2. Archive gzip /etc
    - name:  Créer archive gzip de /etc
      shell: |
        tar -czf /backup/configuration.gz /etc --owner=vagrant --group=vagrant
      args:
        creates: /backup/configuration.gz
        executable: /bin/bash
      become_user: vagrant

    # 3. Permissions strictes
    - name:  Permissions archive (640)
      file:
        path: /backup/configuration.gz
        owner: vagrant
        group: vagrant
        mode: '0640'

    # 4. Récupération master
    - name:  Récupérer sur nœud contrôle
      fetch:
        src: /backup/configuration.gz
        dest: /backup/app2-configuration.gz
        flat: yes
        owner: vagrant
        group: vagrant
        mode: '0640'

    
```

### ✅ Tests Application 1

```bash
# Exécution
ansible-playbook backup.yml -v

# Vérifications
ls -la /backup/app2-configuration.gz
du -sh /backup/app2-configuration.gz
ansible app2 -a "ls -la /backup/"

# Test idempotence
ansible-playbook backup.yml  # Doit être tout OK (vert)
```


---

## 🚀 Application 2 : Déploiement Flask 

Convertir Application 2 Lab 3 (ad-hoc) → Playbook structuré idempotent.

Veuillez transposer les commandes ad-hoc utilisées dans la deuxième application 
du lab3 en un playbook Ansible.
Cela inclut la conversion des différentes opérations ad-hoc,
telles que l'installation de Git,
le clonage d'un référentiel, 
la configuration d'un fichier, 
l'installation de dépendances depuis un fichier requirements.txt, 
la création et la copie d'un fichier de service systemd, 
le reload et le démarrage du service, 
ainsi que la vérification de l'état du service. 
Intégrez ces opérations dans un playbook structuré. 




## 🎓 Compétences Acquises

| Module | Usage Principal |
|--------|-----------------|
| `apt` | Gestion paquets |
| `service/systemd` | Services |
| `copy` | Déploiement fichiers |
| `git` | Clonage dépôts |
| `pip` | Paquets Python |
| `fetch` | Récupération fichiers |
| `ini_file` | Configuration INI |
| `file` | Gestion fichiers/dossiers |
| `uri` | Tests HTTP |
| `wait_for` | Attente ports/services |

---

## 🚀 Prochaines Étapes

➡️ [Lab 5 : Variables Ansible](Lab-5-Variables.md)

---

