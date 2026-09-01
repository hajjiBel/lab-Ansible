**Lab 10 – Ansible Tower (Semaphore UI)
=======================================

## 🎯 Objectifs

À la fin de ce lab, vous serez capable de :

- Comprendre à quoi sert un outil de type "Ansible Tower" et les problèmes qu'il résout
- Déployer une interface web Semaphore UI avec Docker Compose
- Connecter Semaphore à un dépôt Git contenant vos playbooks
- Créer un inventaire, une clé d'accès (credential) et un template de tâche
- Exécuter un de vos playbooks des labs précédents directement depuis l'interface web

## 📋 Introduction

Jusqu'à présent, tous les playbooks de cette formation ont été exécutés en ligne de
commande, depuis le nœud `master`, avec `ansible-playbook`. Cela fonctionne bien
pour un usage individuel, mais pose problème dès qu'une équipe doit partager
l'automatisation :

```
❌ Pas de traçabilité : qui a exécuté quel playbook, et quand ?
❌ Pas de centralisation : chacun lance les playbooks depuis sa propre machine
❌ Pas d'interface : il faut connaître la ligne de commande
❌ Pas de planification simple : difficile de programmer une exécution récurrente
```

**Ansible Tower** (version commerciale Red Hat) et son équivalent open-source
**AWX** répondent à ce besoin en ajoutant une interface web par-dessus Ansible.
Dans ce lab, nous utilisons **Semaphore UI**, une alternative légère et open-source
qui offre les mêmes concepts (Projects, Inventories, Credentials, Templates) tout
en étant beaucoup plus rapide à déployer pour un lab pédagogique.

### Schéma du Lab

```
┌───────────────────────────────┐
│   Dépôt Git (playbooks)        │
│   nginx.yml, backup.yml, ...   │
└───────────────┬────────────────┘
                │ clone / sync
                ▼
┌───────────────────────────────┐
│        Semaphore UI            │
│   (conteneur Docker, port 3000)│
└───────────────┬────────────────┘
                │ SSH
                ▼
┌───────────────────────────────┐
│   Inventaire : app1, app2, db  │
│   (les mêmes machines que      │
│    dans les labs précédents)   │
└───────────────────────────────┘
```

## 📦 Prérequis

- Docker et Docker Compose installés sur `master` (voir Lab 0)
- Les labs 1 à 4 déjà réalisés (inventaire fonctionnel, playbooks `nginx.yml` et
  `backup.yml` disponibles)
- Un dépôt Git (GitHub, GitLab, ou un dépôt local) contenant vos playbooks

---

## Étape 1 : Déployer Semaphore UI

Sur `master`, créez un répertoire dédié :

```bash
mkdir -p ~/semaphore && cd ~/semaphore
```

Créez le fichier `docker-compose.yml` suivant :

```yaml
services:
  semaphore:
    image: semaphoreui/semaphore:latest
    ports:
      - "3000:3000"
    environment:
      SEMAPHORE_DB_DIALECT: postgres
      SEMAPHORE_DB_HOST: postgres
      SEMAPHORE_DB_PORT: 5432
      SEMAPHORE_DB_NAME: semaphore
      SEMAPHORE_DB_USER: semaphore
      SEMAPHORE_DB_PASS: semaphore
      SEMAPHORE_ADMIN: admin
      SEMAPHORE_ADMIN_PASSWORD: admin
      SEMAPHORE_ADMIN_NAME: Admin
      SEMAPHORE_ADMIN_EMAIL: admin@example.com
    depends_on:
      - postgres
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: semaphore
      POSTGRES_USER: semaphore
      POSTGRES_PASSWORD: semaphore
    volumes:
      - postgres_data:/var/lib/postgresql/data
volumes:
  postgres_data:
```

> 💡 Contrairement à une base embarquée, Semaphore s'appuie ici sur un conteneur
> **PostgreSQL** dédié pour stocker projets, inventaires et historique des jobs —
> une configuration plus proche de ce qu'on retrouve en production.

Lancez les conteneurs :

```bash
docker compose up -d
```

Vérifiez qu'ils tournent :

```bash
docker ps
# Vous devriez voir deux conteneurs en état "Up" : "semaphore" et "postgres"
```

## Étape 2 : Premier accès à l'interface

1. Ouvrez votre navigateur sur `http://<IP_MASTER>:3000`
2. Connectez-vous avec les identifiants définis dans le `docker-compose.yml` :
   - **Username** : `admin`
   - **Password** : `admin`

Vous arrivez sur le tableau de bord, vide pour l'instant.

## Étape 3 : Créer un projet

Dans Semaphore, un **Project** regroupe tout ce dont vous avez besoin pour
exécuter des playbooks : dépôt Git, inventaire, clés d'accès, templates.

1. Cliquez sur **New Project**
2. Nommez-le `Formation Ansible`
3. Validez

## Étape 4 : Ajouter la clé SSH (Key Store)

