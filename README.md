# Ansible GitLab

Déploiement automatisé de GitLab CE et GitLab Runner sur AlmaLinux via Ansible.

## Prérequis

### Machine de contrôle (où Ansible est exécuté)
- Ansible installé : `sudo dnf install -y ansible`
- Accès SSH aux VMs cibles avec clé publique

### VMs cibles
- AlmaLinux 9/10
- Utilisateur avec droits `sudo` sans mot de passe
- Ports 22, 80, 443 accessibles

---

## Structure du projet

```
ansible_gitlab/
├── inventory.ini          # Adresses IP des serveurs
├── vars.yml               # Variables de configuration (domaine, smtp, perf...)
├── .env.example           # Template pour les secrets
├── gitlab-install.yaml    # Installation de GitLab CE
├── gitlab-config.yaml     # Configuration (SSL, SMTP, monitoring...)
├── gitlab-runner.yaml     # Installation et enregistrement du Runner
├── gitlab-security.yaml   # Durcissement (firewall, politique de mots de passe)
├── gitlab-backup.yaml     # Script de sauvegarde automatique
└── gitlab-health.yaml     # Vérification de l'état de GitLab
```

---

## Configuration

### 1. Inventaire

Éditer [inventory.ini](inventory.ini) avec les IPs de vos serveurs :

```ini
[gitlab_servers]
192.168.122.19 ansible_user=do

[gitlab_runners]
192.168.122.154 ansible_user=do
```

### 2. Variables

Éditer [vars.yml](vars.yml) pour adapter à votre environnement :

| Variable | Défaut | Description |
|---|---|---|
| `gitlab_domain` | `gitlab.example.com` | Nom de domaine GitLab |
| `gitlab_email_from` | `gitlab@example.com` | Email expéditeur |
| `smtp_address` | `smtp.example.com` | Serveur SMTP |
| `smtp_port` | `587` | Port SMTP |
| `runner_concurrency` | `4` | Jobs parallèles par runner |
| `puma_workers` | `4` | Workers Puma (Rails) |
| `postgresql_shared_buffers` | `512MB` | Mémoire partagée PostgreSQL |
| `backup_keep_time` | `604800` | Rétention des sauvegardes (7 jours en secondes) |
| `backup_path` | `/var/opt/gitlab/backups` | Répertoire des sauvegardes |
| `monitoring_whitelist` | `127.0.0.1, 192.168.122.0/24` | IPs autorisées pour le health check |

### 3. Secrets

Copier `.env.example` en `.env` et remplir les valeurs :

```bash
cp .env.example .env
```

```bash
# .env
GITLAB_RUNNER_TOKEN=glrt-xxxxxxxxxxxx
SMTP_USER=user@example.com
SMTP_PASSWORD=motdepasse
```

Charger les secrets avant d'exécuter les playbooks :

```bash
source .env
```

### 4. Clé SSH

Copier votre clé SSH sur les VMs cibles :

```bash
ssh-copy-id do@192.168.122.19
ssh-copy-id do@192.168.122.154
```

---

## Déploiement

### Déploiement complet (ordre recommandé)

```bash
source .env
ansible-playbook -i inventory.ini \
  gitlab-install.yaml \
  gitlab-config.yaml \
  gitlab-runner.yaml \
  gitlab-security.yaml \
  gitlab-backup.yaml \
  gitlab-health.yaml
```

### Playbooks individuels

```bash
# Installation de GitLab CE
ansible-playbook -i inventory.ini gitlab-install.yaml

# Configuration (SSL, SMTP, monitoring)
ansible-playbook -i inventory.ini gitlab-config.yaml

# Installation du Runner
source .env && ansible-playbook -i inventory.ini gitlab-runner.yaml

# Durcissement sécurité
ansible-playbook -i inventory.ini gitlab-security.yaml

# Sauvegarde automatique
ansible-playbook -i inventory.ini gitlab-backup.yaml

# Vérification de l'état
ansible-playbook -i inventory.ini gitlab-health.yaml
```

### Dry-run (simulation sans changements)

```bash
ansible-playbook -i inventory.ini gitlab-install.yaml --check
```

---

## Accès à GitLab

Après le déploiement, GitLab est accessible via :

```
https://<IP_DU_SERVEUR>
```

Le certificat SSL est auto-signé — accepter l'avertissement du navigateur.

### Identifiants initiaux

- **Utilisateur :** `root`
- **Mot de passe :** généré automatiquement, récupérable avec :

```bash
ssh do@192.168.122.19 "sudo cat /etc/gitlab/initial_root_password"
```

> Ce fichier est supprimé automatiquement 24h après l'installation.

### Réinitialiser le mot de passe root

```bash
ssh do@192.168.122.19 "sudo gitlab-rake 'gitlab:password:reset[root]'"
```

---

## SSL

En environnement local, des certificats auto-signés sont générés automatiquement dans `/etc/gitlab/ssl/`.

Pour un déploiement en production avec un vrai domaine, activer Let's Encrypt dans [vars.yml](vars.yml) en modifiant [gitlab-config.yaml](gitlab-config.yaml) :

```ruby
letsencrypt['enable'] = true
letsencrypt['contact_emails'] = ['admin@example.com']
```

---

## Sauvegardes

Le playbook `gitlab-backup.yaml` installe un script de sauvegarde automatique planifié **tous les jours à 2h du matin**.

- Sauvegarde de l'application : `gitlab-backup create`
- Sauvegarde de la configuration : `gitlab.rb` + `gitlab-secrets.json`
- Nettoyage automatique des sauvegardes de plus de 7 jours
- Synchronisation optionnelle vers S3 si `aws` CLI est installé

Lancer une sauvegarde manuellement :

```bash
ssh do@192.168.122.19 "sudo /opt/gitlab-backup.sh"
```

---

## Dépannage

### Vérifier l'état des services

```bash
ssh do@192.168.122.19 "sudo gitlab-ctl status"
```

### Consulter les logs

```bash
# Logs nginx
ssh do@192.168.122.19 "sudo gitlab-ctl tail nginx"

# Logs Puma (Rails)
ssh do@192.168.122.19 "sudo gitlab-ctl tail puma"

# Logs PostgreSQL
ssh do@192.168.122.19 "sudo gitlab-ctl tail postgresql"
```

### Redémarrer GitLab

```bash
ssh do@192.168.122.19 "sudo gitlab-ctl restart"
```

### Relancer la configuration

```bash
ssh do@192.168.122.19 "sudo gitlab-ctl reconfigure"
```
