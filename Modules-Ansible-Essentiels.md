# Modules Ansible Essentiels

Référence rapide des modules Ansible les plus utilisés, classés par catégorie.

## Connectivité et exécution de commandes

| Module | Description |
|---|---|
| `ping` | Vérifie la connectivité et la présence de Python sur l'hôte |
| `command` | Exécute une commande simple (sans interprétation shell) |
| `shell` | Exécute une commande via le shell (pipes, redirections) |
| `raw` | Exécute une commande brute sans Python distant (utile pour le bootstrap) |
| `script` | Copie et exécute un script local sur l'hôte distant |

## Gestion de fichiers

| Module | Description |
|---|---|
| `copy` | Copie un fichier local vers l'hôte distant |
| `fetch` | Récupère un fichier de l'hôte distant vers le contrôleur |
| `template` | Génère un fichier à partir d'un modèle Jinja2 |
| `file` | Gère fichiers, répertoires, liens symboliques et leurs permissions |
| `lineinfile` | Assure la présence/absence d'une ligne précise dans un fichier |
| `blockinfile` | Insère ou remplace un bloc de texte délimité dans un fichier |
| `ini_file` | Modifie une valeur dans un fichier de configuration INI |
| `stat` | Récupère les métadonnées d'un fichier (existence, taille, permissions) |
| `unarchive` | Extrait une archive (tar, zip) sur l'hôte distant |
| `get_url` | Télécharge un fichier depuis une URL vers l'hôte distant |
| `synchronize` | Synchronise des fichiers via `rsync` entre contrôleur et hôte |

## Gestion des paquets

| Module | Description |
|---|---|
| `apt` | Gère les paquets sur systèmes Debian/Ubuntu |
| `yum` | Gère les paquets sur systèmes RHEL/CentOS (anciennes versions) |
| `dnf` | Gère les paquets sur systèmes RHEL/CentOS/Fedora récents |
| `package` | Gère les paquets de façon générique, indépendante de la distribution |

## Services et système

| Module | Description |
|---|---|
| `service` | Gère l'état d'un service (démarré, arrêté, activé) |
| `systemd` | Gère les services via systemd (plus complet que `service`) |
| `user` | Crée, modifie ou supprime un utilisateur système |
| `group` | Crée, modifie ou supprime un groupe système |
| `authorized_key` | Gère les clés SSH autorisées d'un utilisateur |
| `cron` | Gère les tâches planifiées (crontab) via Ansible |
| `mount` | Gère les points de montage d'un hôte |
| `hostname` | Définit le nom d'hôte d'une machine |
| `sysctl` | Modifie des paramètres noyau (`/etc/sysctl.conf`) |
| `reboot` | Redémarre l'hôte distant et attend qu'il soit de nouveau joignable |

## Sources de code et réseau

| Module | Description |
|---|---|
| `git` | Clone ou met à jour un dépôt Git |
| `uri` | Effectue des requêtes HTTP/API et vérifie les réponses |
| `wait_for` | Attend qu'une condition soit remplie (port ouvert, fichier présent…) |

## Contrôle de flux et débogage

| Module | Description |
|---|---|
| `debug` | Affiche un message ou la valeur d'une variable pour du débogage |
| `set_fact` | Définit une variable calculée pendant l'exécution du playbook |
| `assert` | Vérifie qu'une condition est vraie, échoue sinon |
| `pause` | Met le playbook en pause (attente ou saisie utilisateur) |
| `include_role` / `import_role` | Inclut un rôle dynamiquement ou statiquement dans les tâches |
| `include_tasks` / `import_tasks` | Inclut un fichier de tâches externe |

## Collecte d'informations

| Module | Description |
|---|---|
| `setup` | Collecte les *facts* système d'un hôte (appelé implicitement au début d'un play) |

## Lookup plugins courants (rappel)

| Plugin | Description |
|---|---|
| `lookup('file', ...)` | Lit le contenu d'un fichier local |
| `lookup('pipe', ...)` | Récupère le résultat d'une commande exécutée localement |
| `lookup('url', ...)` | Récupère le contenu d'une URL |
| `lookup('env', ...)` | Lit une variable d'environnement |