Semaphore a besoin de la clé privée utilisée pour se connecter à `app1`, `app2`
et `db` (la même que dans le Lab 1).

1. Menu **Key Store** → **New Key**
2. Name : `ssh-vagrant`
3. Type : **SSH Key**
4. **Login** : `vagrant` ⚠️ ce champ définit l'utilisateur SSH utilisé pour la
   connexion et **prend le dessus sur `ansible_user` de l'inventaire** — s'il
   est laissé vide ou par défaut, Semaphore tentera de se connecter avec un
   mauvais utilisateur (ex. `semaphore`) et l'exécution échouera avec
   `Permission denied (publickey)`
5. Collez le contenu de votre clé privée (`~/.ssh/id_rsa` sur `master`)
6. Enregistrez

## Étape 5 : Déclarer le dépôt Git (Repository)

1. Menu **Repositories** → **New Repository**
2. Name : `playbooks-formation`
3. URL : l'URL de votre dépôt Git contenant vos playbooks (`nginx.yml`,
   `backup.yml`, etc.)
4. SSH Key : sélectionnez `ssh-vagrant` (ou une clé de déploiement Git dédiée si
   le dépôt est privé)
5. Enregistrez

## Étape 6 : Créer l'inventaire

1. Menu **Inventory** → **New Inventory**
2. Name : `inventaire-lab`
3. Type : **Static**
4. Collez le contenu de votre `inventory.ini` du Lab 1 :

```ini
[webservers]
app1 ansible_host=192.168.60.4
app2 ansible_host=192.168.60.5

[databases]
db ansible_host=192.168.60.6

[all_servers:vars]
ansible_user=vagrant
```

5. User Credentials : sélectionnez `ssh-vagrant`
6. Enregistrez

## Étape 7 : Créer un Template de tâche

Le **Task Template** est l'équivalent d'un "Job Template" dans Tower/AWX : il
relie un playbook, un inventaire et des credentials.

1. Menu **Task Templates** → **New Template**
2. Name : `Déploiement Nginx`
3. Playbook Filename : `nginx.yml` (celui du Lab 4)
4. Repository : `playbooks-formation`
5. Inventory : `inventaire-lab`
6. Environment : laissez vide pour l'instant
7. Enregistrez

## Étape 8 : Exécuter le playbook depuis l'interface

1. Ouvrez le template `Déploiement Nginx`
2. Cliquez sur **Run**
3. Observez la sortie en temps réel : elle correspond exactement à ce que vous
   obtiendriez avec `ansible-playbook nginx.yml`, mais avec :
   - L'historique de toutes les exécutions passées
   - L'utilisateur qui a lancé le job
   - La date et la durée d'exécution

## Étape 9 : Déclencher un template automatiquement depuis GitHub

Dans un vrai contexte d'équipe, on ne clique pas manuellement sur **Run** à
chaque fois : c'est un `git push` qui doit déclencher l'exécution du playbook.
C'est le rôle des **Integrations** (webhooks) de Semaphore, combinées à un
webhook GitHub configuré sur votre dépôt `playbooks-formation`.

### 9.1 Exposer Semaphore sur Internet

`master` est sur le réseau privé Vagrant (`192.168.60.1`), injoignable depuis
GitHub. Pour ce lab, on utilise **ngrok** afin d'obtenir une URL publique
temporaire qui redirige vers votre Semaphore local :

```bash
# Sur master, installer ngrok (voir https://ngrok.com/download) puis :
ngrok http 3000
```

Ngrok affiche une URL du type `https://xxxx-xx-xx-xx-xx.ngrok-free.app` : notez-la,
elle redirige vers `http://localhost:3000` de votre conteneur Semaphore.

> 💡 En entreprise, Semaphore serait hébergé sur un serveur avec une adresse
> publique ou accessible via VPN : ngrok ne sert ici qu'à simuler cette
> accessibilité pour le lab.

### 9.2 Créer l'Integration côté Semaphore

1. Dans votre projet, ouvrez le menu **Integrations** → **New Integration**
2. Name : `webhook-github-nginx`
3. Task Template : sélectionnez `Déploiement Nginx`
4. Auth Method : **HMAC** (c'est la méthode de signature utilisée nativement
   par GitHub pour authentifier ses webhooks)
5. Secret : générez une chaîne aléatoire, par exemple :
   ```bash
   openssl rand -hex 20
   ```
   et collez-la dans le champ **Secret**
6. Enregistrez, puis copiez l'**URL du webhook** générée (de la forme
   `https://xxxx.ngrok-free.app/api/integrations/<id>/...`)

### 9.3 Configurer le webhook côté GitHub

1. Sur GitHub, ouvrez le dépôt `playbooks-formation`
2. **Settings** → **Webhooks** → **Add webhook**
3. **Payload URL** : collez l'URL du webhook copiée à l'étape précédente
4. **Content type** : `application/json`
5. **Secret** : collez exactement le même secret que celui saisi dans
   Semaphore (Étape 9.2)
