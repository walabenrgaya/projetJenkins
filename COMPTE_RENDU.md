# TP3 – Jenkins ⇆ Docker (Build → Test → Push → Deploy)

**Étudiant·e :** Wala Ben Rgaya – 3ème LIG – ISG Sousse
**Dépôt :** https://github.com/walabenrgaya/projetJenkins
**Image Docker Hub :** https://hub.docker.com/r/walabenrgaya/monapp

## 1. Installation (Windows 11 Pro)

| Prérequis | État |
|---|---|
| WSL2 (`wsl --install`) | WSL 3.0.5, distribution `docker-desktop` en version 2 |
| Hyper-V + Virtual Machine Platform | activés |
| Docker Desktop (moteur WSL2) | Docker 29.9.2, Docker Compose v5.6.1 |
| Jenkins | 2.580.2 (service Windows), plugins Pipeline, Git, Credentials Binding, Timestamper, GitHub |

Vérification : `docker version`, `docker compose version`, `docker run --rm nginx:alpine nginx -v` → OK.

Accès de Jenkins à Docker : le pipeline exécute `whoami` puis `docker version` avec succès.
Ici le service Jenkins tourne sous `LocalSystem`, qui a déjà accès au moteur Docker Desktop.

## 2. Ports utilisés sur la machine

| Port | Utilisation |
|---|---|
| 8080 | déjà pris par Oracle Database (XDB) |
| 8081 | Jenkins |
| 8082 | conteneur du Smoke Test (au lieu de 8081 dans l'énoncé) |
| 8090 | application déployée par docker compose (au lieu de 8080) |

## 3. Fichiers du projet

- `index.html`, `style.css` : application web du TP2
- `Dockerfile` : image `nginx:alpine` + fichiers statiques
- `.dockerignore`
- `Jenkinsfile` : pipeline Checkout → Docker Build → Smoke Test → Push → Deploy
- `docker-compose.yml` : déploiement de `walabenrgaya/monapp:latest`

## 4. Pipeline Jenkins (job `TP3-Docker`, « Pipeline script from SCM »)

- **Checkout** : `checkout scm` du dépôt GitHub, branche `main`
- **Docker Build** : `docker build -t walabenrgaya/monapp:build-N .`
- **Smoke Test** : lance le conteneur sur 8082, vérifie `HTTP/1.1 200 OK` avec curl, puis le supprime
- **Push** : `docker login` avec le credential `dockerhub-creds` (access token Docker Hub), push des tags `build-N` et `latest`
- **Deploy (Staging)** : `docker compose pull` + `docker compose up -d`, puis vérification HTTP 200 sur 8090

Correction apportée au Jenkinsfile de l'énoncé : sous Windows, `echo %PASS% | docker login`
envoie le mot de passe suivi d'un espace (l'espace avant `|`). On écrit donc
`echo %PASS%| docker login ... || exit /b 1`, qui arrête aussi le stage si la connexion échoue.

## 5. Webhook GitHub (CI réactive)

- Jenkins : « GitHub hook trigger for GITScm polling » coché dans le job.
- Jenkins tourne sur `localhost` : il a été exposé temporairement avec un tunnel
  `cloudflared tunnel --url http://localhost:8081` (URL `https://….trycloudflare.com`).
- GitHub : Settings → Webhooks → `https://….trycloudflare.com/github-webhook/`, événement `push`.
- Ping et push livrés avec HTTP 200.

## 6. Résultats

| Build | Déclenchement | Résultat |
|---|---|---|
| #1 | manuel | Build + Smoke Test OK, Push KO (credential absent) |
| #2, #3 | manuel | Push KO (mot de passe refusé par Docker Hub) |
| #4 | manuel | **SUCCESS** (avec un access token Docker Hub) |
| #5 | **push GitHub (webhook)** | **SUCCESS** : Build, Test, Push, Deploy |

Log du build #5 (extraits) :

```
Started by GitHub push by walabenrgaya
HTTP/1.1 200 OK                       (smoke test, port 8082)
Login Succeeded
build-5: digest: sha256:562792ba...
latest: digest: sha256:562792ba...
Container monapp-web-1 Started
HTTP/1.1 200 OK                       (déploiement, port 8090)
Build+Test+Push+Deploy OK
Finished: SUCCESS
```

L'application déployée est accessible sur http://localhost:8090.
