# Home Lab Apps - Déploiement Ansible pour Docker

Ce dépôt gère le déploiement d'applications conteneurisées sur une infrastructure **home lab** via Ansible, en respectant une architecture standardisée et sécurisée.

## 📁 Structure du dépôt
```
/home/dch/projects/home-lab-apps/
├── .env.json                # Configuration centrale (secrets, chemins, variables)
├── .gitignore              # Exclusion des fichiers sensibles
├── ansible/
│   ├── ansible.cfg          # Configuration Ansible
│   ├── inventories/
│   │   └── production/
│   │       ├── hosts.yml    # Mapping runners ↔ applications
│   │       └── group_vars/  # Variables globales
│   ├── roles/
│   │   └── docker_app_deploy/ # Rôle générique de déploiement
│   │       ├── tasks/       # Tâches Ansible
│   │       └── templates/   # Templates Docker Compose
│   ├── site.yml            # Playbook principal
│   └── site_vars/
│       └── apps_definitions/ # Définitions des applications
└── README.md               # Documentation (ce fichier)
```

## 🚀 Applications déployées
| Application      | Description                          | Port  | Statut          |
|------------------|--------------------------------------|-------|-----------------|
| **Hello**        | Serveur web minimal (NGINX)          | 8080  | ✅ Déployable   |
| **Home Assistant** | Plateforme domotique (stable)        | 8123  | ✅ Déployable   |

## ⚙️ Prérequis
- **Ansible** : `>= 2.10`
- **Docker** : `>= 20.10` sur les runners
- **SSH** : Clé `~/.ssh/id_ed25519_homelab` configurée pour les runners
- **GitHub** : Clé `~/.ssh/id_rsa_dchaumond` pour l'accès au dépôt

## 🔧 Déploiement
1. **Configurer `.env.json`** :
   ```json
   {
     "docker_admin_home": "/home/docker-admin/apps",
     "ansible": {"user": "docker-admin"},
     "apps": {
       "hello": {"image": "nginxdemos/hello:plain-text", "ports": ["8080:80"]}
     }
   }
   ```

2. **Lancer le déploiement** :
   ```bash
   cd ansible
   ansible-playbook site.yml --limit <runner>
   ```

3. **Vérifier les conteneurs** :
   ```bash
   ssh docker-admin@<runner> "docker ps"
   ```

## 🔐 Sécurité
- **Secrets** : Exclus via `.gitignore` (`.env.json`, clés SSH).
- **Permissions** : Tous les fichiers appartiennent à `docker-admin:docker`.
- **Réseaux** : Réseau Docker dédié (`app_network`) pour isoler les applications.

## 🛠️ Gestion des Applications

### 1. Ajouter une nouvelle application

#### **Étapes** :
1. **Définir l'application** dans `.env.json` :
   Ajoutez une entrée dans la section `apps` avec les paramètres requis (image, ports, volumes, variables d'environnement).
   ```json
   {
     "apps": {
       "votre_app": {
         "image": "votre_image:tag",
         "ports": ["8081:80"],
         "volumes": [
           {
             "host_path": "data",
             "container_path": "/config"
           }
         ],
         "env_vars": {
           "VAR1": "valeur1",
           "VAR2": "valeur2"
         }
       }
     }
   }
   ```

2. **Déclarer l'application dans l'inventaire** :
   Ajoutez le nom de l'application à la liste `apps` du runner cible dans `ansible/inventories/production/hosts.yml`.
   ```yaml
   apps-runner-ryzen:
     ansible_host: apps-runner-ryzen
     apps:
       - hello
       - homeassistant
       - votre_app  # <-- Ajoutez ici
   ```

3. **Créer les définitions Ansible (optionnel)** :
   Si l'application nécessite des tâches spécifiques, créez un fichier dans `ansible/site_vars/apps_definitions/` (ex: `app_votre_app.yml`).

---

### 2. Déployer une application

#### **Commande** :
```bash
cd /home/dch/projects/home-lab-apps/ansible
ansible-playbook site.yml --limit <runner> --tags <app_name>
```

#### **Exemple** (déployer `votre_app` sur `apps-runner-ryzen`) :
```bash
ansible-playbook site.yml --limit apps-runner-ryzen --tags votre_app
```

#### **Vérification** :
- **Conteneur** : `docker ps | grep <app_name>`
- **Accès** : `curl http://<runner>:<port>` ou via un navigateur.

---

### 3. Arrêter/Démarrer une application

#### **Arrêter** :
```bash
ansible -i inventories/production/hosts.yml <runner> -m shell -a "cd {{ env_json.docker_admin.apps_home }}/<app_name> && docker compose down"
```

#### **Démarrer** :
```bash
ansible -i inventories/production/hosts.yml <runner> -m shell -a "cd {{ env_json.docker_admin.apps_home }}/<app_name> && docker compose up -d"
```

#### **Redémarrer** :
```bash
ansible -i inventories/production/hosts.yml <runner> -m shell -a "docker restart <app_name>"
```

---

### 4. Supprimer une application

#### **Étapes** :
1. **Arrêter le conteneur** :
   ```bash
   ansible -i inventories/production/hosts.yml <runner> -m shell -a "docker stop <app_name> && docker rm <app_name>"
   ```

2. **Supprimer les répertoires** :
   ```bash
   ansible -i inventories/production/hosts.yml <runner> -m file -a "path={{ env_json.docker_admin.apps_home }}/<app_name> state=absent"
   ```

3. **Nettoyer la configuration** :
   - Supprimez l'entrée de l'application dans `.env.json`.
   - Retirez le nom de l'application de `ansible/inventories/production/hosts.yml`.

#### **Exemple** (supprimer `votre_app`) :
```bash
ansible -i inventories/production/hosts.yml apps-runner-ryzen -m shell -a "docker stop votre_app && docker rm votre_app"
ansible -i inventories/production/hosts.yml apps-runner-ryzen -m file -a "path=/home/docker-admin/apps/votre_app state=absent"
```

---

## 📜 Licence
MIT - Voir [LICENSE](LICENSE) *(à créer si nécessaire)*.