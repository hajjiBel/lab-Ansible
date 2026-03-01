# Formation Ansible 

## 📚 Table des Matières Complète

1. **[Lab 0 - Infrastructure Locale](./Lab-0-Infrastructure.md)**
   - Configuration Vagrant
   - Création des VMs
   - Réseau et SSH

2. **[Lab 1 - Installation d'Ansible](./Lab-01-Installation.md)** 
   - Installation sécurisée
   - Configuration SSH RSA 4096
   - Architecture d'Ansible
   - Tests de connectivité avancés

3. **[Lab 2 - Configuration d'Ansible](./Lab-02-Configuration.md)**
   - Fichier ansible.cfg optimisé
   - Structure du projet professionnelle
   - Inventaires YAML avancés
   - Inventaires dynamiques (AWS/Azure)
   - Variables multi-niveaux

4. **[Lab 3 - Commandes Ad-Hoc](./Lab-03-Commandes-Ad-Hoc.md)** 
   - Modules essentiels (ping, command, shell, setup, file, copy, yum, service, user, lineinfile)
   - Schémas d'exécution détaillés
   - Cas d'usage courants
   - Exercices pratiques

5. **[Lab 4 - Playbooks Ansible](./Lab-04-Playbooks.md)** 
   - Structure YAML complète
   - Playbooks simples et multi-plays
   - Modules essentiels (debug, template, block)
   - Conditions et boucles
   - Handlers et notifications
   - Gestion des erreurs

6. **[Lab 5 - Variables Avancées](./Lab-05-Variables.md)** 
   - Hiérarchie des variables
   - group_vars et host_vars
   - Filters Jinja2 (30+ filters)
   - Variables d'environnement
   - Ansible Vault pour données sensibles
   - Variables multi-environnement

7. **[Lab 6 - Contrôle d'Exécution](./Lab-06-Controle-Execution.md)** 
   - Conditions avancées avec when
   - Boucles : loop, until, with_*
   - Gestion des erreurs : rescue, always
   - Blocs d'exécution
   - Handlers avancés
   - Stratégies (linear, free, debug)

8. **[Lab 7 - Templates Jinja2](./Lab-07-Templates.md)** 
   - Syntaxe Jinja2 complète
   - Templates simples et complexes
   - Conditions et boucles dans templates
   - Filters avancés
   - Cas d'usage réels (nginx.conf, my.cnf, script shell)

9. **[Lab 8 - Rôles Ansible](./Lab-08-Roles.md)** 
   - Structure des rôles
   - Rôles simples (common) et complexes (webserver)
   - Dépendances entre rôles
   - Inclusion avancée
   - Partage sur Galaxy
   - Tests avec Molecule

10. **[Lab 9 - Ansible Vault](./Lab-09-Vault.md)** 
    - Chiffrement des données sensibles
    - Gestion des mots de passe
    - Integration dans playbooks
    - CI/CD (GitHub Actions, GitLab CI, Jenkins)
    - Bonnes pratiques de sécurité
