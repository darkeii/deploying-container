# Deploying a Static Website in a Docker Container on a VPS

A reusable template for hosting any plain HTML/CSS/JS site on a VPS using
Docker + nginx (reverse proxy) + Let's Encrypt (HTTPS). Follow this for every
new static site you want to host — just swap the project name.

## Prerequisites (one-time, per VPS)

These only need to be done once per server, not per project:

- Docker installed (`curl -fsSL https://get.docker.com | sudo sh`)
- nginx installed (`sudo apt install nginx`)
- certbot installed (`sudo apt install certbot python3-certbot-nginx`)
- ufw configured to allow 22, 80, 443
- A domain (or subdomain) pointed at the VPS's public IP via a DNS A record

## Steps for each new project

Replace `PROJECT_NAME` and `DOMAIN` below with your actual values throughout.

### 1. Create the project folder and clone the repo

\`\`\`bash
sudo mkdir -p /opt/PROJECT_NAME
sudo chown $USER:$USER /opt/PROJECT_NAME
cd /opt/PROJECT_NAME
git clone <your-repo-url> .
\`\`\`

### 2. Write the Dockerfile

\`\`\`bash
nano /opt/PROJECT_NAME/Dockerfile
\`\`\`

\`\`\`dockerfile
FROM nginx:alpine
COPY . /usr/share/nginx/html
EXPOSE 80
\`\`\`

Assumes `index.html` sits at the repo root. If your build output lives in a
subfolder (`dist/`, `build/`, `public/`), change the `COPY` source accordingly.

### 3. Add a .dockerignore

\`\`\`bash
nano /opt/PROJECT_NAME/.dockerignore
\`\`\`

\`\`\`
.git
.gitignore
README.md
\`\`\`

### 4. Build and run the container

Pick a port that isn't already used by another project on this VPS (keep a
running list — e.g. 3000, 3001, 3002...).

\`\`\`bash
cd /opt/PROJECT_NAME
docker build -t PROJECT_NAME .
docker run -d --name PROJECT_NAME --restart unless-stopped -p 127.0.0.1:PORT:80 PROJECT_NAME
\`\`\`

Verify:

\`\`\`bash
docker ps
curl -I http://127.0.0.1:PORT
\`\`\`

### 5. nginx server block

\`\`\`bash
sudo nano /etc/nginx/sites-available/DOMAIN
\`\`\`

\`\`\`nginx
server {
    listen 80;
    listen [::]:80;
    server_name DOMAIN;

    location / {
        proxy_pass http://127.0.0.1:PORT;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
\`\`\`

Enable it:

\`\`\`bash
sudo ln -s /etc/nginx/sites-available/DOMAIN /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
\`\`\`

### 6. Get HTTPS

\`\`\`bash
sudo certbot --nginx -d DOMAIN
\`\`\`

Certbot auto-edits the server block to add the SSL config and an HTTP→HTTPS
redirect, and sets up auto-renewal.

### 7. Deploy script (for future updates)

\`\`\`bash
nano /opt/PROJECT_NAME/deploy.sh
\`\`\`

\`\`\`bash
#!/bin/bash
set -e
cd /opt/PROJECT_NAME
echo "Pulling latest code..."
git pull
echo "Rebuilding image..."
docker build -t PROJECT_NAME .
echo "Replacing container..."
docker stop PROJECT_NAME || true
docker rm PROJECT_NAME || true
docker run -d --name PROJECT_NAME --restart unless-stopped -p 127.0.0.1:PORT:80 PROJECT_NAME
echo "Done. Live at https://DOMAIN"
\`\`\`

\`\`\`bash
chmod +x /opt/PROJECT_NAME/deploy.sh
echo "deploy.sh" >> /opt/PROJECT_NAME/.gitignore
\`\`\`

From now on, after pushing changes to GitHub, update the live site with:

\`\`\`bash
/opt/PROJECT_NAME/deploy.sh
\`\`\`

### 8. (Optional) Auto-deploy on every push via GitHub Actions

Generate a dedicated deploy key on the VPS (never reuse your personal SSH key):

\`\`\`bash
ssh-keygen -t ed25519 -f ~/.ssh/github_actions_deploy -N ""
cat ~/.ssh/github_actions_deploy.pub >> ~/.ssh/authorized_keys
cat ~/.ssh/github_actions_deploy
\`\`\`

Copy the private key output into the repo's GitHub secrets:
**Settings → Secrets and variables → Actions**
- `DRAGON_HOST` = VPS public IP
- `DRAGON_SSH_KEY` = the private key

Then add `.github/workflows/deploy.yml`:

\`\`\`yaml
name: Deploy to staging
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: SSH and deploy
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.DRAGON_HOST }}
          username: ubuntu
          key: ${{ secrets.DRAGON_SSH_KEY }}
          script: /opt/PROJECT_NAME/deploy.sh
\`\`\`

Every push to `main` now redeploys automatically.

## Notes / gotchas learned the hard way

- `mkdir -p /opt/PROJECT_NAME` needs `sudo` — `/opt` is root-owned by default.
  Run `sudo chown $USER:$USER /opt/PROJECT_NAME` right after, or every
  subsequent command in that folder needs `sudo` too.
- If the site is behind Cloudflare (orange-cloud proxied), add a
  `set_real_ip_from` + `real_ip_header CF-Connecting-IP` block in nginx
  (e.g. `/etc/nginx/conf.d/cloudflare.conf`) so upstream apps see the real
  visitor IP instead of Cloudflare's edge IP. Without this, anything that
  does IP-based logic (rate limiting, brute-force protection, geolocation)
  will misbehave.
- `docker run` with a name that's already taken will fail — always
  `docker stop` + `docker rm` the old container before starting a new one
  with the same name (the deploy script above handles this).
- Each project gets its own port on `127.0.0.1` — never expose app ports
  directly to the internet; only nginx should be reachable on 80/443.





