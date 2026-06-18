# Règles de Codage et Bonnes Pratiques pour le Déploiement Docker/Ansible

Ce document décrit les conventions, règles et apprentissages clés pour travailler avec ce dépôt.

---

## 1. Structure du Dépôt
### Arborescence Standard
```text
/home/dch/projects/home-lab-apps/
├── .env.json                # Variables centralisées (SECRETS et configurations) - LOCAL ONLY
├── ansible/
│   ├── inventories/
│   │   └── production/
│   │       ├── hosts.yml    # Mapping des applications → runners
│   │       └── group_vars/  # Variables globales des runners
│   ├── roles/
│   │   └── docker_app_deploy/
│   │       ├── tasks/       # Tâches Ansible (ex: main.yml)
│   │       └── templates/   # Templates Jinja2 pour générer:
│   │           ├── docker-compose.yml.j2  # Fichier Docker Compose généré
│   │           └── app_readme.j2          # Documentation générée (README.md)
│   ├── site_vars/
│   │   └── apps_definitions/ # Définitions des applications (app_<nom>.yml)
│   ├── deploy_app.yml       # Playbook de déploiement
│   └── site.yml            # Playbook maître
├── RULES.md                # Ce fichier
└── AGENTS.md               # Instructions pour les agents IA
```

### 1.2 Arborescence sur les Runners
- **Chemin** : `/home/docker-admin/apps/<app_name>/`
- **Fichiers déployés** :
  ```text
  /home/docker-admin/apps/
  └── <app_name>/
      ├── docker-compose.yml    # Généré par Ansible (ne pas modifier manuellement)
      ├── data/                 # Volumes persistants (permissions: docker-admin:docker, 770)
      └── README.md             # Documentation générée automatiquement
  ```
- **Fichiers exclus** :
  - `.env.json` (local uniquement, **jamais déployé**).
  - Fichiers temporaires ou de build (ex: `node_modules/`).
- **Permissions** :
  - `data/` : `chown docker-admin:docker && chmod 770`.
  - `docker-compose.yml` : `644` (lecture seule pour `docker-admin`).

### Détails des Templates Jinja2
- **Emplacement** : `ansible/roles/docker_app_deploy/templates/`
- **Fichiers critiques** :
  - `docker-compose.yml.j2` : Génère le fichier `docker-compose.yml` pour chaque application.
    - **Variables utilisées** : `env_json.apps[app_name]` (image, ports, volumes, etc.).
    - **Réseaux** : Utilise `app_network` si `network_mode` n'est pas défini.
  - `app_readme.j2` : Génère le fichier `README.md` pour chaque application.
    - **Contenu** : Description, ports, volumes, commandes de gestion, et variables d'environnement.
- **Validation** : Toujours tester la génération avec `ansible template` avant déploiement.

---

