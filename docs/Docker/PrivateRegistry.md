---
title: "Secure Docker Registry Setup with TLS Authentication and UI"
date: 2025-04-28
categories:
  - Docker
  - DevOps
  - Security
tags:
  - docker
  - registry
  - tls
  - authentication
  - security
  - registry-ui
authors:
  - ruchany13
summary: "Learn how to set up a secure Docker Registry using TLS, authentication, and a user-friendly UI. Ideal for simple, resource-efficient private registries."
---

# Secure Docker Registry Setup with TLS, Authentication, and UI 🚀

In this blog post, we will set up a secure Docker Registry enhanced with TLS encryption, HTTP authentication, and a user-friendly web interface (**registry-ui**) developed by [Joxit](https://github.com/Joxit/docker-registry-ui/tree/main). Special thanks to Joxit for creating a clean and intuitive UI for Docker registries.

Docker Registry is an open-source solution provided by Docker to store Docker images privately. While alternatives like Harbor offer richer features such as role-based access control, vulnerability scanning, and advanced user management, Docker Registry excels in simplicity and efficiency. It's perfect for lightweight and resource-friendly setups without compromising security.

All the necessary code and configurations used in this blog are available on my [GitHub repository](https://github.com/ruchany13/MyDockerExamples/tree/main/PrivateRegistry).

## 🛠 What You Need

After completing this setup, you'll have:

- A Docker Registry with basic authentication.
- TLS encryption with SSL certificates.
- Docker Registry and UI managed via Docker Compose.
- Ability to manage Docker images through an intuitive web UI and Docker CLI.

## 📁 Project Directory Structure

Before we start, organize your project folder clearly as shown below:

```text title="Folder Structure"
PrivateRegistry/
├── auth/
│   └── registry.passwd
├── nginx/
│   ├── nginx.conf
│   └── ssl/
│       ├── fullchain.pem
│       └── privkey.pem
├── registry_data/
└── docker-compose.yml
```

!!! warning "Disk Space Planning"
    Ensure that the server has **enough disk space** allocated for `registry_data/`, where Docker images will be stored.  
    Docker images, especially in production, can grow significantly over time. Plan your storage accordingly based on expected image size and frequency of updates.

---

## 🏗 Step-by-Step Setup

### Step 1: Create Authentication Credentials 🔒

Create authentication credentials using `htpasswd`. Replace `username` with your desired registry username:

```bash title="Create Credentials"
mkdir auth/
cd auth
htpasswd -Bc registry.passwd username
```

### Step 2: Generate SSL Certificate 🔐

You must secure your Docker Registry with SSL. For this, we will use Certbot to generate a trusted SSL certificate. Example domain: `repo.ruchan.dev` (Replace this with your own domain)

```bash title="Install and Run Certbot"
sudo apt install certbot
sudo certbot certonly --standalone -d repo.ruchan.dev
```

Certbot generates:

- Fullchain certificate: `/etc/letsencrypt/live/repo.ruchan.dev/fullchain.pem`
- Private key: `/etc/letsencrypt/live/repo.ruchan.dev/privkey.pem`

Copy these files into your project's `nginx/ssl/` directory:

```bash title="Copy Certificates"
mkdir -p nginx/ssl/
sudo cp /etc/letsencrypt/live/repo.ruchan.dev/fullchain.pem nginx/ssl/
sudo cp /etc/letsencrypt/live/repo.ruchan.dev/privkey.pem nginx/ssl/
```

!!! note "Self-signed Certificates"
    If using self-signed or untrusted certificates, copy the `fullchain.pem` to your Docker hosts:

    ```bash title="Trust Self-signed Certificates"
    sudo mkdir -p /etc/docker/certs.d/repo.ruchan.dev:443/
    sudo cp fullchain.pem /etc/docker/certs.d/repo.ruchan.dev:443/ca.crt
    sudo systemctl restart docker
    ```

    Trusted certificates generated via Certbot do not require these extra steps.

### Step 3: Docker Compose Configuration 🛠️

Here's the complete `docker-compose.yml` file integrating registry, UI, and Nginx with SSL:

```yaml title="docker-compose.yml"
version: '3'
services:
  registry:
    image: registry:latest
    container_name: registry
    hostname: registry
    restart: always
    environment:
      REGISTRY_AUTH: htpasswd
      REGISTRY_AUTH_HTPASSWD_REALM: Registry-Realm
      REGISTRY_AUTH_HTPASSWD_PATH: /auth/registry.passwd
      REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY: /data
      REGISTRY_STORAGE_DELETE_ENABLED: 'true'
    volumes:
      - ./registry_data:/data
      - ./auth:/auth
    networks:
      - docker

  nginx:
    image: nginx:mainline-alpine
    container_name: nginx_registry
    restart: unless-stopped
    tty: true
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./nginx/ssl/:/etc/nginx/ssl/
    networks:
      - docker
    depends_on:
      - registry

  registry-ui:
    image: joxit/docker-registry-ui:main
    hostname: registry-ui
    restart: always
    environment:
      - SINGLE_REGISTRY=true
      - UI_ROOT_PATH=/ui
      - REGISTRY_TITLE=Docker Registry UI
      - DELETE_IMAGES=true
      - SHOW_CONTENT_DIGEST=true
      - NGINX_PROXY_PASS_URL=http://registry:5000
    container_name: registry-ui
    networks:
      - docker

networks:
  docker:
    driver: bridge
```

!!! warning "Storage Directory Best Practice"
    For production environments, instead of using a **local ./registry_data** folder, consider mounting a **dedicated disk volume** or using **external storage** for `/data`.  
    Example change in `docker-compose.yml`:

    ```yaml
    volumes:
      - /mnt/registry_data:/data
    ```

    This ensures persistence and scalability for large Docker image repositories.

!!! info "Domain Configuration"
    Replace `repo.ruchan.dev` in the above configurations with your own domain.

### Step 4: Nginx Configuration 🌐

Place this configuration into `nginx/nginx.conf` (replace `repo.ruchan.dev` with your domain):

```nginx title="nginx.conf"
events {}
http {
  upstream docker-registry {
    server registry:5000;
  }

  upstream registry-ui {
    server registry-ui:80;
  }

  server {
    listen 443 ssl;
    server_name repo.ruchan.dev;

    ssl_certificate /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;

    location /ui/ {
      proxy_pass http://registry-ui/;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
    }

    location / {
      client_max_body_size 0;

      add_header 'Access-Control-Allow-Origin' 'https://repo.ruchan.dev' always;
      add_header 'Access-Control-Allow-Methods' 'HEAD, GET, OPTIONS, DELETE' always;
      add_header 'Access-Control-Allow-Credentials' 'true' always;
      add_header 'Access-Control-Allow-Headers' 'Authorization, Accept, Cache-Control' always;

      proxy_pass http://docker-registry;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
    }
  }
}
```

!!! warning "UI Path Conflict"
    The UI is mapped under `/ui/`. If you create Docker repositories that start with `/ui/`, it can conflict with the UI route and cause problems.  
    Solutions:
    - Map the UI under a different path (e.g., `/dashboard/`).
    - Host the UI under a different subdomain (e.g., `ui.repo.ruchan.dev`).

Start your services:

```bash title="Start Services"
docker-compose up -d
```

---

## 🎨 Using the Registry UI

Access the Registry UI via:

```
https://repo.ruchan.dev/ui/
```

Here you'll see your Docker images visually.

![Registry UI Screenshot](https://camo.githubusercontent.com/881d8c50cbb52beab2e59e91a7a419aee7df54c94abcbbeb9798476b07f009a7/68747470733a2f2f7261772e6769746875622e636f6d2f4a6f7869742f646f636b65722d72656769737472792d75692f6d61696e2f646f636b65722d72656769737472792d75692e676966)

---

## 🚢 Docker Push/Pull Operations

### Docker Login

```bash title="Login"
docker login repo.ruchan.dev
```

### Push Images

```bash title="Push Image"
docker tag my-image:latest repo.ruchan.dev/my-image:latest
docker push repo.ruchan.dev/my-image:latest
```

### Pull Images

```bash title="Pull Image"
docker pull repo.ruchan.dev/my-image:latest
```

---

## 🎯 Conclusion

Docker Registry combined with authentication, TLS encryption, and registry-ui offers a secure yet simple solution for managing Docker images privately. While it lacks advanced Harbor-like features, it's lightweight, resource-efficient, and secure for many use cases.

Happy containerizing! 🚀

> **Written by:** [Ruchan Yalçın](https://ruchan.dev) | [GitHub](https://github.com/ruchany13)

## 🏢 Production Environment Recommendations

Although this setup is suitable for development and small-scale production, for a more reliable production-ready deployment, consider the following improvements:

### 🔄 1. Implement Regular Backups

Docker images stored in the registry are critical assets and must be protected.

- **What to do**: Regularly back up the `registry_data/` directory to external storage or cloud solutions.
- **Suggestions**:
  - Local disk backups (e.g., external drives)
  - Network Attached Storage (NAS)
  - Cloud storage solutions (e.g., AWS S3, Google Cloud Storage, Backblaze B2)

These methods ensure that even in case of server failure, your images remain safe.

### 🔒 2. Automate TLS Certificate Renewal

Certbot certificates expire every 90 days.

- **What to do**: Set up automatic certificate renewal and reload Nginx after renewal.
- **Suggestions**:
  - Use `certbot renew` command periodically.
  - Automate the process with scheduled tasks (e.g., cron jobs).

This ensures your site stays secure without manual intervention.

### ⚡ 3. Plan for High Availability (Optional)

For mission-critical setups:
- Deploy multiple `registry` instances behind an Nginx load balancer.
- Use a shared storage solution (like NFS or S3-backed storage).
- Use a health check system for containers.

Even a simple backup `docker-compose` stack in another server can increase resilience.


### 🛡️ 4. Enhance Security Settings

- **Nginx Enhancements**:
  - Add rate limiting to protect against brute-force attacks.
  - Restrict allowed IPs or authentication methods if necessary.

- **Example Basic Rate Limit** in `nginx.conf`:

    ```nginx title="Add inside server block"
    limit_req_zone $binary_remote_addr zone=one:10m rate=5r/s;

    location / {
      limit_req zone=one burst=10;
      proxy_pass http://docker-registry;
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header X-Forwarded-Proto $scheme;
    }
    ```

- **Auth Hardening**: Rotate your `htpasswd` credentials periodically.


---

By following these practices, you can make your Docker Registry setup much more robust, scalable, and production-friendly! 🚀

Happy containerizing! 🛡️
