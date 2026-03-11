# Déploiement Ansible — deploy-ppc

Déploie les services PPC (`ppc`, `dbsync`, `promtail`) sur des edges Tailscale.

## Prérequis

### Machine locale (contrôleur Ansible)
```bash
pip install ansible
ansible-galaxy collection install community.docker ansible.posix
```

### Chaque edge
- Docker + Docker Compose v2 installés
- Tailscale actif et connecté au même tailnet
- Utilisateur `ubuntu` (ou autre) dans le groupe `docker`
- SSH accessible via Tailscale MagicDNS

---

## Configuration

### 1. Inventaire — `inventory/hosts.yml`

Ajouter chaque edge avec son nom MagicDNS Tailscale ou son IP `100.x.x.x` :

```yaml
edge-mon-site:
  ansible_host: mon-edge.tailnet-xyz.ts.net   # ou 100.x.y.z
```

Obtenir les noms/IPs disponibles :
```bash
tailscale status
```

### 2. Variables par site — `inventory/host_vars/<edge>/vars.yml`

```yaml
site_name: mon-site
```

### 3. Secrets Postgres — `inventory/host_vars/<edge>/vault.yml`

Remplir les valeurs puis chiffrer :
```bash
ansible-vault encrypt inventory/host_vars/mon-edge/vault.yml
```

---

## Déploiement

```bash
# Tester la connectivité SSH vers tous les edges
ansible edges -m ping --ask-vault-pass

# Déployer sur tous les edges (en parallèle)
ansible-playbook playbook.yml --ask-vault-pass

# Déployer sur un seul edge
ansible-playbook playbook.yml --limit edge-site-01 --ask-vault-pass

# Dry-run (voir ce qui serait changé)
ansible-playbook playbook.yml --check --ask-vault-pass
```

---

## Ce que fait le playbook

1. Crée `/data/app/deploy-ppc/` sur l'edge si absent
2. Synchronise le repo local (rsync, sans `.env` ni `.git`)
3. Génère les fichiers `.env` à partir des `host_vars` (via ansible-vault)
4. Lance `docker compose pull && docker compose up -d` pour `ppc`, `dbsync`, `promtail`

---

## Ajouter un nouvel edge

```bash
# 1. Créer les host_vars
mkdir -p inventory/host_vars/mon-nouvel-edge
cp inventory/host_vars/edge-site-01/vars.yml inventory/host_vars/mon-nouvel-edge/vars.yml
cp inventory/host_vars/edge-site-01/vault.yml inventory/host_vars/mon-nouvel-edge/vault.yml

# 2. Éditer vars.yml et vault.yml
# 3. Chiffrer le vault
ansible-vault encrypt inventory/host_vars/mon-nouvel-edge/vault.yml

# 4. Ajouter l'host dans inventory/hosts.yml
# 5. Déployer
ansible-playbook playbook.yml --limit mon-nouvel-edge --ask-vault-pass
```
