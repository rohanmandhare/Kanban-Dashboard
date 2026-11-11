# Project-02 Evidence Checklist

Do not mark an item complete until the output or screenshot exists.

- [ ] `npm ci` successful
- [ ] Local app opened in browser
- [ ] `npm run build` successful; output directory confirmed
- [ ] Docker image built successfully
- [ ] `docker inspect` confirms non-root user (UID 101)
- [ ] Container reports `healthy`
- [ ] `/healthz` returns HTTP 200
- [ ] Docker Hub contains a versioned image tag
- [ ] Jenkins credentials configured; token not printed in logs
- [ ] Jenkins pipeline passes checkout/build/security/push/deploy/validation
- [ ] GitHub push webhook triggers a build
- [ ] EC2 app URL works from the intended network
- [ ] `docker ps`, `docker logs`, `docker inspect` captured
- [ ] Failed deployment and rollback demonstrated safely on a test branch
- [ ] AWS region, EC2 public IP/DNS, Docker Hub URL and GitHub URL recorded
