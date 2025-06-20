# Node.js CI/CD Deployment with GitHub Actions (Docker & Nginx)

This repository demonstrates a complete CI/CD pipeline for deploying a **Node.js (Express)** application to an Ubuntu server (with a public IP) using **Docker containers** and **GitHub Actions**. It supports separate **staging** and **production** environments using Git branching strategy (e.g. a `develop` branch deploys to a staging environment, and `main` branch to production). The deployment uses **Nginx** as a reverse proxy in front of the Node.js app, with optional SSL termination via Let’s Encrypt (Certbot). Both Continuous Integration (CI) (running tests and lint checks) and Continuous Deployment (CD) (automatic server deploys via SSH) are set up.

## Project Structure

Below is an overview of the repository layout, including the Node.js app, Docker configuration, Nginx config, and GitHub Actions workflows:

```bash
├── app.js               # Node.js Express application (entry point)
├── package.json         # Node.js dependencies and scripts (including test, lint, start)
├── Dockerfile           # Docker image specification for the Node app
├── docker-compose.yml   # Docker Compose file for running the app container
├── .env.staging         # Environment variables for staging (non-sensitive demo values)
├── .env.production      # Environment variables for production (non-sensitive demo values)
├── nginx/
│   └── myapp.conf       # Example Nginx site config for the app (reverse proxy settings)
└── .github/
    └── workflows/
        ├── ci.yml           # GitHub Actions workflow: run tests & lint on pushes
        └── deploy.yml       # GitHub Actions workflow: deploy to server on branch push
```

**Note:** In a real project, sensitive values (API keys, database passwords, etc.) should not be committed in `.env.staging` or `.env.production`. Those are shown here for demonstration and would typically be stored in GitHub Secrets or other secret management in a real deployment.

## Node.js Application Setup (`app.js`)

The Node.js app is a simple Express server with basic routing. It reads environment variables (for example, a port number and an environment name) and exposes at least one route (e.g. the root route returning a simple message). This keeps the focus on deployment rather than application logic:

```javascript
// app.js
const express = require('express');
require('dotenv').config();  // Load .env file (for local dev; in Docker, env vars are passed differently)
const app = express();

// Use PORT from environment if provided, otherwise default to 3000
const PORT = process.env.PORT || 3000;
const ENV = process.env.NODE_ENV || process.env.ENVIRONMENT || 'development';

// Basic route
app.get('/', (req, res) => {
  res.send(`Hello from Node.js! Environment: ${ENV}`);
});

// A health check route (useful for testing)
app.get('/health', (req, res) => {
  res.json({ status: 'OK', env: ENV });
});

// Start the server
app.listen(PORT, () => {
  console.log(`App listening on port ${PORT} (Environment: ${ENV})`);
});
```

In the example above, we use `dotenv` in development to load variables from a `.env` file. In the Docker deployment, these variables will be provided via Docker Compose, so loading via `dotenv` isn't strictly necessary in production – it's included mainly to ease local testing.

We also include a `package.json` with necessary dependencies and scripts:

```json
{
  "name": "my-node-app",
  "version": "1.0.0",
  "main": "app.js",
  "scripts": {
    "start": "node app.js",
    "test": "echo \"Running tests...\" && exit 0",
    "lint": "echo \"Running linter...\" && exit 0"
  },
  "dependencies": {
    "express": "^4.18.2",
    "dotenv": "^16.0.3"
  }
}
```

*(In a real project, you would replace the dummy `test` and `lint` scripts with actual testing (e.g. using Jest or Mocha) and linting (e.g. ESLint) commands. The above scripts simply echo a message and succeed, for illustration.)*

## Docker Configuration

**Dockerfile:** We containerize the Node.js app using a Dockerfile. This ensures consistent runtime environment across staging/production. A simple Dockerfile for a Node/Express app might be:

```Dockerfile
# Dockerfile
FROM node:18-alpine          # Use an official Node.js runtime image (alpine for small size)
WORKDIR /app

# Install dependencies (copy package.json first for caching)
COPY package*.json ./
RUN npm i --only=production

# Copy application source code
COPY . .

# Specify port (optional, mainly for documentation)
ENV PORT=3000
EXPOSE 3000

# Start the app
CMD ["npm", "start"]
```

This Dockerfile uses Node 18 LTS on Alpine Linux for a lightweight image. It sets up the working directory, installs Node dependencies, copies the app code, and then defines the startup command. The app will run on port 3000 inside the container.

