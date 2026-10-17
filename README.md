# shared-workflows

Workflows GitHub Actions réutilisables pour les API .NET.

| Workflow | Rôle |
| --- | --- |
| [dotnet-sonar-ci.yml](.github/workflows/dotnet-sonar-ci.yml) | CI : build, tests, couverture et analyse SonarCloud |
| [dotnet-cd.yaml](.github/workflows/dotnet-cd.yaml) | CD : version, build Docker, scan Trivy, push sur Zot, déploiement via Watchtower |

Ce sont des workflows `workflow_call` : ils ne se lancent pas seuls. Chaque projet crée son propre workflow qui les appelle (exemples plus bas). C'est aussi dans le workflow du projet que se déclare le lancement manuel (`workflow_dispatch`).

## Sommaire

- [CI : build, tests et SonarCloud](#ci--build-tests-et-sonarcloud)
- [CD : build, push et déploiement](#cd--build-push-et-déploiement)
- [Préparer le serveur (une seule fois)](#préparer-le-serveur-une-seule-fois)
- [Lancer la CI/CD à la main](#lancer-la-cicd-à-la-main)
- [Versionnement](#versionnement)
- [Dépannage](#dépannage)

---

## CI : build, tests et SonarCloud

Étapes : checkout, setup .NET et Java 17, `dotnet restore`, début d'analyse Sonar, `dotnet build` (Release), `dotnet test` avec couverture OpenCover, fin d'analyse. Le job échoue si le quality gate Sonar n'est pas respecté (`sonar.qualitygate.wait=true`).

### Appel depuis un projet

`.github/workflows/ci.yaml` dans le repo de l'API :

```yaml
name: CI

on:
  pull_request:
  workflow_dispatch:

jobs:
  ci:
    uses: <org>/shared-workflows/.github/workflows/dotnet-sonar-ci.yml@main
    with:
      sonar-project-key: mon-org_mon-api
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      SONAR_ORGANIZATION_KEY: ${{ secrets.SONAR_ORGANIZATION_KEY }}
```

### Inputs

| Input | Obligatoire | Défaut | Description |
| --- | --- | --- | --- |
| `sonar-project-key` | oui | | Clé du projet sur SonarCloud |
| `dotnet-version` | non | `10.0.x` | Version du SDK .NET |

### Secrets

| Secret | Description |
| --- | --- |
| `SONAR_TOKEN` | Token SonarCloud |
| `SONAR_ORGANIZATION_KEY` | Clé de l'organisation SonarCloud |

---

## CD : build, push et déploiement

Le workflow a deux jobs.

**`build` : Version, build & push**

1. Validation de la config de déploiement. Si `deploy: true`, il faut `watchtower-url` (en `https://`) et le secret `WATCHTOWER_TOKEN`. Le job échoue ici, avant d'avoir poussé quoi que ce soit.
2. Calcul de la prochaine version (voir [Versionnement](#versionnement)).
3. Build de l'image Docker (`Dockerfile` à la racine par défaut, contexte `.`).
4. Scan Trivy de l'image. Le job échoue sur les vulnérabilités `CRITICAL,HIGH` qui ont un correctif disponible.
5. Création et push du tag git `vX.Y.Z`.
6. Login sur le registry puis push de l'image avec deux tags : `X.Y.Z` et `deploy-tag` (`latest` par défaut).

**`deploy` : Deploy with Watchtower** (seulement si `deploy: true`)

1. `POST {watchtower-url}/v1/update` avec le token en Bearer. Watchtower re-pull l'image et recrée le container.

Il n'y a pas de vérification de santé après la mise à jour, voir [Le déploiement n'est pas vérifié](#le-déploiement-nest-pas-vérifié).

Un seul déploiement à la fois par repo (`concurrency`), sans annulation d'un déploiement en cours.

### Appel de la CD depuis un projet

`.github/workflows/cd.yaml` dans le repo de l'API :

```yaml
name: CD

on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      bump:
        description: "Type de version"
        type: choice
        options: [auto, patch, minor, major]
        default: auto

jobs:
  cd:
    # Évite de builder, taguer et déployer une branche de feature depuis le bouton "Run workflow"
    if: github.ref == 'refs/heads/main'
    uses: <org>/shared-workflows/.github/workflows/dotnet-cd.yaml@main
    permissions:
      contents: write        # pousse le tag git
      pull-requests: read    # retrouve la PR liée au commit
    with:
      registry: registry.mycompany.com
      watchtower-url: https://watchtower.mycompany.com
      bump: ${{ inputs.bump || 'auto' }}
    secrets: inherit
```

`secrets: inherit` transmet `REGISTRY_USERNAME`, `REGISTRY_TOKEN` et `WATCHTOWER_TOKEN`, à condition qu'ils existent dans les secrets du repo ou de l'organisation.

### Inputs de la CD

| Input | Obligatoire | Défaut | Description |
| --- | --- | --- | --- |
| `registry` | oui | | Hôte du registry, avec le port si besoin (`registry.mycompany.com:5000`) |
| `image-name` | non | `owner/repo` en minuscules | Nom de l'image, sans le registry |
| `dockerfile` | non | `Dockerfile` | Chemin du Dockerfile |
| `scan` | non | `true` | Scan Trivy avant le push |
| `scan-severity` | non | `CRITICAL,HIGH` | Sévérités qui font échouer le scan |
| `bump` | non | `auto` | `auto`, `major`, `minor` ou `patch` |
| `deploy` | non | `true` | Déployer après le push |
| `watchtower-url` | si `deploy` | | URL HTTPS de l'API Watchtower |
| `deploy-tag` | non | `latest` | Tag mobile que le container exécute sur le serveur |

Pour pousser l'image sans déployer, passe `deploy: false` : `watchtower-url` et `WATCHTOWER_TOKEN` ne sont alors plus nécessaires.

### Le déploiement n'est pas vérifié

La CD ne teste pas l'API après la mise à jour : les runners GitHub sont sur Internet et ne peuvent pas joindre une API privée. Un run vert signifie donc seulement que l'image est poussée et que Watchtower a répondu 200. Watchtower répond 200 même quand aucun container n'a été mis à jour (container absent, arrêté, sans le label `watchtower.enable`, ou sur un autre tag que `deploy-tag`).

Pour savoir si la nouvelle version tourne :

- consulte les logs de Watchtower sur le serveur (`docker logs watchtower`) ;
- configure les notifications de Watchtower (Teams, Slack, email...) pour être prévenu à chaque mise à jour ;
- vérifie l'image du container (`docker ps`) ou la version exposée par ton API.

### Secrets de la CD

| Secret | Obligatoire | Description |
| --- | --- | --- |
| `REGISTRY_USERNAME` | oui | Login du compte du registry utilisé pour pousser l'image. Ce n'est pas l'hôte du registry (`zot.logards.fr`), qui se passe dans l'input `registry`. |
| `REGISTRY_TOKEN` | oui | Token (ou mot de passe) de ce compte, passé à `docker login` comme mot de passe |
| `WATCHTOWER_TOKEN` | si `deploy` | Token Bearer de l'API HTTP de Watchtower |

---

## Préparer le serveur (une seule fois)

Watchtower **met à jour** des containers existants, il n'en crée jamais. Le premier déploiement se fait donc à la main, puis la CD prend le relais.

### 1. Compose sur le serveur

```yaml
services:
  api:
    image: registry.mycompany.com/<owner>/<repo>:latest   # même tag que deploy-tag
    restart: unless-stopped
    labels:
      - com.centurylinklabs.watchtower.enable=true        # si Watchtower tourne avec --label-enable

  watchtower:
    image: containrrr/watchtower
    restart: unless-stopped
    environment:
      - WATCHTOWER_HTTP_API_UPDATE=true
      - WATCHTOWER_HTTP_API_TOKEN=<le même que le secret WATCHTOWER_TOKEN>
      - WATCHTOWER_LABEL_ENABLE=true
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /root/.docker/config.json:/config.json:ro         # credentials pour pull sur le registry privé
```

Puis `docker compose up -d`.

À respecter :

- Le tag de l'image dans le compose doit être celui de `deploy-tag`. Watchtower ne re-pull que le tag avec lequel le container a été créé.
- L'API de Watchtower doit être joignable en **HTTPS** depuis GitHub (reverse proxy devant le port 8080). Le workflow refuse une URL `http://`.
- Le serveur doit pouvoir pull sur Zot, d'où le `config.json` issu d'un `docker login`.
- Watchtower ignore les containers arrêtés par défaut.

### 2. Zot

- Activer les API keys dans la config (`"apikey": true` dans `http.auth`), puis créer une clé pour l'utilisateur de la CI (UI Zot ou endpoint `/zot/auth/apikey`).
- Donner à cet utilisateur les droits `create` et `update` sur le repo concerné dans `accessControl`, sinon le push est refusé.
- Tester à la main avant le premier run : `docker login registry.mycompany.com -u <user> -p <apikey>`.

### 3. Secrets GitHub

Dans les secrets du repo ou de l'organisation : `REGISTRY_USERNAME`, `REGISTRY_TOKEN`, `WATCHTOWER_TOKEN` (et `SONAR_TOKEN`, `SONAR_ORGANIZATION_KEY` pour la CI).

---

## Lancer la CI/CD à la main

Il faut que le workflow du projet déclare `workflow_dispatch:` (c'est le cas dans les exemples ci-dessus).

1. Onglet **Actions** du repo.
2. Choisir le workflow (CI ou CD) dans la liste de gauche.
3. **Run workflow**, choisir la branche, et pour la CD le type de version.
4. **Run workflow**.

Le sélecteur de branche propose toutes les branches. Le `if: github.ref == 'refs/heads/main'` de l'exemple CD empêche de déployer autre chose que `main`. Sans lui, un run manuel sur une branche de feature la buildera, la taguera et la déploiera.

---

## Versionnement

Chaque run de la CD crée un tag `vX.Y.Z` à partir du dernier tag `v*` (ou `v0.0.0` s'il n'y en a pas). Le type d'incrément est déterminé par l'input `bump` :

| `bump` | Effet |
| --- | --- |
| `auto` (défaut) | Lit `#major`, `#minor` ou `#patch` dans le **titre de la PR** liée au commit (insensible à la casse). Sans mot-clé, ou sans PR liée, c'est `patch`. |
| `major` | `1.2.3` → `2.0.0`, le titre de la PR est ignoré |
| `minor` | `1.2.3` → `1.3.0`, le titre de la PR est ignoré |
| `patch` | `1.2.3` → `1.2.4`, le titre de la PR est ignoré |

Exemple de titre de PR : `feat: nouvelle route de recherche #minor`.

Un lancement manuel n'a pas de PR liée : sans `bump` explicite, c'est toujours un `patch`.

---

## Dépannage

| Symptôme | Cause probable |
| --- | --- |
| Échec immédiat, `deploy is enabled but these are missing: ...` | `watchtower-url` ou le secret `WATCHTOWER_TOKEN` n'est pas fourni. Passe-les, ou mets `deploy: false`. |
| `watchtower-url must start with https://` | Le token Bearer ne doit pas circuler en clair. Utilise une URL HTTPS. |
| `bump must be auto, major, minor or patch` | Valeur de `bump` invalide. |
| Échec du step Trivy | Vulnérabilité `CRITICAL` ou `HIGH` avec correctif disponible. Mets à jour l'image de base ou les dépendances, ou ajuste `scan-severity`. |
| Échec du push d'image | `REGISTRY_USERNAME` / `REGISTRY_TOKEN` invalides, ou droits `create` / `update` manquants sur Zot. |
| `Trigger Watchtower update` en erreur | Watchtower arrêté ou injoignable, `WATCHTOWER_HTTP_API_UPDATE` non activé, ou token incorrect. Le step retente 3 fois avant d'échouer. |
| Run vert mais l'ancienne version tourne toujours | Watchtower répond 200 même quand rien n'a été mis à jour. Le container est absent ou arrêté, n'a pas le label `watchtower.enable`, ou tourne sur un autre tag que `deploy-tag`. Voir [Le déploiement n'est pas vérifié](#le-déploiement-nest-pas-vérifié). |
| Le tag git existe déjà | Deux runs ont calculé la même version. Relance le run. |
