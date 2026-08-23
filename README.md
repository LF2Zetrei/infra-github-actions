# CD — déploiement continu vers le VPS

## Principe

Un seul workflow réutilisable (`reusable-deploy.yml`), stocké dans ce repo
centralisé. Chaque petit projet ne contient qu'un fichier minimal qui
l'appelle avec ses propres paramètres (nom, type, chemin).

Évite de dupliquer 80 lignes de YAML dans chacun de tes 20+ projets — un
seul endroit à maintenir si tu changes la logique de déploiement.

## Prérequis côté VPS

Le projet doit déjà avoir été déclaré une première fois via Ansible
(`group_vars/all.yml` + `ansible-playbook`) — le service systemd et la
config nginx doivent exister AVANT le premier déploiement CD. Le CD ne
fait que mettre à jour le code et redémarrer le service, il ne crée rien.

## Mise en place, étape par étape

### 1. Créer une clé SSH dédiée au déploiement (différente de ta clé perso)

```bash
ssh-keygen -t ed25519 -f ~/.ssh/deploy_key -N ""
```

### 2. Autoriser cette clé sur le VPS

```bash
ssh-copy-id -i ~/.ssh/deploy_key.pub lucas@bylucas.com
```

### 3. Autoriser l'utilisateur à redémarrer les services sans mot de passe

GitHub Actions doit pouvoir faire `sudo systemctl restart nom-projet` sans
prompt interactif. Sur le VPS :

```bash
sudo visudo -f /etc/sudoers.d/lucas-systemctl
```

Contenu (limite les droits au strict nécessaire, pas de sudo total) :

```
lucas ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart *, /usr/bin/systemctl is-active *
```

### 4. Créer ce repo "infra-github-actions" sur GitHub

Contient uniquement `reusable-deploy.yml`. Doit être un repo **public**,
ou tes autres repos doivent avoir accès si privé (paramètre d'organisation
GitHub, "Access" dans les settings Actions du repo centralisé).

### 5. Configurer les secrets, une fois par projet (ou au niveau organisation)

Dans chaque repo projet (ou mieux : au niveau de ton compte/organisation
GitHub pour ne pas les ressaisir à chaque fois) :

- `VPS_HOST` : l'IP ou le domaine du VPS
- `VPS_USER` : lucas
- `VPS_SSH_KEY` : le contenu de `~/.ssh/deploy_key` (la clé PRIVÉE)

### 6. Copier le fichier deploy.yml dans chaque projet

Adapter `exemple-projet-deploy.yml` (renommé en `.github/workflows/deploy.yml`
dans chaque repo) avec le bon `project_name`, `project_type`, `remote_path`.

## Flux une fois en place

```
git push sur main
   -> GitHub Actions build le projet
   -> rsync envoie les fichiers sur le VPS
   -> service systemd redémarré
   -> vérification que le service tourne
```

## Ce que ça ne fait pas

- Ne crée pas le service systemd ni la config nginx (rôle d'Ansible, une fois)
- N'exécute aucun test avant déploiement (à ajouter si tu veux, `npm test`
  avant l'étape de build, avec `if: failure()` pour bloquer le déploiement)
- N'a pas de rollback automatique en cas d'échec
