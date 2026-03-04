Voici quelques exemples d'utilisation du module "lookup" :

***Récupérer une valeur à partir d'un fichier YAML :


- name: Load data from file
  debug:
    msg: "{{ lookup('file', '/path/to/data.yml') }}"


***Récupérer le résultat d'une commande exécutée sur l'hôte distant :

- name: Get current date
  debug:
    msg: "{{ lookup('pipe', 'date +%Y-%m-%d') }}"
	
	
	
	
***Récupérer le contenu d'une URL :

- name: Get website content
  debug:
    msg: "{{ lookup('url', 'http://example.com') }}"
	
	
	
	
	
**Récupérer une variable définie dans un fichier d'inventaire :

- name: Get variable from inventory
  debug:
    msg: "{{ lookup('env', 'MY_VARIABLE') }}"
	
	
	
***Récupérer des informations de cache de facteurs :
- name: Get cached fact
  debug:
    msg: "{{ lookup('ansible_facts', 'ansible_distribution_version') }}"