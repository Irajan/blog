ervice Docker Deployment Guide on GCP
This guide outlines the theoretical concepts and practical steps for deploying multiple
containerized web services (Next.jsapplication and a Marketing website) onto a single Google
Cloud Platform (GCP) Virtual Machine (VM)instance using Docker,Docker Compose, and Nginx
as a reverse proxy for domain-based routing.

1. Theoretical Concepts and Architecture

1.1 Docker and Docker Compose
Docker: Allows you to package your application and its dependencies into a container. This
ensures that your application runs reliably and consistently across different computing
environments (local machine, VM, etc.).

Docker Compose: A tool for defining and running multi-container Docker applications. You
use a YAML file (docker-compose.yml) to configure all your application's services, networks,
and volumes, and then start everything with a single command.

1.2 Docker Internal Network

When using Docker Compose, a dedicated bridge network is automatically created by default.
Crucially: Containers on this network can communicate with each other using their service
names as hostnames (e.g.,nextjs_app can talk to marketing_site).

This internal network traffic is hidden from the outside world and is what allows Nginx to forward
requests securely without exposing internal container ports directly to the internet.

1.3 Nginx Reverse Proxy
Role: Nginx acts as the single point of entry for all incoming HTTP/HTTPS traffic on the VM
(ports 80 and 443).

Reverse Proxying:
Based on the incoming domain name (the Host header, e.g., example.com or
app.example.com)

Nginx forwards the request to the correct internal Docker container via the internal network. This
is how domain-based routing is achieved.

Diagram: The following diagram illustrates how external requests are routed through Nginx to
the appropriate internal container via the Docker network.

2. GCP VM and Firewall Setup
2.1 Create the VM Instance

1. 2. 3. Select OS: Choose a Linux distribution (e.g. Ubuntu 22.04 LTS is highly
recommended).

Machine Type:
Select an appropriate machine type (e.g., e2-micro for testing, or e2-small/e2-medium for
production load).

Network Tags: Assign a network tag
(e.g., web-server) to the VM.
2.2 Configure Firewall Rules (GCP Console Step-by-Step)
You need to allow incoming traffic on the standard HTTP and HTTPS ports to reach your VM
from the outside world.
1. Navigate to VPC Network: In the GCP Console, go to VPC network -
> Firewall.
2. Create Rule: Click CREATE FIREWALL RULE.
3. Details:
Name: allow-web-traffic-docker (or similar, descriptive name).
Network: Select the VPC network your VM uses (usually default).

Direction of traffic: Ingress.
Action on match:Allow.
Targets: Specified target tags.
Target tags: Enter the tag you assigned to your VM(e.g., web-server).
Source filter: IP ranges.
Source IP ranges: 0.0.0.0/0 (Allows traffic from anywhere).
4. Protocols and ports: Select Specified protocols and ports.
Check tcp and enter ports: 80, 443.
5. Save: Click CREATE.

3. Prerequisites Installation (via SSH)

SSH into your new GCP VM instance and run the following commands.

3.1 Update System Packages
sudo apt update && sudo apt upgrade -y
3.2 Install Docker Engine (Recommended for Debian/Ubuntu)
These commands use lsb_release robust across Debian-based systems.-cs to dynamically
detect your distribution's codename, making the installation more

# 2. Install essential tools and create the keyrings directory
sudo apt install ca-certificates curl gnupg ls -release -y b
sudo mkdir -p /etc/apt/keyrings

# 3. Download Docker's GPG key
# Note: Using 'ubuntu' in the URL is generally ompatible with Debian/Ubuntu derivatives
 ccurl -fsSL
[https://download.docker.com/linux/ubuntu/gpg](https://download.docker.com/linux/ubut
u/gpg) |

# 4. Set proper permissions
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# 5.
Add the Docker repository dynamically
echo \

“deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg]

