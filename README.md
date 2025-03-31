# Production-Ready Docker Deployment on AWS EC2 with GitHub Actions CI/CD and SSL

## **Description**

This project provides a complete, production-grade solution to deploy a Docker-based web application on an AWS EC2 instance using GitHub Actions for CI/CD. It includes a secure EC2 setup, Docker Compose-based service orchestration, SSL termination via NGINX with Let's Encrypt, and secrets management using GitHub Secrets and, optionally, AWS Systems Manager. Perfect for teams or individuals looking to set up a robust deployment pipeline with best practices and automation.


- [Description](#description)
- [Problem Statement](#problem-statement)
- [Architecture Overview](#architecture-overview)
    - [Tech Stack & Requirements](#tech-stack--requirements)
    - [Directory Structure Explanation](#directory-structure-explanation)
    - [GitHub Actions CI/CD Pipeline](#github-actions-cicd-pipeline)
    - [Maintenance & Scaling Guide](#maintenance--scaling-guide)
    - [Bonus: Database Migration Handling](#bonus-database-migration-handling)
    - [License](#license)
    - [Live Deployment Demo or Blog](#live-deployment-demo-or-blog)


## **Problem Statement**

Deploying containerized applications in a production environment often lacks standardization, especially across cloud platforms. Manual server configurations, unencrypted traffic, and fragile deployment pipelines introduce vulnerabilities and maintenance burdens. This project addresses these critical issues by offering a streamlined, secure, and automated deployment solution.

Core problems solved:

- Manual and error-prone server setup.
- Unsecured deployments with missing SSL.
- Lack of CI/CD automation for Dockerized applications.
- Poor secret and config management.
- Scalability and maintenance issues in long-term environments.

This repository helps DevOps teams and backend engineers establish production-grade deployment infrastructure with minimal friction and future scalability in mind.

# **Architecture Overview**

The architecture is designed for modularity, security, and ease of automation. Below is a high-level overview:

1. **EC2 Instance (Ubuntu)**
    - Hardened server running Docker and Docker Compose
    - Configured with a static IP and limited open ports
2. **NGINX Reverse Proxy**
    - Handles HTTPS termination via Let's Encrypt or custom SSL
    - Forward traffic to Dockerized services (frontend/backend)
3. **Docker & Docker Compose**
    - Manages services including the app, NGINX, and optionally a database
    - Uses production-ready `.env.production` and `docker-compose.yml`
4. **GitHub Actions (CI/CD)**
    - Builds and optionally pushes Docker images to Docker Hub or AWS ECR
    - SSH into the EC2 server, pull the latest image, and restart services
5. **Secrets Management**
    - GitHub Actions Secrets for CI/CD credentials and environment variables
    - Optional integration with AWS Parameter Store for runtime secrets
6. **SSL & Domain Routing**
    - Let's Encrypt via Certbot for automatic SSL renewal
    - Optional DNS routing via Route 53 or Cloudflare
7. **Database Integration (Optional)**
    - PostgreSQL container or external DB like Supabase
    - Auto-run migrations during CI/CD using tooling like Alembic or Prisma

## **Tech Stack & Requirements**

This solution uses a modern DevOps and cloud-native stack to ensure a reliable and secure production deployment.

- **Infrastructure:**
    - AWS EC2 (Ubuntu 20.04 or later)
    - AWS VPC, Security Groups
    - Domain name (Route 53, Cloudflare, etc.)
- **Runtime & Tools:**
    - Docker 24+
    - Docker Compose v2+
    - NGINX (as reverse proxy)
    - Certbot (for Let's Encrypt SSL)
- **CI/CD:**
    - GitHub Actions
    - SSH (private key auth)
    - Docker Hub or AWS ECR (optional for image hosting)
- **Secrets Management:**
    - GitHub Actions Secrets
    - Optional: AWS Systems Manager Parameter Store
- **Database (Optional):**
    - PostgreSQL via Docker container
    - External DB service like Supabase
- **Optional Tools:**
    - Alembic (Python) or Prisma (Node.js) for DB migrations
    - AWS CLI (for automated provisioning)
    - Terraform (for IaC, if desired)

## **Directory Structure Explanation**

A clean and modular file structure helps ensure maintainability and ease of automation.

```
project-root/
│
├── .github/
│   └── workflows/
│       └── deploy.yml           # GitHub Actions CI/CD workflow
│
├── nginx/
│   └── default.conf             # NGINX reverse proxy configuration
│
├── docker-compose.yml          # Multi-service orchestration
├── Dockerfile                  # Application container build file
├── .env.production             # Environment-specific config vars
├── deploy.sh                   # Optional shell script for manual SSH deploy
├── certbot-init.sh             # SSL certificate bootstrap script
├── README.md                   # Project documentation
└── scripts/
    └── migrate.sh              # Optional DB migration hook
```

Each folder and file is designed to isolate concerns:

- `.github/workflows` handles CI/CD pipelines
- `nginx/` contains reverse proxy rules and SSL redirects
- Root contains build, deploy, and orchestration logic
- `scripts/` can be extended for migrations, backups, or monitoring

**Step-by-Step Setup Instructions:**

1. **Provision an EC2 Instance**
    - Use AWS Console or CLI to create an EC2 Ubuntu instance (20.04+)
    - Open only required ports in the Security Group (e.g., 22 for SSH, 80/443 for HTTP/HTTPS)
    - Associate an Elastic IP and attach a key pair for SSH access
2. **Install Dependencies on EC2**
    
    ```bash
    sudo apt update && sudo apt upgrade -y
    sudo apt install docker.io docker-compose certbot python3-certbot-nginx -y
    sudo usermod -aG docker ubuntu
    ```
    
    - Log out and log back in to activate Docker group access
3. **Clone the Repository**
    
    ```bash
    git clone <https://github.com/your-username/your-repo.git>
    cd your-repo
    ```
    
4. **Set Up Environment File**
    - Copy `.env.production.example` to `.env.production` and fill in variables
5. **Configure NGINX**
    - Replace the domain name in `nginx/default.conf`
    - Start containers:
    
    ```bash
    docker-compose up -d --build
    ```
    
6. **Enable SSL with Certbot**
    
    ```bash
    sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
    ```
    
    - Follow prompts to generate and auto-renew certificates
7. **Setup DNS Routing (Optional)**
    - Point your domain (via Route 53 or Cloudflare) to the EC2 Elastic IP
8. **Run Database Migrations (Optional)**
    
    ```bash
    bash scripts/migrate.sh
    ```
    

## **GitHub Actions CI/CD Pipeline**

This project includes a full GitHub Actions workflow to automate building and deploying the application.

`.github/workflows/deploy.yml`

```yaml
name: Deploy to EC2

on:
  push:
    branches:
      - main
      - release

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Code
        uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build and Push Image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: yourdockerhubuser/app:latest

      - name: SSH Deploy to EC2
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ubuntu
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            cd /home/ubuntu/your-repo
            git pull origin main
            docker-compose pull
            docker-compose up -d --build
```

Make sure the following secrets are configured in your GitHub repository:

- `DOCKER_USERNAME`
- `DOCKER_PASSWORD`
- `EC2_HOST`
- `EC2_SSH_KEY` (your private key, base64-encoded or multiline format)

**Security Best Practices:**

Security is a critical part of any production-grade infrastructure. Below are the recommended best practices for hardening your EC2 instance, protecting secrets, and securing the deployment pipeline:

1. **Port Restrictions:**
    - Allow only required ports in AWS Security Groups: 22 (SSH), 80 (HTTP), 443 (HTTPS).
    - Limit SSH access to specific IP addresses (your static IP) using inbound rules.
    - Disable unused ports and block all incoming traffic by default.
2. **SSH Key Handling:**
    - Never commit private keys to the repository.
    - Store SSH private keys securely using GitHub Secrets (`EC2_SSH_KEY`).
    - Use a strong key pair and rotate keys periodically.
    - Consider enabling fail2ban or similar intrusion prevention tools on the EC2 instance.
3. **NGINX Hardening Tips:**
    - Disable server tokens: `server_tokens off;` in `nginx.conf` to hide version info.
    - Limit request sizes and rate-limit sensitive endpoints.
    - Enforce HTTPS redirection using `return 301 https://$host$request_uri;`.
    - Use strong SSL protocols and ciphers; disable outdated TLS versions.
4. **CI/CD Pipeline Protection:**
    - Use GitHub Actions Secrets to avoid hardcoding credentials.
    - Protect `main` and `release` branches with review rules and CI checks.
    - Do not expose logs containing sensitive data (e.g., API keys or tokens).
5. **Environment Variables and Secrets:**
    - Do not include `.env.production` in source control; use `.env.production.example` instead.
    - Load secrets using GitHub Secrets or AWS Parameter Store securely during runtime.
    - For additional safety, use tools like `dotenv-vault`, `sops`, or HashiCorp Vault.
6. **SSL Certificate Renewal Automation:**
    - Use Certbot's systemd timer or cron job to auto-renew SSL:
        
        ```bash
        sudo systemctl status certbot.timer
        ```
        
    - You can also manually test auto-renewal using:
        
        ```bash
        sudo certbot renew --dry-run
        ```
        
7. **Logging and Monitoring:**
    - Enable logging in NGINX (`access.log` and `error.log`).
    - Monitor EC2 usage and set up alerts for abnormal CPU/memory/disk activity.
    - Use AWS CloudWatch or external logging systems (like Datadog or Loggly) for deeper visibility.
8. **Updates and Patch Management:**
    - Regularly run updates on the EC2 instance:
        
        ```bash
        sudo apt update && sudo apt upgrade -y
        ```
        
    - Subscribe to security mailing lists or alerts relevant to your stack.

By following these best practices, you significantly reduce the risk of unauthorized access, service disruptions, and data leakage in your production deployment.

## **Maintenance & Scaling Guide**

To ensure your deployment remains stable, scalable, and maintainable over time, follow these recommended practices for ongoing operations:

1. **Adding New Services:**
    - Update your `docker-compose.yml` with the new service definition.
    - If the service needs to be accessible via the web, add a corresponding NGINX location block.
    - Rebuild and redeploy using:
        
        ```bash
        docker-compose up -d --build
        ```
        
    - Update the CI/CD pipeline if new build or deployment steps are needed.
2. **Rotating Secrets:**
    - Generate new values for environment secrets or credentials.
    - Update `.env.production` on the server securely.
    - Update GitHub Actions Secrets via repository settings.
    - Restart services after secret changes:
        
        ```bash
        docker-compose down && docker-compose up -d
        ```
        
3. **Automating SSL Renewal:**
    - Certbot installs a systemd timer by default to auto-renew certificates.
    - You can confirm it’s active with:
        
        ```bash
        sudo systemctl list-timers | grep certbot
        ```
        
    - For custom automation, create a cron job:
        
        ```bash
        0 3 * * * certbot renew --quiet
        ```
        
4. **Database Migrations:**
    - Maintain a script (`scripts/migrate.sh`) that handles schema migrations.
    - Integrate it in your CI/CD or run it manually after deploys.
    - Example:
        
        ```bash
        docker-compose exec backend alembic upgrade head
        ```
        
5. **Scaling Horizontally:**
    - For load balancing, consider deploying behind AWS Application Load Balancer (ALB).
    - Use ECS or EKS for managed container orchestration at scale.
    - For stateful apps, ensure session management and data storage are handled externally (e.g., S3, RDS).
6. **Scaling Vertically:**
    - Upgrade EC2 instance type (t3.medium → t3.large, etc.) based on CPU/RAM usage.
    - Use AWS CloudWatch alarms to monitor thresholds and send alerts.
7. **Backup Strategy:**
    - Back-up database volumes regularly using cron + `pg_dump` or managed RDS snapshotting.
    - Store `.env.production` backups securely outside the EC2 instance.
    - Optionally, version backups in S3 with lifecycle rules.
8. **Disaster Recovery Plan:**
    - Use Infrastructure as Code (Terraform/CloudFormation) to recreate environments.
    - Maintain clear documentation for redeploying from scratch.
    - Periodically test recovery procedures to validate readiness.
9. **Dependency Updates:**
    - Monitor Docker base images for updates.
    - Use Dependabot for GitHub Actions and Dockerfile dependencies.
    - Rebuild images regularly to patch vulnerabilities.
10. **Log Rotation and Storage:**
- Configure log rotation for Docker and NGINX logs.
- Store rotated logs in a persistent volume or external logging service.
- Example for NGINX logrotate config:
    
    ```
    /var/log/nginx/*.log {
      daily
      missingok
      rotate 14
      compress
      delaycompress
      notifempty
      create 640 www-data adm
      sharedscripts
      postrotate
        systemctl reload nginx
      endscript
    }
    ```
    

Following this maintenance and scaling guide will ensure your system stays healthy, secure, and scalable as user demands and project complexity grow.

## **Bonus: Database Migration Handling**

Managing database schema changes is crucial in any production deployment. A well-structured migration process ensures your application stays in sync with the database without downtime.

1. **Recommended Tools:**
    - **Python Projects:** Use [Alembic](https://alembic.sqlalchemy.org/)
    - **Node.js Projects:** Use [Prisma Migrate](https://www.prisma.io/docs/concepts/components/prisma-migrate)
    - **Go Projects:** Use [golang-migrate](https://github.com/golang-migrate/migrate)
    - **General SQL:** Use [Flyway](https://flywaydb.org/) or [Liquibase](https://www.liquibase.org/)
2. **Folder Structure:**
Organize migration files within a `migrations/` folder:
    
    ```
    project-root/
    ├── migrations/
    │   ├── versions/
    │   └── env.py
    └── scripts/
        └── migrate.sh
    
    ```
    
3. **Migration Script (`scripts/migrate.sh`):**
Example using Alembic:
    
    ```bash
    #!/bin/bash
    echo "Running DB migrations..."
    docker-compose exec backend alembic upgrade head
    
    ```
    
    Make the script executable:
    
    ```bash
    chmod +x scripts/migrate.sh
    
    ```
    
4. **Integrating with CI/CD:**
Add a step in your GitHub Actions `deploy.yml` to run the migration after deployment:
    
    ```yaml
    - name: Run DB Migrations
      run: |
        ssh -o StrictHostKeyChecking=no ubuntu@${{ secrets.EC2_HOST }} << 'EOF'
          cd /home/ubuntu/your-repo
          bash scripts/migrate.sh
        EOF
    
    ```
    
5. **Production Considerations:**
    - Ensure migrations are idempotent.
    - Always back up the database before applying critical changes.
    - Run migrations in a staging environment before production.
    - Monitor logs during and after migration for rollback needs.

By automating and version-controlling your database migrations, you minimize human error and reduce risks during deployment cycles.

## **License**

This project is licensed under the MIT License.

You are free to use, modify, and distribute this codebase for both personal and commercial projects.

**MIT License Summary**

- No liability or warranty is provided.
- You must include the original license and copyright.
- You may reuse the code in closed-source applications.

For full legal terms, see the [LICENSE](./LICENSE) file included in this repository.

## **Live Deployment Demo or Blog**

A complete walkthrough and hands-on demonstration of this project is coming soon.

- **Live Blog Post:** Coming Soon
- **Demo Video (Optional):** Coming Soon
- **Screenshots & Diagrams:** Coming Soon

These resources will help you follow along step by step, debug setup issues, and understand the reasoning behind key architectural choices.

Feel free to fork this repository and use it as a boilerplate for your production deployments.