## 2. Fichiers Clés et Leur Rôle
### 2.1 `.env.json` (Local Only)
- **Rôle** :
  - Fichier **centralisé et local** (exclu de Git via `.gitignore`) pour définir **toutes** les variables (secrets, configurations, versions d'images, ports, volumes, etc.).
  - **Jamais déployé** sur les runners. Utilisé uniquement pour générer les configurations Ansible/Docker.
- **Règles** :
  - **Secrets** : Chiffrés via **Ansible Vault** (recommandé) ou exclus de Git.
  - **Structure obligatoire** :
    ```json
    {
      "docker_admin": { "user": "docker-admin", "uid": 1000, "gid": 1000, "home": "/home/docker-admin", "apps_home": "/home/docker-admin/apps" },
      "apps": {
        "<app_name>": {
          "image": "ghcr.io/org/app:1.2.3",  // Version sémantique (pas de 'latest')
          "ports": ["8080:80"],
          "volumes": [{"host_path": "data", "container_path": "/config"}],
          "env_vars": {"TZ": "Europe/Paris"}
        }
      },
      "networks": {
        "app_network": {"subnet": "172.21.0.0/16"}
      }
    }
    ```
  - **Exemple d'exclusion Git** : Ajouter `.env.json` dans `.gitignore.

### 2.2 `ansible/inventories/production/hosts.yml`
- **Rôle** : Définit **quel runner héberge quelle application**.
- **Structure** :
  ```yaml
  apps_runners:
    hosts:
      <runner_name>:
        ansible_host: <IP_ou_hostname>
        ansible_user: docker-admin
        apps:
          - app1
          - app2
  ```
- **Règle** : Éviter les doublons (un même `app_name` sur plusieurs runners).
- **Règles** :
  - Utiliser le format suivant pour mapper les applications :
    ```yaml
    apps_runners:
      hosts:
        <runner_name>:
          ansible_host: <IP_ou_hostname>
          ansible_user: docker-admin
          apps:
            - app1
            - app2
    ```
  - **Ne jamais** modifier manuellement les fichiers sur les runners. Tout changement doit passer par Ansible.

### 2.3 `ansible/site_vars/apps_definitions/app_<nom>.yml`
- **Rôle** : Définit les paramètres spécifiques à une application (utilise les variables de `.env.json`).
- **Règles** :
  - Toujours référencer les variables de `.env.json` via `{{ secrets.<chemin> }}`.
  - Exemple pour `homeassistant` :
    ```yaml
    app_name: homeassistant
    image: "{{ secrets.apps.homeassistant.image }}"
    network_mode: "{{ secrets.apps.homeassistant.network_mode }}"
    cap_add: "{{ secrets.apps.homeassistant.cap_add }}"
    volumes: "{{ secrets.apps.homeassistant.volumes }}"
    env_vars: "{{ secrets.apps.homeassistant.env_vars }}"
    ```

### 2.4 Templates Jinja2 (`roles/docker_app_deploy/templates/`)
- **Rôle** : Génèrent les fichiers `docker-compose.yml` et `README.md` pour chaque application.
- **Règles** :
  - **Syntaxe YAML** :
    - Indentation stricte (2 espaces, **pas de tabs**).
    - Sauts de ligne obligatoires entre les clés de niveau supérieur (ex: `restart:`, `network_mode:`, `environment:`).
    - Utiliser des guillemets pour les valeurs contenant des caractères spéciaux (ex: `"{{ value }}"`).
  - **Variables** :
    - Toujours référencer `.env.json` via `env_json.<chemin>`.
    - Exemple pour les volumes :
      ```jinja2
      volumes:
        {%- for volume in env_json.apps[app_name].volumes %}
        - "{{ env_json.docker_admin.apps_home }}/{{ app_name }}/{{ volume.host_path }}:{{ volume.container_path }}"
        {%- endfor %}
      ```
  - **Validation** :
    - Toujours valider les templates avec `yamllint` et `jinja2-lint` avant déploiement.
    - Tester la génération avec :
      ```bash
      ansible localhost -m template -a "src=roles/docker_app_deploy/templates/docker-compose.yml.j2 dest=/tmp/test.yml"
      ```

---

## 3. Règles de Déploiement
### 3.1 Idempotence
- **Définition** : Les playbooks Ansible doivent pouvoir être exécutés **plusieurs fois sans effet de bord**. Un déploiement idempotent garantit que :
  - Les fichiers générés (`docker-compose.yml`, `README.md`) sont **identiques** à chaque exécution.
  - Les conteneurs sont **recréés uniquement si nécessaire** (ex: changement d'image ou de configuration).
  - Les permissions et répertoires ne sont **pas modifiés** s'ils sont déjà corrects.
- **Outils** :
  - Utiliser les modules Ansible `template`, `file`, et `docker-compose` pour garantir l'idempotence.
  - Éviter les commandes shell brutes (ex: `docker-compose down && docker-compose up -d`). Préférer les modules dédiés.
  - **Exemple** :
    ```yaml
    - name: Démarrer l'application
      ansible.builtin.command: docker-compose up -d --remove-orphans
      args:
        chdir: "{{ env_json.docker_admin.apps_home | realpath }}/{{ app_name }}"
      register: docker_compose_up
      changed_when: "'Creating' in docker_compose_up.stdout or 'Recreating' in docker_compose_up.stdout"
    ```

### 3.2 Workflow de Déploiement (Local → Runner)
1. **Local** :
   - Définir les variables dans `.env.json` (ex: `apps.hello.image`).
   - Générer les configurations avec Ansible :
     ```bash
     ansible localhost -m template -a "src=ansible/roles/docker_app_deploy/templates/docker-compose.yml.j2 dest=/tmp/test.yml" --extra-vars '{"app_name": "hello"}'
     ```
2. **Runner** :
   - Déployer via `deploy_app.yml` :
     ```bash
     ansible-playbook ansible/deploy_app.yml --extra-vars "app_name=hello target_runner=apps-runner-ryzen"
     ```
   - Vérifier les permissions :
     ```bash
     ansible apps-runner-ryzen -m file -a "path=/home/docker-admin/apps/hello/data owner=docker-admin group=docker mode=770"
     ```
3. **Vérification** :
   - Statut des conteneurs :
     ```bash
     ansible <runner> -m shell -a "docker ps | grep <app_name>"
     ```
   - Logs :
     ```bash
     ansible <runner> -m shell -a "docker logs <app_name>"
     ```

### 3.2 Sécurité
- **Interdictions** :
  - ❌ **Jamais** utiliser `sudo` ou `root` pour les opérations Docker.
  - ❌ **Jamais** monter `/var/run/docker.sock` dans un conteneur (sauf agents de monitoring validés).
  - ❌ **Jamais** utiliser le tag `latest` pour les images Docker. Toujours spécifier une version sémantique.
  - ❌ **Jamais** stocker des secrets en clair dans les fichiers YAML ou templates.
- **Permissions** :
  - Les répertoires de volumes (`data/`) doivent appartenir à `docker-admin:docker` (UID/GID 1000).
  - Utiliser `chmod 770` pour les répertoires persistants (jamais `777`).

### 3.3 Résolution des Problèmes
| Problème                          | Solution                                                                                     |
|-----------------------------------|----------------------------------------------------------------------------------------------|
| **Erreur YAML**                   | Valider avec `yamllint` et `docker-compose config`. Vérifier les sauts de ligne et l'indentation. |
| **Conflit de ports**              | Vérifier les ports utilisés sur le runner avec `ss -tulnp`.                                |
| **Permissions sur les volumes**   | `chown docker-admin:docker /home/docker-admin/apps/<app_name>/data && chmod 770 data`      |
| **Image Docker introuvable**      | Vérifier le nom/tag de l'image dans `.env.json`.                                            |
| **Variables manquantes**          | Vérifier que toutes les variables sont définies dans `.env.json`.                          |

---

## 4. Gestion des Secrets et Sécurité
### 4.1 Règles pour les Secrets
- **Interdictions** :
  - ❌ **Jamais** stocker de secrets en clair dans :
    - `.env.json` (local).
    - Fichiers Ansible/YAML (ex: `hosts.yml`, `app_*.yml`).
    - Templates Jinja2.
  - ❌ **Jamais** utiliser `latest` pour les images Docker.
- **Bonnes pratiques** :
  - **Ansible Vault** : Chiffrer les secrets avec `ansible-vault encrypt_string`.
  - **Variables d'environnement** : Injecter les secrets via `.env` ou `docker secrets`.
  - **Exemple** :
    ```yaml
    # Dans app_<nom>.yml
    env_vars: "{{ secrets.apps.<app_name>.env_vars | combine({'DB_PASSWORD': vault_db_password}) }}"
    ```

### 4.2 Exemple de `.gitignore`
```gitignore
# Secrets locaux
.env.json
*.vault.yml

# Fichiers générés
**/docker-compose.yml
**/README.md
```

## 5. Apprentissages Clés
### 4.1 Problèmes Courants et Solutions
1. **Erreurs YAML dans les templates Jinja2** :
   - **Cause** : Sauts de ligne manquants entre les clés YAML (ex: `restart: unless-stoppednetwork_mode: host`).
   - **Solution** : Toujours ajouter un saut de ligne après chaque clé de niveau supérieur.

2. **Fusion incorrecte des variables d'environnement** :
   - **Cause** : Le template tente de fusionner des dictionnaires (`env_vars`) avec des clés statiques (ex: `PUID`).
   - **Solution** : Utiliser uniquement les variables définies dans `.env.json` et éviter les fusions manuelles.

3. **Problèmes de permissions sur les volumes** :
   - **Cause** : Les répertoires `data/` sont créés avec les mauvaises permissions.
   - **Solution** : Toujours vérifier les permissions après création :
     ```bash
     chown docker-admin:docker /home/docker-admin/apps/<app_name>/data && chmod 770 data
     ```

### 4.2 Bonnes Pratiques
- **Idempotence** : Les playbooks Ansible doivent être idempotents. Toujours utiliser des modules comme `file`, `template`, ou `docker-compose` pour garantir cela.
- **Réseaux Docker** : Préférer les réseaux dédiés (`bridge`) plutôt que le mode `host`, sauf pour les applications comme Home Assistant qui nécessitent un accès réseau direct.
- **Documentation** : Toujours générer un `README.md` pour chaque application avec les commandes de gestion et les détails techniques.
- **Validation** : Toujours valider les fichiers YAML et les templates avant déploiement.

---

## 5. Exemples Concrets
### 5.1 Déploiement de `hello`
```bash
ansible-playbook ansible/deploy_app.yml --extra-vars "app_name=hello target_runner=apps-runner-ryzen"
```

### 5.2 Déploiement de `homeassistant`
```bash
ansible-playbook ansible/deploy_app.yml --extra-vars "app_name=homeassistant target_runner=ha-runner-ryzen"
```

### 5.3 Vérification des Conteneurs
```bash
ansible apps-runner-ryzen -m shell -a "docker ps | grep hello"
ansible ha-runner-ryzen -m shell -a "docker ps | grep homeassistant"
```

---

## 6. Outils Recommandés
| Outil               | Usage                                                                                     |
|---------------------|-------------------------------------------------------------------------------------------|
| `yamllint`          | Valider la syntaxe YAML des fichiers et templates.                                        |
| `jinja2-lint`       | Valider la syntaxe des templates Jinja2.                                                  |
| `ansible-lint`      | Valider les playbooks Ansible.                                                            |
| `docker-compose`    | Valider et tester les fichiers `docker-compose.yml`.                                      |
| `jq`                | Manipuler et valider le fichier `.env.json`.                                              |

---

## 7. Contact et Support
Pour toute question ou problème, contacter l'équipe DevOps ou consulter la documentation officielle :
- [Ansible Documentation](https://docs.ansible.com/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [Jinja2 Documentation](https://jinja.palletsprojects.com/)