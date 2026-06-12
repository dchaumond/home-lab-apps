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

## 📜 Licence
MIT - Voir [LICENSE](LICENSE) *(à créer si nécessaire)*.