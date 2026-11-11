# Project-02 — Kanban Dashboard CI/CD with Docker, Jenkins and AWS EC2

This starter packages the supplied React/Vite dashboard into a non-root Docker image and deploys versioned images through Jenkins.

## Architecture

GitHub → Jenkins Pipeline → Docker build → Docker Hub (versioned tag) → EC2 Docker deployment → health validation

**Assumption to verify:** the repository has `package.json`, `package-lock.json`, and a Vite build that writes to `dist/`. Check this in VS Code before running the pipeline. Never commit secrets.

## 1. Validate locally (Windows / VS Code)

Install Git, Node.js LTS and Docker Desktop. In PowerShell:

```powershell
git clone https://github.com/Vennilavanguvi/kanban-dashboard.git
cd kanban-dashboard
npm ci
npm run dev
```

Open the local URL printed by Vite. Test the production build:

```powershell
npm run build
Test-Path .\dist\index.html
```

Record the framework, build command, output directory and required environment variables. If the repository uses a different output directory or has no lockfile, adapt the Dockerfile first.

## 2. Add the files

Copy `Dockerfile`, `.dockerignore`, `nginx.conf`, `Jenkinsfile`, and `README.md` to the root of the cloned repository. Replace `YOUR_DOCKERHUB_USERNAME` in `Jenkinsfile`, then commit and push.

Build locally:

```powershell
docker build -t kanban-dashboard:test .
docker run --rm -d --name kanban-test -p 8080:8080 kanban-dashboard:test
curl.exe -i http://localhost:8080/healthz
docker inspect kanban-test --format "{{.Config.User}}"
docker inspect kanban-test --format "{{.State.Health.Status}}"
docker logs kanban-test
docker rm -f kanban-test
```

The image runs as UID 101 and listens on container port 8080. `/healthz` should return HTTP 200 and `ok`.

## 3. AWS EC2 prerequisites

Use a supported Ubuntu LTS EC2 instance with enough memory/disk for Jenkins and Docker builds. Install Docker and Jenkins using the current official instructions for that Ubuntu version. Ensure the Jenkins agent can run `docker`, `git`, and `curl`.

Security group guidance:
- SSH (22): only your current public IP/CIDR.
- Jenkins (8080): restrict to your IP or use an SSH tunnel/VPN; do not expose to everyone.
- App (80): open only to the intended audience, or restrict to your IP for a demo.

Jenkins access to the Docker daemon is effectively root-level access. Understand the risk before adding the Jenkins user to the Docker group, and keep Jenkins patched.

**Deployment target:** this Jenkinsfile deploys to the Docker host running the pipeline. If Jenkins and the application are on the same EC2, it deploys there. If they are separate EC2 instances, adapt the deployment stage for SSH or an agent on the application host.

## 4. Configure Jenkins

Install Pipeline, Git and Credentials Binding plugins. Docker Pipeline is optional because this Jenkinsfile uses Docker CLI commands.

1. Create a Pipeline job.
2. Choose **Pipeline script from SCM** → Git.
3. Repository: `https://github.com/Vennilavanguvi/kanban-dashboard.git`
4. Branch: `*/main` (change if the repository uses another default branch).
5. Script path: `Jenkinsfile`.
6. Create a Jenkins username/password credential with ID `dockerhub-creds`. Username is your Docker Hub username; password is a Docker Hub access token.
7. Replace `YOUR_DOCKERHUB_USERNAME` in the Jenkinsfile and commit it.
8. Ensure the agent can access the Docker daemon.

## 5. GitHub webhook

In GitHub repository **Settings → Webhooks → Add webhook**:
- Payload URL: `http(s)://YOUR_JENKINS_HOST/github-webhook/`
- Content type: `application/json`
- Event: push event

Enable the GitHub push trigger in the Jenkins job. GitHub must be able to reach the webhook endpoint. If Jenkins is private, use a secure tunnel or polling for the lab rather than exposing Jenkins broadly.

## 6. Pipeline stages

- Checkout source; derive a build-number + Git-SHA tag.
- Build and inspect image.
- Check the configured runtime user is non-root.
- Push the versioned image to Docker Hub using Jenkins credentials.
- Pull and deploy the image on the same Docker host as Jenkins.
- Apply memory/CPU limits.
- Wait for Docker health status and curl `/healthz`.
- Attempt to restore the previous image if deployment or validation fails.

This is a **stop/start deployment with rollback**, not guaranteed zero-downtime blue-green deployment. A short interruption may occur during replacement. True blue-green needs a proxy/load balancer or separate target instances plus a traffic-switch step.

## 7. Evidence checklist

Capture screenshots/output for:
1. Local application running in browser.
2. Successful `npm run build`.
3. Successful Docker build.
4. `docker inspect` showing `User=101`.
5. Container health status `healthy`.
6. Docker Hub repository with versioned tags.
7. Jenkins console output showing stages.
8. GitHub webhook delivery and auto-triggered Jenkins build.
9. `docker ps`, `docker logs`, and `docker inspect`.
10. A failed deployment and rollback demonstrated safely on a test branch.
11. Application URL response from curl/browser.

EC2 commands:

```bash
docker ps
docker logs --tail 100 kanban-dashboard
docker inspect kanban-dashboard --format 'User={{.Config.User}} Health={{.State.Health.Status}}'
curl -i http://127.0.0.1/healthz
docker image ls
```

## Security notes

- Never commit `.env`, access keys, registry tokens or private keys.
- Store registry credentials in Jenkins Credentials.
- Frontend build variables are shipped to browsers; never embed runtime secrets in Vite variables.
- Pin and review base-image versions and scan images (for example with Trivy) before production use.
- Keep Jenkins and dependencies patched.

## Troubleshooting

- `npm ci` fails: verify the lockfile matches `package.json`.
- `dist` missing: check the build script/output directory.
- Docker permission denied in Jenkins: check agent permissions and restart Jenkins after group changes.
- Docker Hub push denied: check `dockerhub-creds`, token permissions and username.
- Port 80 already in use: identify the service/container before changing anything.
- Health check fails: inspect `docker logs kanban-dashboard` and `docker inspect`.
- Webhook not triggering: verify URL reachability, delivery status and Jenkins trigger configuration.

## Submission placeholders

- GitHub: `https://github.com/Vennilavangu/kanban-dashboard`
- Docker Hub: `https://hub.docker.com/r/YOUR_DOCKERHUB_USERNAME/kanban-dashboard`
- AWS Region: fill in your actual region
- EC2 Public DNS/IP: fill in after deployment
- Application URL: `http://YOUR_EC2_PUBLIC_IP/`