[https://download.docker $(ls_release -cs) stable" | sudo tee
/etc/apt/sources.list.d/docker /dev/null

# 6. Update apt
sudo apt update
# 7. Install Docker Engine
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin
docker-compose-plugin -y

3.3 Add User to Docker Group (Optional, but Recommended)
This allows you to run Docker commands without sudo. You must log out and log back in for
this change to take effect.
sudo usermod -aG docker $USER

4. Project Structure and Configuration
Create the following folder structure on your VM and place your code (and the following
configuration files)inside.

4.1 Setup Directory Structure
# Navigate to your home directory

cd ~
# Create the main deployment folder and required sub

mkdir -p deployment/nginx
mkdir deployment/nextjs_app
mkdir deployment/marketing_site
cd deployment
directories

Your final structure should look like this, with the files defined in the next section:

Let’s take following as the example contents of config files

Docker-compose.yml file

version: '3.8'

# Define the custom bridge network for internal communication

networks:

 web_network:

   driver: bridge

services:

 # 1. Next.js Application Service (app.example.com)

 nextjs_app:

   container_name: nextjs_app

   build:

     context: ./nextjs_app # Location of the Next.js Dockerfile

   image: nextjs-app-image:latest

   # Expose internal port 3000 for Nginx to access

   expose:

     - "3000"

   networks:

     - web_network

   restart: always

 # 2. Marketing Website Service (example.com)

 marketing_site:

   container_name: marketing_site

   build:

     context: ./marketing_site # Location of the Marketing Dockerfile

   image: marketing-site-image:latest

   # Expose internal port 80 for Nginx to access

   expose:

     - "80"

   networks:

     - web_network

   restart: always

 # 3. Nginx Reverse Proxy Service

 nginx:

   container_name: nginx_proxy

   image: nginx:latest

   ports:

     # Expose Nginx ports 80 and 443 to the host VM, serving as the entry

point

     - "80:80"

     - "443:443"

   volumes:

     # Mount the custom Nginx config to overwrite the default Nginx config

     - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf

     # Placeholder for Certbot (SSL) volumes to be added later

     # - ./certbot/conf:/etc/letsencrypt

     # - ./certbot/www:/var/www/certbot

   networks:

     - web_network

   depends_on:

     - nextjs_app

     - marketing_site

   restart: always

Nginx.conf file

# Main Nginx configuration file
worker_processes 1;

events {
    worker_connections 1024;
}

http {
    include mime.types;
    default_type application/octet-stream;

    sendfile on;
    keepalive_timeout 65;

    # Increase buffer sizes for Next.js large payloads
    proxy_buffers 16 64k;
    proxy_buffer_size 128k;

    # Define the Next.js app server (app.example.com)
    server {
        listen 80;
        server_name app.example.com;

        location / {
            # Forward to the internal service 'nextjs_app' on port 3000
            proxy_pass http://nextjs_app:3000;

            # Standard Next.js/Proxy Headers
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # Required for Next.js to handle websockets/hot reloading correctly
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';

            # Set timeout to handle long-running Next.js requests
            proxy_read_timeout 300;
        }
    }

    # Define the Marketing site server (example.com)
    server {
        listen 80 default_server;
        server_name example.com [www.example.com](https://www.example.com); # List the root
domain and www alias

        location / {
            # Forward to the internal service 'marketing_site' on port 80 (default Nginx port)

proxy_pass http://marketing_site:80;

            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}

Update DNS record

# -d runs the containers in detached (background) mode
docker compose up --build -d

### Step 5.2: Verify Containers
Ensure all containers are running and healthy.
```bash
docker ps
*You should see `nginx_proxy`, `nextjs_app`, and `marketing_site` listed with `Status` as `Up`.*

### Step 5.3: Verify Network
Ensure the containers are on the same network and can communicate.

```bash
docker network ls
# Find the network named deployment_web_network (or similar)
# Now inspect the network to confirm all three services are listed as endpoints
docker network inspect deployment_web_network

## 6. Domain Name System (DNS) Mapping

This is the process of linking your domain names (e.g., `example.com`) to the physical location
of your server (your GCP VM's External IP).

### 6.1 Prerequisites

1.  **GCP External IP:** You must have the static **External IP address** of your VM instance.
2.  **DNS Provider Access:** You must have administrative access to your domain's DNS
management settings (usually through your domain registrar like GoDaddy, Namecheap, or a
separate service like Cloudflare).

### 6.2 The Mapping Process: Creating A Records

You will be creating **A records** (Address records) for each domain and subdomain you want
Nginx to handle. An A record maps a hostname to an IPv4 address.

1.  **Log in** to your domain registrar or DNS provider's control panel.
2.  **Navigate** to the DNS management or Zone Editor section for your domain
(`example.com`).
3.  **Create the records** as follows, replacing `[YOUR_GCP_VM_EXTERNAL_IP]` with the
actual IP address of your VM.

| Type | Host/Name | Value/Target | Purpose |
| :--- | :--- | :--- | :--- |
| **A** | `@` | `[YOUR_GCP_VM_EXTERNAL_IP]` | Maps the root domain (`example.com`) to
the VM. |
| **A** | `www` | `[YOUR_GCP_VM_EXTERNAL_IP]` | Maps the `www` subdomain
(`www.example.com`) to the VM. |
| **A** | `app` | `[YOUR_GCP_VM_EXTERNAL_IP]` | Maps the Next.js app subdomain
(`app.example.com`) to the VM. |

> **Note on TTL (Time To Live):** DNS changes are not instant. Most providers use a default
TTL (e.g., 3600 seconds = 1 hour). While setting up, you might temporarily set the TTL to a
lower value (e.g., 60 seconds) to speed up propagation, but remember to raise it back for
production performance.

### 6.3 Verification of DNS Propagation

After saving the A records, you need to wait for the changes to propagate globally (which can
take a few minutes up to 48 hours, depending on the TTL).

You can verify the propagation using local command-line tools:

#### CLI Commands to Verify

1.  **Check Root Domain (`example.com`):**
    ```bash
    dig example.com +short
    # Expected output: [YOUR_GCP_VM_EXTERNAL_IP]

2.  **Check Subdomain (`app.example.com`):**
    ```bash
    dig app.example.com +short
    # Expected output: [YOUR_GCP_VM_EXTERNAL_IP]

Once these commands return your VM's external IP address, your domain is successfully
pointing to your Nginx proxy, and the deployment is ready for external access (as described in
the request flow).


