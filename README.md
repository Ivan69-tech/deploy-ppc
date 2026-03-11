# deploy-ppc

Repo de déploiement des services Docker sur des edges via Tailscale :

| Service | Rôle |
|---|---|
| `ppc` | Contrôleur Python principal (API port 8008, frontend port 8888) |
| `dbsync` | Synchronisation SQLite → Postgres toutes les 15s |
| `promtail` | Collecte de logs vers Loki |

Les fichiers de chaque service sont déployés dans `/data/app/deploy-ppc/` sur chaque edge.

---

## Prérequis

### Machine locale (contrôleur)
- Tailscale connecté au même tailnet que les edges
- Ansible installé : `pip install ansible`
- Collections Ansible : `ansible-galaxy collection install community.docker ansible.posix`

### Chaque edge
- Docker + Docker Compose v2 installés
- Tailscale actif et connecté
- Utilisateur SSH (défaut : `ubuntu`) dans le groupe `docker`
- SSH accessible via MagicDNS Tailscale

---

## Structure du repo

```
deploy-ppc/
├── ppc/                    # Service PPC principal
│   ├── docker-compose.yml
│   ├── config.yaml
│   ├── frontend_config.yaml
│   └── .env                # gitignore — généré par Ansible (site_name)
├── dbsync/                 # Synchronisation base de données
│   ├── docker-compose.yml
│   ├── config.yaml
│   └── .env                # gitignore — généré par Ansible (credentials Postgres)
├── promtail/               # Collecte de logs
│   ├── docker-compose.yml
│   └── promtail-config.yml
├── ansible/                # Déploiement multi-edges
│   ├── ansible.cfg
│   ├── playbook.yml
│   ├── inventory/
│   │   ├── hosts.yml                        # liste des edges
│   │   └── host_vars/<edge>/
│   │       ├── vars.yml                     # site_name
│   │       └── vault.yml                    # secrets Postgres (chiffrés)
│   └── roles/deploy-ppc/
│       ├── tasks/main.yml
│       └── templates/                       # génération des .env
└── deploy-ppc.py           # Script de déploiement pour un seul edge
```

---

## Déploiement Ansible (recommandé — multi-edges)

### 1. Configurer l'inventaire

Éditer `ansible/inventory/hosts.yml` et ajouter chaque edge avec son nom MagicDNS Tailscale ou son IP `100.x.x.x` :

```yaml
edges:
  hosts:
    mon-edge:
      ansible_host: mon-edge   # nom dans `tailscale status`
```

### 2. Configurer les variables par edge

Pour chaque edge, créer un dossier dans `ansible/inventory/host_vars/` :

```bash
mkdir -p ansible/inventory/host_vars/mon-edge
```

**`vars.yml`** — variables non-sensibles :
```yaml
site_name: mon-site
```

**`vault.yml`** — secrets Postgres (à chiffrer) :
```yaml
dbsync_postgres_host: "db.example.com"
dbsync_postgres_port: 5432
dbsync_postgres_db: "ppc"
dbsync_postgres_user: "ppc_user"
dbsync_postgres_password: "secret"
```

Chiffrer le vault :
```bash
ansible-vault encrypt ansible/inventory/host_vars/mon-edge/vault.yml
```

### 3. Déployer

```bash
cd ansible/

# Tester la connectivité SSH vers tous les edges
ansible edges -m ping --ask-vault-pass

# Déployer sur tous les edges en parallèle
ansible-playbook playbook.yml --ask-vault-pass

# Déployer sur un seul edge
ansible-playbook playbook.yml --limit mon-edge --ask-vault-pass
```

Le playbook effectue dans l'ordre :
1. Crée `/data/app/deploy-ppc/` sur l'edge
2. Synchronise les fichiers du repo (rsync, sans `.env` ni `.git`)
3. Génère les fichiers `.env` à partir des variables vault
4. Lance `docker compose pull && docker compose up -d` pour chaque service

---

## Déploiement script Python (edge unique)

```bash
python deploy-ppc.py --edge <NOM_EDGE> [--user ubuntu]
```

Se connecte via `tailscale ssh`, fait un `git pull` sur l'edge puis relance tous les services.

> Nécessite que le repo soit déjà cloné dans `/data/app/deploy-ppc` sur l'edge et que Tailscale SSH soit activé.
