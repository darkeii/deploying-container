# Deploying container block

## 1. Create the project folder and clone the repo
```
sudo mkdir -p /opt/PROJECT_NAME
sudo chown $USER:$USER /opt/PROJECT_NAME
cd /opt/PROJECT_NAME
git clone <your-repo-url>
```
**2. Write the Dockerfile**
```
nano /opt/PROJECT_NAME/Dockerfile
```
```
FROM nginx:alpine
COPY . /usr/share/nginx/html
EXPOSE 80
```