**docker-compose.yml:** We use Docker Compose to manage the container (and potentially other services in the future). The Compose file defines how to run the Node app container in both staging and production. We will leverage environment-specific files for configuration:

```yaml
# docker-compose.yml
version: '3'
services:
  app:
    container_name: myapp-web           # Name of the container
    build: .                            # Build image from the Dockerfile in this directory
    ports:
      - "3000:3000"                     # Map container's port 3000 to server's port 3000
    env_file: 
      - .env                            # Use environment variables from .env file (will be set to staging or production via deployment script)
    restart: unless-stopped
```

Key points in this Compose setup:

* It exposes port 3000 of the container to port 3000 on the host. Nginx (running on the host) will reverse proxy to this port.
* It uses an `env_file` directive pointing to a file named `.env`. **In practice, we will dynamically choose the correct environment file** on deployment (either `.env.staging` or `.env.production`) by copying or symlinking it to `.env` before running Docker Compose. This mechanism allows us to deploy the same container image with different environment-specific settings.
* The `restart: unless-stopped` policy ensures the container auto-starts on server reboot or if it crashes.

Using an environment file with Compose keeps sensitive configuration out of the repository’s Dockerfile and Compose (you can reuse the same Compose file for both environments, simply swapping env files). The Docker Compose CLI supports an override of the default `.env` file via `--env-file` flag if needed. For example, you can run:

```bash
docker-compose --env-file .env.staging up -d --build
```

to explicitly use the staging config, and similarly for production. This is how our deployment scripts will differentiate the environments.

## Environment Variables for Staging vs Production

We maintain two separate environment files at the root of the repo: **.env.staging** and **.env.production**. These files contain key-value pairs that will be loaded into the container to configure the app for each environment:

* `.env.staging` – for example:

  ```bash
  NODE_ENV=staging
  ENVIRONMENT=staging
  PORT=3000
  # (Other staging-specific vars, e.g. DB connection strings, API keys pointing to staging services)
  ```
* `.env.production` – for example:

  ```bash
  NODE_ENV=production
  ENVIRONMENT=production
  PORT=3000
  # (Other production-specific vars)
  ```

In our simple app, we’re just using `ENVIRONMENT` to print which environment the app is running in. In a real scenario, you might have different database URIs or API endpoints here. During deployment, the GitHub Actions workflow will ensure the correct file is used:

* For **staging** deployments (from the `develop` branch), the workflow will either run Compose with `--env-file .env.staging` or copy `.env.staging` to `.env` before bringing up the container.
* For **production** deployments (from the `main` branch), it will use `.env.production` similarly.

By separating these, you ensure that, for instance, any analytics keys or database URLs for production are not used in staging and vice versa. It also prevents accidental use of production credentials in a test environment.

## Nginx Configuration (Reverse Proxy)

We will use **Nginx** on the Ubuntu server as a reverse proxy to route HTTP(S) requests to the Node.js app container. Nginx will listen on standard ports (80 for HTTP, 443 for HTTPS) and forward traffic to the Node app (which listens on port 3000 internally).

**Installation:** On the server, Nginx can be installed via apt. For example:

```bash
sudo apt update && sudo apt install nginx -y    # Install Nginx on Ubuntu:contentReference[oaicite:0]{index=0}
```

Once installed, we create an Nginx site configuration for our app (assuming a domain name or using the server IP). For example, `/etc/nginx/sites-available/myapp.conf`:

```nginx
# nginx/myapp.conf (sample Nginx config for our Node.js app)
server {
    listen 80;
    server_name example.com;  # replace with your domain or IP

    location / {
        proxy_pass http://localhost:3000;   # forward requests to Node app
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

In this configuration:

* Nginx listens on port 80 for `example.com` (you should use your domain or server IP here).
* All requests (`location /`) are proxied to `http://localhost:3000`, where our Node Docker container will be listening. We include standard proxy headers:

  * `Host` header is preserved,
  * `X-Real-IP` and `X-Forwarded-For` pass on the client’s IP address to the Node app,
  * The last two headers enable WebSocket upgrades if our app ever uses them (not strictly needed for a basic app, but good practice for Node/Express).
* If you have multiple domains or subdomains (for example, a staging subdomain), you would create a similar server block for each, possibly pointing to different container ports or servers. For simplicity, this example assumes one site.

Enable this site by creating a symlink to `sites-enabled` (if on Ubuntu default Nginx setup):

