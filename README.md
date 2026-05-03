# deploy-ppc

Repo de déploiement des services Docker sur des edges via Tailscale :

| Service | Rôle |
|---|---|
| `ppc` | Contrôleur Python principal (API port 8008, frontend port 8888) |
| `dbsync` | Synchronisation SQLite → Postgres toutes les 15s |
| `promtail` | Collecte de logs vers Loki |

Les fichiers de chaque service sont déployés dans `/data/app/` sur chaque edge.

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
SITE_NAME: testbench-edge-rpi
```

**`vault.yml`** — secrets Postgres à renseigner en local :

```yaml
POSTGRES_HOST: "db.example.com"
POSTGRES_PORT: 5432
POSTGRES_DATABASE: "ppc"
POSTGRES_USER: "ppc_user"
POSTGRES_PASSWORD: "secret"
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

1. Crée `/data/app/` sur l'edge
2. Synchronise les fichiers du repo (rsync, sans `.env` ni `.git`)
3. Génère les fichiers `.env` à partir des variables vault
4. Lance `docker compose pull && docker compose up -d` pour chaque service