6. **Which events would you like to trigger this webhook?** : cochez
   **Just the push event**
7. Cliquez sur **Add webhook**

GitHub envoie immédiatement un événement `ping` : dans l'onglet **Recent
Deliveries** du webhook, vous devez voir une réponse `200`.

## ✅ Application 2 : déclencher le job avec un commit

1. En local, modifiez légèrement le playbook `nginx.yml` (par exemple, ajoutez
   un commentaire ou changez le message du `debug` final)
2. Commitez et poussez le changement :
   ```bash
   git add nginx.yml
   git commit -m "test: déclenchement du job via webhook GitHub"
   git push
   ```
3. Sur GitHub, dans **Settings → Webhooks → Recent Deliveries**, vérifiez
   qu'une nouvelle livraison `push` est apparue avec un statut `200`
4. Dans l'interface Semaphore, menu **History**, vérifiez qu'un nouveau job
   `Déploiement Nginx` a démarré automatiquement, sans avoir cliqué sur **Run**

**Critères de réussite :**
- ✅ Le webhook GitHub affiche une livraison `push` réussie (`200`)
- ✅ Le job `Déploiement Nginx` apparaît dans **History**, déclenché par le
  commit et non par un clic manuel dans l'UI

---

## ✅ Application 1 : créer un second template

En reprenant le playbook `backup.yml` du Lab 4 :

1. Créez un nouveau Task Template `Sauvegarde App2`
2. Limitez son exécution à l'hôte `app2` (champ **Limit**)
3. Exécutez-le depuis l'interface
4. Vérifiez dans l'historique (**History**) que le job apparaît avec son statut,
   sa durée et les logs complets

**Critères de réussite :**
- ✅ Le job `Déploiement Nginx` s'exécute avec succès depuis l'interface
- ✅ Le job `Sauvegarde App2` s'exécute uniquement sur `app2`
- ✅ L'historique des deux exécutions est consultable dans **History**

---

## 🐛 Dépannage

### Semaphore ne démarre pas

```bash
docker compose logs semaphore
```

Vérifiez que le port 3000 n'est pas déjà utilisé par un autre service.

### Erreur "Permission denied (publickey)" lors de l'exécution d'un job

- Vérifiez que la clé collée dans **Key Store** est bien la clé **privée**
  complète (avec les lignes `-----BEGIN ... -----` et `-----END ... -----`)
- Vérifiez que la clé publique correspondante est bien présente dans
  `~/.ssh/authorized_keys` sur `app1`, `app2` et `db` (Lab 1)

### Le job se connecte avec le mauvais utilisateur (`semaphore@...: Permission denied`)

C'est le piège le plus fréquent : le champ **Login** de la credential SSH
(Key Store) n'a pas été renseigné avec `vagrant` et Semaphore utilise un
utilisateur par défaut à la place de `ansible_user` défini dans l'inventaire.
Éditez la credential `ssh-vagrant`, corrigez le champ **Login**, puis relancez
le job.

### Le webhook GitHub affiche une erreur (`401`, signature invalide) ou n'arrive jamais

- Vérifiez que le **Secret** est **identique, sans espace superflu**, des deux
  côtés (GitHub et Integration Semaphore) — une signature HMAC invalide
  entraîne un rejet de la requête
- Vérifiez que le tunnel ngrok est toujours actif : les URL gratuites
  expirent ou changent à chaque redémarrage de `ngrok http 3000` ; si l'URL a
  changé, mettez à jour le **Payload URL** dans les paramètres du webhook
  GitHub
- Dans **Recent Deliveries** sur GitHub, cliquez sur la livraison en échec
  pour voir le code de retour exact et le corps de la réponse

### Le Repository ne se synchronise pas

- Vérifiez l'URL du dépôt (HTTPS ou SSH selon la clé fournie)
- Si le dépôt est privé, utilisez une clé de déploiement dédiée plutôt que la
  clé SSH des machines cibles

---

## 📚 Ressources supplémentaires

- Documentation Semaphore UI : https://docs.semui.co/
- Documentation Ansible Tower : https://docs.ansible.com/ansible-tower/
- Comparatif AWX / Semaphore / Tower : recherchez "Ansible Tower alternatives"
  sur le blog Ansible officiel

## 🎓 Conclusion

Ce lab clôt la formation Ansible : vous êtes parti d'une infrastructure locale
(Lab 0), êtes passé par l'installation, la configuration, les commandes ad-hoc,
les playbooks, les variables, le contrôle d'exécution, les templates, les rôles
et le chiffrement des secrets avec Vault — pour terminer par la mise en place
d'une interface centralisée d'exécution, brique essentielle pour faire passer
Ansible d'un outil individuel à un outil d'équipe.

---

**Fin de la formation Ansible 🎉**