```bash
sudo ln -s /etc/nginx/sites-available/myapp.conf /etc/nginx/sites-enabled/
sudo nginx -t   # test configuration
sudo systemctl reload nginx   # apply changes
```

Now Nginx will route traffic from port 80 to our app. At this point, if the Node app container is up, you should be able to visit `http://example.com` (or the server’s IP) and see the "Hello from Node.js! Environment: ..." message.

## GitHub Actions Workflows

We use GitHub Actions to automate both testing (CI) and deployment (CD). Workflows are defined in YAML files under `.github/workflows/`.

### 1. Continuous Integration (CI) Workflow – `ci.yml`

This workflow runs on every push (and/or pull request) to the repository (you can adjust branches as needed). Its purpose is to ensure code quality and correctness before deployment by running linters and tests:

```yaml
# .github/workflows/ci.yml
name: CI - Test and Lint

on:
  push:
    branches: ["main", "develop", "feature/*"]   # run on pushes to main, develop, or feature branches
  pull_request:

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: 18

      - name: Install dependencies
        run: npm ci

      - name: Lint code
        run: npm run lint   # runs the linter (e.g. ESLint) on the code

      - name: Run tests
        run: npm test       # runs the test suite
```

**What this does:** On each push or PR, the workflow checks out the repo, sets up Node.js 18, installs dependencies (using `npm ci` for consistency with lockfile), then runs `npm run lint` and `npm test`. If any of these steps fail (exit with non-zero status), the workflow will be marked as failed, preventing a deploy. You might configure branch protection so that the `develop` or `main` branch cannot be deployed unless this CI passes.

*(In our example, the lint and test scripts are placeholders. In a real project, you’d ensure `npm run lint` actually lints your code (e.g. via ESLint with a config), and `npm test` runs your test suite.)*

### 2. Continuous Deployment (CD) Workflow – `deploy.yml`

This workflow is triggered **only on pushes to certain branches** – namely `develop` (to deploy to staging) and `main` (to deploy to production). It uses SSH to connect to the server and deploy the new version. We’ll write it as a single workflow that handles both environments by checking the branch name, but you could also split it into two separate workflow files if preferred.

```yaml
# .github/workflows/deploy.yml
name: CD - Deploy to Server (Staging & Production)

on:
  push:
    branches:
      - "main"
      - "develop"

jobs:
  deploy:
    runs-on: ubuntu-latest

    # Only proceed if the CI job on this commit succeeded (optional but recommended)
    # needs: [ build-and-test ]

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Determine environment
        id: envdetermine
        run: |
          if [ "${GITHUB_REF##*/}" = "main" ]; then echo "env=production" >> $GITHUB_OUTPUT; else echo "env=staging" >> $GITHUB_OUTPUT; fi

      - name: Deploy via SSH
        uses: easingthemes/ssh-deploy@v2.1.5
        env:
          SSH_PRIVATE_KEY: ${{ secrets.SSH_KEY }}       # private key for SSH, added in GitHub secrets
          REMOTE_HOST: ${{ secrets.SERVER_HOST }}       # server IP or hostname
          REMOTE_USER: ${{ secrets.SERVER_USER }}       # SSH username (e.g., "ubuntu")
          REMOTE_PORT: ${{ secrets.SERVER_PORT || 22 }} # SSH port (22 by default)
          # Only deploy the necessary files (exclude node_modules, etc.)
          SOURCE: "."                                   # source is the whole repository (after checkout)
          EXCLUDE: ".git/,node_modules/"
          TARGET: "/home/ubuntu/myapp-${{ steps.envdetermine.outputs.env }}/"  # target dir on server (separate for staging/prod)
          SCRIPT_AFTER: |
            # On the server: navigate to project directory and deploy with Docker
            cd /home/ubuntu/myapp-${{ steps.envdetermine.outputs.env }}/
            # Use the appropriate env file for Docker Compose
            if [ "${{ steps.envdetermine.outputs.env }}" = "production" ]; then 
              cp .env.production .env; 
            else 
              cp .env.staging .env; 
            fi
            docker-compose up -d --build   # build and run the container
```

Let’s break down what the deploy job is doing:

* **Trigger:** It runs on pushes to `main` or `develop` branches.
* **Checkout:** We checkout the repo to access files for copying.
* **Determine environment:** A small shell step checks the branch name: if it's `main`, we mark `env=production`, otherwise `env=staging`. We use this to adjust paths and behavior later.
* **SSH Deploy Action:** We use the `easingthemes/ssh-deploy` action (an open-source GitHub Action) which uses **rsync over SSH** to upload files and run commands remotely. We provide it the necessary secrets and instructions:

  * `SSH_PRIVATE_KEY`: the private key (stored in GitHub Secrets) for SSH auth on the server.
  * `REMOTE_HOST`: the server’s IP or hostname (from secrets).
  * `REMOTE_USER`: the SSH login username on the server (from secrets, e.g. `"ubuntu"` for an AWS/DO Ubuntu box).
  * `REMOTE_PORT`: (optional, defaults to 22).
  * `SOURCE`: `.` meaning sync the entire repository (all files from the checkout) to the server.
  * `EXCLUDE`: we exclude certain paths from upload. Here we exclude the `.git` folder and `node_modules` (in case someone had built locally) to minimize transfer.
  * `TARGET`: the target directory on the server. We include the environment in the path (e.g. `/home/ubuntu/myapp-production/` vs `/home/ubuntu/myapp-staging/`) so that staging and production deployments reside in separate directories. This is one way to allow both environments to coexist on the same server (if desired). You could also use separate servers for each environment, in which case you’d use different host/SSH secrets instead.
  * `SCRIPT_AFTER`: commands to run **on the remote server** *after* the files are synced:

    * Change directory to the deployment folder.
    * Copy the appropriate environment file to `.env` (overwriting the old one) so Docker Compose will use the correct variables.
    * Run `docker-compose up -d --build`. This will build the Docker image (using the updated code) and start the container. We could also add `docker-compose down` or `docker-compose stop` before the `up` if we want to ensure old containers are removed first; however, `up --build` will recreate the container with the new image, and since we keep the same container name, Compose will handle replacing it. The `-d` flag runs it in the background (detached).

This single workflow handles both branches. In the example above, we used one set of SSH secrets (`SSH_KEY`, `SERVER_HOST`, etc.) and differentiated the target directory by environment. Alternatively, you could simplify by using **two separate workflows**: one that triggers on `develop` with its own host/user (perhaps a staging server) and another on `main` with production server credentials. The approach will depend on whether you deploy to the same server or different servers for staging/production:

* **Same Server for both**: Use different target directories and maybe different port or container names. Nginx can route e.g. `staging.example.com` to one container and `example.com` to another. (Our workflow above is set up this way, with directory separation.)
* **Different Servers**: Use separate secrets for each environment. For example, `STAGING_HOST`, `PROD_HOST`, etc., and possibly maintain two workflow files or use if/else in one workflow to switch which secrets to use.

Regardless of approach, storing the SSH key and server details as **GitHub Secrets** is crucial. In our case we used environment variables the action expects. For example:

* `secrets.SSH_KEY` contains the *private* key for SSH login (the public key is on the server).
* `secrets.SERVER_HOST` is the server’s IP (or DNS name).
* `secrets.SERVER_USER` is the SSH username (e.g. `"ubuntu"` or `"root"` depending on your server setup).
* (If using separate servers, you’d have `STAGING_HOST`, `PROD_HOST`, etc. as needed.)
* Optionally, you might also store `LETSENCRYPT_EMAIL` as a secret (the email to use for Certbot), if you plan to trigger certificate issuance via automation (though in this guide we handle SSL setup manually on the server).

**Security:** Notice we never store sensitive data (like the SSH key or any production passwords) in the repo – only in Secrets. The SSH key is injected at runtime to establish the connection. Ensure your SSH key has **no passphrase** (GitHub Actions can’t prompt for one). Generate it with an empty passphrase for this purpose, and add the public key to your server’s `~/.ssh/authorized_keys` for the deployment user.

### Workflow Summary

With CI/CD workflows in place:

* When you push code to a feature or development branch, the CI workflow runs tests and lint checks.
* When you push or merge to **`develop`**, the CI runs and then the deploy workflow uploads the code to the **staging environment** and runs `docker-compose` to update the container.
* When you push or merge to **`main`**, the same happens for the **production environment**.

You might add additional safeguards, e.g. requiring manual approval for deploying to production, or only deploying on version tags, etc., but the above is a basic automated flow.

## Server Setup and Deployment Instructions

Setting up the Ubuntu server is a one-time (per server) task. Below are the steps to prepare the server for running our Dockerized Node app behind Nginx. These steps assume you have an Ubuntu 20.04/22.04 server (e.g., a cloud VM or droplet) with a public IP, and that you have root or sudo access.

