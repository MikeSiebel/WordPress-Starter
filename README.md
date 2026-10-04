# WordPress Starter

Reusable WordPress starter template for local development with Docker and deployment through GitHub Actions.

## Project structure

```text
.
├── .github/
│   └── workflows/
│       └── cicd.yaml
├── .srv/
├── mu-plugins/
├── plugins/
├── themes/
├── .gitignore
├── docker-compose.yml
├── README.md
└── wp-version-control.cfg
```

## Requirements

* Docker Desktop
* WSL 2
* Git

## Local development

Start the containers:

```powershell
docker compose up -d
```

WordPress:

http://localhost:8000

phpMyAdmin:

http://localhost:8181

Check containers:

```powershell
docker compose ps
```

Stop the project:

```powershell
docker compose down
```

## WordPress version

The WordPress version used for deployment is defined in:

```text
wp-version-control.cfg
```

The current version is WordPress 7.1.

## GitHub Actions deployment

The workflow is located at:

```text
.github/workflows/cicd.yaml
```

Deployment is triggered by:

* push to `dev` branch → development server
* published GitHub Release → production server

### Required GitHub Secrets

The repository requires:

```text
SSH_HOST
SSH_USER
SSH_PASSWORD
SSH_PORT
DEV_DEPLOY_PATH
PROD_DEPLOY_PATH
```

`DEV_DEPLOY_PATH` is used for deployments from the `dev` branch.

`PROD_DEPLOY_PATH` is used for published GitHub Releases.

## Plugins and themes

Custom WordPress plugins belong in:

```text
plugins/
```

Custom themes belong in:

```text
themes/
```

Must-use plugins belong in:

```text
mu-plugins/
```

The starter repository contains only the standard `index.php` protection files. Project-specific plugins, themes and mu-plugins should be added when creating a new project.

## Creating a new project

Use this repository as a GitHub Template Repository.

After creating a new repository from the template:

1. Clone the new repository locally.
2. Configure the project-specific WordPress plugins and theme.
3. Configure GitHub Actions Secrets.
4. Set the development and production deployment paths.
5. Start the local environment with Docker.

## Important

Do not commit:

* database files
* WordPress uploads
* local `.srv` data
* passwords or SSH credentials
* generated dependency directories such as `node_modules`

Project-specific configuration should be added only after creating a new repository from this template.
