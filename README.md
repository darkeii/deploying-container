# Deploying container block

## 1. Create the project folder and clone the repo
```
sudo mkdir -p /opt/PROJECT_NAME
sudo chown $USER:$USER /opt/PROJECT_NAME
cd /opt/PROJECT_NAME
git clone <your-repo-url>
```
## 2. Write the Dockerfile
```
nano /opt/PROJECT_NAME/Dockerfile
```
```
FROM nginx:alpine
COPY . /usr/share/nginx/html
EXPOSE 80
```
## 3. Add a .dockerignore
```
nano /opt/PROJECT_NAME/.dockerignore
```
```
.git
.gitignore
README.md
```
## 4. Build and run the container

Pick a port that isn't already used by another project on this VPS (keep a running list — e.g. 3000, 3001, 3002...).
Ports in use : 3000(portfolio), 3001(NextCloud)
```
cd /opt/PROJECT_NAME
docker build -t PROJECT_NAME .
docker run -d --name PROJECT_NAME --restart unless-stopped -p 127.0.0.1:PORT:80 PROJECT_NAME
```
**Verify:**
```
docker ps
curl -I http://127.0.0.1:PORT
```
## 5. nginx server block
```
sudo nano /etc/nginx/sites-available/DOMAIN
```
```
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
```
**Enable it:**
```
sudo ln -s /etc/nginx/sites-available/DOMAIN /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```
## 6. Get HTTPS** ( Using certbot )
```
sudo certbot --nginx -d DOMAIN
```
>> certbot will auto-edit the container block to add the SSL config and an HTTP→HTTPS redirect.

## 7. Deploy script (for future updates)
>> Can skip this step if you don't need to change the files/code inside the block frequently.
    You can manually run the "deploy.sh" commands.

```
nano /opt/PROJECT_NAME/deploy.sh
```
```
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
```
## Make this file executable:
```
chmod +x /opt/PROJECT_NAME/deploy.sh
echo "deploy.sh" >> /opt/PROJECT_NAME/.gitignore
```
### From now on, after pushing changes to GitHub, update the live site with:
```
/opt/PROJECT_NAME/deploy.sh
```

#### NOTE: You can also setup, auto deploy on every push on github via generating a dedicated deploy key on VPS (I have not tried it yet).


# Self notes
```/opt``` is a Linux convention for "third-party applications that don't belong to the base OS", all the deployed projects can be accessed here. Separate from your ```home``` folder. It's owned by ```root``` by default, so ```mkdir``` needs ```sudo```. The ```chown``` line hands ownership to your own user so you don't need ```sudo``` for every command after this.


```FROM nginx:alpine``` says "start from a tiny, pre-built Linux image that already has nginx installed" (Alpine is a minimal Linux distro, keeps the image small).
```COPY . /usr/share/nginx/html``` copies your project's files into nginx's default folder for serving web pages inside that container.
```EXPOSE 80``` documents that the container listens on port 80 internally.


```docker build``` actually executes the Dockerfile's steps and produces a reusable image, tagged with a name (```-t```).
```docker run``` starts an actual running container from that image: ```-d``` runs it in the background, ```--name``` gives it a friendly name for later reference, ```--restart unless-stopped``` means it auto-restarts if it crashes or the VPS reboots, and ```-p 127.0.0.1:PORT:80``` maps the container's internal port 80 to a specific port on the host — but only on 127.0.0.1 (localhost), meaning it's not reachable from the internet directly. That's intentional — only nginx should be the public-facing entry point.


This tells the host's nginx: "when a request comes in for this specific domain, forward it to the container listening on that local port." The proxy_set_header lines pass along useful info (the real hostname, the visitor's real IP, etc.) to the container, since otherwise the container would just see nginx as the source of every request. ln -s (symlink) into sites-enabled is how nginx actually activates a config file — configs in ```sites-available``` alone don't do anything until linked.