**1. Install Docker Engine and Docker Compose:**

* Follow Docker’s official installation for Ubuntu (you can use the convenience script or apt repository). For a quick setup on Ubuntu, you can install the `docker.io` package and the Compose plugin:

  ```bash
  sudo apt update
  sudo apt install -y docker.io docker-compose
  ```

  This installs Docker and the old `docker-compose` tool. Alternatively, install **Docker Engine** and **Docker Compose Plugin** as per the latest instructions for better support. Ensure your user is added to the `docker` group if you want to run Docker without root. Verify Docker works by running `sudo docker run hello-world`.

**2. Install Nginx:**

* As mentioned, install Nginx via apt:

  ```bash
  sudo apt install -y nginx:contentReference[oaicite:6]{index=6}
  ```

  After installation, ensure Nginx is running (`systemctl status nginx`). If you have a firewall (ufw), allow HTTP/HTTPS: e.g., `sudo ufw allow 'Nginx Full'` (this opens ports 80 and 443).

**3. Configure Nginx for your app:**

* Create the Nginx config file (as shown in the **Nginx Configuration** section above). Use your actual domain name or server IP in the `server_name`. Place the config in `/etc/nginx/sites-available/myapp.conf`, then enable it (`ln -s` to sites-enabled). Test config (`nginx -t`) and reload Nginx. At this point, Nginx is ready to forward traffic to your app **once it’s running**. If you visit your domain/IP now you might get a bad gateway until the app container is up – that’s expected.

**4. Prepare directories on the server:**

* Decide where on the server you want to deploy the application code. In our GitHub Actions, we chose `/home/ubuntu/myapp-staging` and `/home/ubuntu/myapp-production` for staging and prod respectively. You should create these directories and give ownership to the SSH user (if not already in their home). For example:

  ```bash
  sudo mkdir -p /home/ubuntu/myapp-staging /home/ubuntu/myapp-production
  sudo chown -R ubuntu:ubuntu /home/ubuntu/myapp-staging /home/ubuntu/myapp-production
  ```

  (Replace `ubuntu:ubuntu` with your username and group if different.)

* These directories will receive the uploaded files from the GitHub Actions (the rsync deployment will populate them). They also will hold the Docker Compose file, the Dockerfile, and so on – essentially a copy of your repo (minus excluded files).

**5. Set up SSH access for GitHub Actions:**

* Generate an SSH key pair for deployments (on your **local machine** or through the GitHub web UI). Ensure no passphrase. For example, on your local machine:

  ```bash
  ssh-keygen -t rsa -b 4096 -m PEM -f github_deploy_key -N ""
  ```

  This creates a 4096-bit RSA key in PEM format with no passphrase. The `-m PEM` ensures the private key is in a format that the Action (which uses OpenSSH/rsync) expects.

* Copy the **public key** (`github_deploy_key.pub`) to the Ubuntu server’s `~/.ssh/authorized_keys` for the user that will perform deployment (e.g., add it to `/home/ubuntu/.ssh/authorized_keys`). You can use `ssh-copy-id` or manually append the key text. Test that you can SSH into the server from your local machine using the private key:

  ```bash
  ssh -i github_deploy_key [email protected]
  ```

  (It should log in without password if setup correctly.)

* In your GitHub repository, go to **Settings > Secrets and Variables > Actions** and add the following **Secrets**:

  * `SSH_KEY` – contents of the *private* key (`github_deploy_key`). Open the file and copy the text (the key starting with `-----BEGIN RSA PRIVATE KEY-----` through `END RSA PRIVATE KEY-----`). Store that whole text as the secret value. This will be referenced in the workflow as `${{ secrets.SSH_KEY }}`.
  * `SERVER_HOST` – the IP address or hostname of your server (e.g., `203.0.113.10` or `example.com`).
  * `SERVER_USER` – the SSH username (e.g., `"ubuntu"` or your configured user).
  * (If you used a non-standard SSH port, also add `SERVER_PORT`).
  * If using separate servers or keys for staging/prod, add those accordingly (e.g., `SSH_KEY_STAGING`, `STAGING_HOST`, etc., and modify the workflow to use them for the `develop` branch).

**6. (Optional) Add Swap or increase resources:** If your server is very small (e.g., a 1GB RAM VM), Docker image builds (especially with Node compiling dependencies) might need more memory. Consider adding a swap file or using a server with sufficient RAM so that `docker-compose build` won't fail due to lack of memory.

