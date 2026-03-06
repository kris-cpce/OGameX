# Deploying changes from your development machine to the server

This guide shows a simple, repeatable workflow for pushing code from your dev machine (local workspace) to the remote server where OGameX runs under Docker Compose. It's written for a development environment — it prioritizes clarity and safety over complex production practices.

Assumptions
- You have a fork/remote repository and use branches (for example `reskin`).
- Your remote server has the project checked out in `/opt/OGameX` (adjust paths if different).
- The server runs Docker and `docker compose` and uses the same Compose files you have locally.
- You have SSH access to the server.

1) Work locally and prepare your change
- Create a feature branch (if you haven't already):

```bash
# from your local repo root
git checkout -b reskin-small
# edit files, test locally
```

- Stage and commit your changes:

```bash
git add -A
git commit -m "Small reskin: header & CSS override"
```

- Push the branch to your fork / remote:

```bash
git push origin reskin-small
```

2) Optional: review & merge
- If you use PRs, open a pull request on GitHub and merge when ready.
- Alternatively you can deploy directly from the branch.

3) SSH to the server and update the code
- SSH in and go to the project directory:

```bash
ssh youruser@your.server.ip
cd /opt/OGameX
```

- Fetch and check out the branch you pushed:

```bash
# fetch updates
git fetch origin
# switch to the branch (this will update files to the selected revision)
git checkout reskin-small
# Or pull latest on the deployed branch:
# git pull origin reskin-small
```

If the server has local changes that should be preserved, stash them first or use a safer update strategy (ask if you need that workflow).

4) Build backend dependencies and frontend assets on the server (recommended)
- There are two approaches: build inside containers (recommended) or build locally and commit compiled assets.

A) Build inside Docker (recommended):

```bash
# Ensure containers are up (this uses your compose files)
docker compose up -d

# Install PHP/composer dependencies inside the app container
docker compose exec ogamex-app composer install --no-interaction --prefer-dist

# Generate/app key and run migrations (if you added database changes)
docker compose exec ogamex-app php artisan key:generate --force
docker compose exec ogamex-app php artisan migrate --force

# Build frontend assets using the persistent frontend service (if you configured one)
docker compose up -d ogamex-frontend
docker compose exec ogamex-frontend npm ci
docker compose exec ogamex-frontend npm run build

# If your frontend is one-shot (no persistent service), run:
# docker compose run --rm ogamex-frontend npm ci
# docker compose run --rm ogamex-frontend npm run build
```

B) Build locally and commit (alternative)
- Run `npm ci` and `npm run build` locally, commit the generated `public/` assets, push, then pull on the server. This avoids installing Node on the server but means compiled assets are stored in git.

5) Restart web services so Nginx/PHP use the new code

```bash
docker compose restart ogamex-app ogamex-webserver
# optional: docker compose logs -f ogamex-app
```

6) Troubleshooting quick checks
- If you see 502 from Nginx, ensure PHP-FPM is up in the app container:

```bash
docker compose ps
docker compose logs --tail=200 ogamex-app
docker compose logs --tail=200 ogamex-webserver
```

- If Laravel cannot write logs/files, fix permissions on the host for `storage` and `bootstrap/cache`. Find the php-fpm UID inside the container and chown on the host:

```bash
# inside container - find likely user (run on server)
docker compose exec ogamex-app bash -lc "id -u www-data 2>/dev/null || id -u www 2>/dev/null || id -u nginx 2>/dev/null || id -u  www-data || true"

# Example (if UID:GID are 33:33):
sudo chown -R 33:33 /opt/OGameX/storage /opt/OGameX/bootstrap/cache
sudo chmod -R 775 /opt/OGameX/storage /opt/OGameX/bootstrap/cache

# With SELinux enabled (Rocky/CentOS):
sudo semanage fcontext -a -t httpd_sys_rw_content_t '/opt/OGameX/storage(/.*)?'
sudo restorecon -Rv /opt/OGameX/storage /opt/OGameX/bootstrap/cache
```

7) Rollback (simple)
- If the deploy introduces a problem, you can quickly go back to the previous commit:

```bash
# see recent commits
git log --oneline -n 5
# reset to previous commit (example: use the previous commit SHA)
git checkout <previous-commit-sha>
# then restart services as above
```

8) Helpful tips and best practices for a beginner
- Use branches for each feature/fix and keep `main`/`master` stable.
- Test locally before pushing. If you can run the app and the web UI locally in Docker, do so.
- Keep `public/` compiled assets out of Git for development; build them on the server inside the `ogamex-frontend` container as shown.
- Use `docker compose pull` before `docker compose up -d --build` when you rely on updated images.

9) Example one-line deploy (quick)

```bash
# On server, inside /opt/OGameX
git fetch origin && git checkout reskin-small && git pull origin reskin-small && 
# build deps & assets
docker compose up -d && 

docker compose exec ogamex-app composer install --no-interaction --prefer-dist && 

docker compose exec ogamex-frontend npm ci && docker compose exec ogamex-frontend npm run build && 

docker compose restart ogamex-app ogamex-webserver
```

If you'd like, I can:
- Create a simple deploy script (`deploy.sh`) in the repo that automates these steps on the server.
- Add a GitHub Actions workflow to build assets and push a release image (more advanced).

---

If you want the `deploy.sh` script now, tell me and I will scaffold it and explain how to use it.