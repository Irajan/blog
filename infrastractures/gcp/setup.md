# Multi-Service Docker Deployment on GCP

Deploy two containerized web services—a Next.js app and a marketing site—onto a single Google Cloud Platform (GCP) virtual machine using Docker, Docker Compose, and Nginx for domain-based routing.

## Architecture Overview

- **Docker containers** provide consistent runtimes across developer machines and GCP VMs.
- **Docker Compose** orchestrates multiple services with a declarative YAML file.
- **Internal Docker networking** lets services communicate using DNS-style service names without exposing ports publicly.
- **Nginx reverse proxy** terminates all HTTP/HTTPS traffic on the VM and forwards requests to the correct service based on the incoming domain.

## GCP VM Provisioning

### Create the VM Instance

1. Open the GCP Console and navigate to `Compute Engine → VM instances → Create Instance`.
2. Select an OS image such as **Ubuntu 22.04 LTS** for long-term support.
3. Choose a machine type (`e2-micro` for testing, `e2-small` or `e2-medium` for production).
4. Assign a network tag like `web-server` to reference in firewall rules.
5. Reserve a static external IP if you plan to map DNS records.

### Configure Firewall Rules

1. Go to `VPC network → Firewall → Create firewall rule`.
2. Name the rule `allow-web-traffic-docker` (or similar) and attach it to the correct VPC.
3. Set **Direction** to `Ingress` and **Action on match** to `Allow`.
4. Target the previously created network tag (e.g., `web-server`).
5. Specify source IP range `0.0.0.0/0` to allow global access.
6. Enable TCP ports `80` and `443`, then create the rule.

## Install Prerequisites (via SSH)

SSH into the VM and install Docker and helper tooling.

### Update System Packages

```bash
sudo apt update && sudo apt upgrade -y
```

### Install Docker Engine on Ubuntu/Debian

```bash
sudo apt install ca-certificates curl gnupg -y
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo \"$VERSION_CODENAME\") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```

### Allow Docker Usage Without `sudo`

```bash
sudo usermod -aG docker $USER
```

Log out and back in (or run `newgrp docker`) so the group change takes effect.

## Project Structure

Create the deployment layout under your home directory.

```bash
cd ~
mkdir -p deployment/nginx
mkdir -p deployment/nextjs_app
mkdir -p deployment/marketing_site
```

Expected structure:

```
deployment/
├── docker-compose.yml
├── marketing_site/
│   └── Dockerfile
├── nextjs_app/
│   └── Dockerfile
└── nginx/
    └── nginx.conf
```

## Configuration Files

### `docker-compose.yml`

```yaml
version: '3.8'

networks:
  web_network:
    driver: bridge

services:
  nextjs_app:
    container_name: nextjs_app
    build:
      context: ./nextjs_app
    image: nextjs-app-image:latest
    expose:
      - "3000"
    networks:
      - web_network
    restart: always

  marketing_site:
    container_name: marketing_site
    build:
      context: ./marketing_site
    image: marketing-site-image:latest
    expose:
      - "80"
    networks:
      - web_network
    restart: always

  nginx:
    container_name: nginx_proxy
    image: nginx:latest
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf
      # - ./certbot/conf:/etc/letsencrypt      # Enable after configuring SSL
      # - ./certbot/www:/var/www/certbot       # Enable after configuring SSL
    networks:
      - web_network
    depends_on:
      - nextjs_app
      - marketing_site
    restart: always
```

### `nginx/nginx.conf`

```nginx
# Main Nginx configuration for domain-based routing
worker_processes 1;

events {
    worker_connections 1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile on;
    keepalive_timeout 65;

    proxy_buffers 16 64k;
    proxy_buffer_size 128k;

    server {
        listen 80;
        server_name app.example.com;

        location / {
            proxy_pass http://nextjs_app:3000;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_read_timeout 300;
        }
    }

    server {
        listen 80 default_server;
        server_name example.com www.example.com;

        location / {
            proxy_pass http://marketing_site:80;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

## Deployment Workflow

1. Copy your Next.js and marketing site application code into `deployment/nextjs_app` and `deployment/marketing_site` respectively.
2. From the `deployment` directory, build and start the stack:

   ```bash
   docker compose up --build -d
   ```

3. Confirm the containers are running:

   ```bash
   docker ps
   ```

   Expect to see `nginx_proxy`, `nextjs_app`, and `marketing_site` with a `Status` of `Up`.

4. Verify the shared network:

   ```bash
   docker network ls
   docker network inspect deployment_web_network
   ```

   Ensure all three services appear as endpoints on `deployment_web_network` (or the auto-generated network name).

## DNS Configuration

### Prerequisites

- Static external IP address of your GCP VM.
- Access to your domain registrar or DNS provider.

### Create A Records

Add the following records, replacing `[YOUR_GCP_VM_EXTERNAL_IP]` with the VM's static IP:

| Type | Host/Name | Value/Target                | Purpose                                           |
| :--- | :-------- | :-------------------------- | :------------------------------------------------ |
| A    | `@`       | `[YOUR_GCP_VM_EXTERNAL_IP]` | Routes `example.com` to the VM.                   |
| A    | `www`     | `[YOUR_GCP_VM_EXTERNAL_IP]` | Routes `www.example.com` to the VM.               |
| A    | `app`     | `[YOUR_GCP_VM_EXTERNAL_IP]` | Routes `app.example.com` to the Next.js service.  |

> **TTL tip:** Use a lower TTL (e.g., 300 seconds) during setup to speed up propagation, then raise it for production stability.

### Verify Propagation

Use `dig` (or `nslookup`) from your local machine:

```bash
dig example.com +short
dig app.example.com +short
```

Both commands should return `[YOUR_GCP_VM_EXTERNAL_IP]`. Once verified, your domains will route through Nginx to the appropriate containerized service.