**7. Trigger a Deployment:**

With everything configured, push a commit to the `develop` branch (or merge a PR into `develop`). This should trigger the CI workflow; if tests pass, it then triggers the CD workflow (as shown above). The CD job will rsync the repository to `/home/ubuntu/myapp-staging/` on the server and run `docker-compose up -d --build`. On the server, Docker will build the image (using the Dockerfile in that folder) and start the container.

* You can run `docker ps -a` on the server (via SSH) to see if the container is running. It should show an `myapp-web` container bound to port 3000.
* Check the logs with `docker logs myapp-web` to ensure it started correctly (it should print the console log "App listening on port 3000...").
* Now visiting `http://<your-server-ip>/` (or the domain) in a browser should display the hello message from the app (staging environment). 🎉

If you then push to `main`, the same process will deploy the production environment (perhaps into `/home/ubuntu/myapp-production/` directory). If using the same server, you’d likely map Nginx to the same port 3000 for both – which means you shouldn’t run both containers concurrently on the same port. One solution is to run the staging container on a different port (say 3001) and adjust Nginx (and Compose ports) for the staging domain. Another simpler approach is to use a separate server for production vs staging to avoid port conflicts. **For clarity, you may choose to maintain separate servers for staging and production in a real scenario.** The workflows can be adjusted accordingly (different secrets for each).

## Optional: SSL Configuration with Let's Encrypt

For production deployments, it's highly recommended to serve traffic over HTTPS. We can obtain a free SSL/TLS certificate from **Let’s Encrypt** using the **Certbot** tool, and configure Nginx to use it.

**1. Install Certbot:** The simplest method on Ubuntu is to use the snap package or the system package. For instance, using apt:

```bash
sudo apt install certbot python3-certbot-nginx -y:contentReference[oaicite:11]{index=11}
```

This installs the Certbot client and the Nginx plugin.

**2. Obtain Certificate:** Make sure your domain’s DNS is pointing to the server’s IP. Then run Certbot:

```bash
sudo certbot --nginx -d example.com -d www.example.com:contentReference[oaicite:12]{index=12}
```

Replace the domains with your actual domain (you can include both apex and www or any subdomains as needed). Certbot will prompt for an email (for renewal notices) and agreement to terms if not provided in flags. The `--nginx` plugin will automatically configure Nginx for SSL – it will create a new Nginx configuration with port **443** for SSL and update your port **80** server block to redirect to HTTPS. After this runs, Nginx will be serving your site over HTTPS with the certificate in place.

Certbot will install certificate files to `/etc/letsencrypt/live/your_domain/` and edit the Nginx config to include them. For example, your `myapp.conf` may now contain lines like:

```nginx
    listen 443 ssl;
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    # ...other SSL options...
```

and a separate server block to redirect port 80 to 443:

```nginx
server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}
```

This is handled by Certbot automatically for you. The resulting configuration will look similar to the one below, which includes the HTTPS server and a redirect from HTTP:

```nginx
server {
    listen 443 ssl http2;
    server_name example.com;
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    # ... (SSL protocols, ciphers can be tuned here) ...
    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}
```

**3. Automate Renewal:** Let’s Encrypt certificates last 90 days. Certbot’s installation typically sets up a cron job or systemd timer to auto-renew. You can test renewal with `sudo certbot renew --dry-run`. As long as Nginx and the DNS remain the same, Certbot will renew and update the certs without any downtime. Ensure port 80 is open, as the renewal uses HTTP challenge by default.

With SSL in place, your app is now securely accessible at **`https://example.com`**. 🎁

*(If you have separate staging and production domains, you’d obtain certificates for both. You might choose to not enable HTTPS on a private staging environment, but it’s generally a good idea to mirror production as much as possible.)*

## Conclusion

By structuring the project with Docker and using GitHub Actions for CI/CD, we achieve a robust deployment pipeline:

* Developers can focus on code (with CI ensuring code quality).
* Merging to the staging branch triggers an automatic deploy to a staging server for testing in an environment similar to production.
* Promotion to production is as simple as merging to the main branch, which triggers a deploy to the live server.
* Docker and Nginx provide a consistent runtime and entry point, and using environment-specific configs (`.env.staging`/`.env.production`) ensures the app knows which environment it’s running in and can adjust behavior accordingly.
