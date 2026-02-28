# GOST HTTP/2 Proxy Server Template

A Docker-based GOST (GO Simple Tunnel) proxy server template with automatic SSL certificate management via Let's Encrypt and Cloudflare DNS validation.

## Overview

This project provides a ready-to-deploy HTTP/2 proxy server using:
- **GOST**: High-performance tunnel proxy written in Go
- **Certbot**: Automatic SSL/TLS certificate management with Let's Encrypt
- **Cloudflare DNS**: DNS-01 challenge for domain validation
- **Docker Compose**: Easy deployment and orchestration

## Prerequisites

Before getting started, ensure you have:

1. **Domain Name on Cloudflare**
   - A domain managed through Cloudflare
   - Cloudflare API token with DNS edit permissions
   - [Create API token here](https://dash.cloudflare.com/profile/api-tokens)

2. **VPS (Virtual Private Server)**
   - **Recommended Specs**: 20GB storage, 1GB RAM, 1TB bandwidth
   - **Recommended Provider**: Bandwagon Host (搬瓦工) with CN2 GIA network
   - SSH access with root privileges
   - Public IP address

## VPS Initialization

### Step 1: Install Fresh Ubuntu OS

1. Access your VPS provider's dashboard (e.g., Bandwagon Host)
2. Stop the VPS instance
3. Install **Ubuntu 22.04 LTS**
4. Note down the root password provided

### Step 2: Initial System Setup

SSH into your VPS as root and run the following commands:

```bash
# Restore full Ubuntu installation
unminimize

# Create a non-root user for maintenance
useradd -m -s /bin/bash maintain

# Install prerequisites
apt-get install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```

### Step 3: Install Docker and Docker Compose

```bash
# Set up Docker's official GPG key and repository
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null

# Update package index
apt-get update

# Install Docker and Docker Compose
apt-get install -y \
    docker-ce \
    docker-ce-cli \
    containerd.io \
    docker-buildx-plugin \
    docker-compose-plugin

# Enable Docker service
systemctl start docker
systemctl enable docker

# Add maintain user to docker and sudo groups
usermod -aG docker maintain
usermod -aG sudo maintain

# Verify installation
docker --version
docker compose version

# Set password for maintain user
passwd maintain

# Switch to maintain user
su maintain
```

### Step 4: Configure SSH Key Authentication

```bash
# Switch to maintain user if not already
cd ~

# Set up SSH authorized keys
mkdir -p ~/.ssh
touch ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys

# Add your public SSH key (paste your key and press Ctrl+D)
cat > ~/.ssh/authorized_keys

# Disable password authentication for security
sudo sed -i '$a PasswordAuthentication no' /etc/ssh/sshd_config
sudo sshd -t -f /etc/ssh/sshd_config
sudo systemctl reload ssh
```

### Step 5: Install GitHub CLI (Optional but Recommended)

```bash
(type -p wget >/dev/null || (sudo apt update && sudo apt install wget -y)) \
    && sudo mkdir -p -m 755 /etc/apt/keyrings \
    && out=$(mktemp) && wget -nv -O$out https://cli.github.com/packages/githubcli-archive-keyring.gpg \
    && cat $out | sudo tee /etc/apt/keyrings/githubcli-archive-keyring.gpg > /dev/null \
    && sudo chmod go+r /etc/apt/keyrings/githubcli-archive-keyring.gpg \
    && sudo mkdir -p -m 755 /etc/apt/sources.list.d \
    && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null \
    && sudo apt update \
    && sudo apt install gh -y
```

## Configuration

### Step 1: Clone Repository

```bash
# Clone this repository to your VPS
git clone https://github.com/YOUR_USERNAME/gost-template.git
cd gost-template
```

### Step 2: Update Configuration Files

You need to update the following configuration values in three files:

#### 1. `gost/cloudflare.ini`

| Configuration | Description | Example |
|--------------|-------------|---------|
| `dns_cloudflare_api_token` | Your Cloudflare API token with DNS edit permissions | `abc123def456ghi789jkl012mno345` |

**How to get Cloudflare API Token:**
1. Go to [Cloudflare Dashboard](https://dash.cloudflare.com/profile/api-tokens)
2. Click "Create Token"
3. Use "Edit zone DNS" template
4. Select your domain under "Zone Resources"
5. Create and copy the token

**File content:**
```ini
dns_cloudflare_api_token = YOUR_CLOUDFLARE_API_TOKEN_HERE
```

#### 2. `docker-compose.yml`

| Configuration | Description | Example |
|--------------|-------------|---------|
| `YOUR_CLOUDFLARE_MAIL_HERE` | Your Cloudflare account email | `user@example.com` |
| `YOUR_FULL_DOMAIN_HERE_FOR_EXAMPLE_DOCS_DOT_FACEBOOK_DOT_COM` | Full domain name for SSL certificate (must be managed by Cloudflare) | `proxy.yourdomain.com` |

**Replace in certbot command section (lines 27-28):**
```yaml
- -m YOUR_CLOUDFLARE_MAIL_HERE          # Change to your email
- -d YOUR_FULL_DOMAIN_HERE_FOR_EXAMPLE_DOCS_DOT_FACEBOOK_DOT_COM  # Change to your domain
```

#### 3. `gost/config.yaml`

| Configuration | Description | Example |
|--------------|-------------|---------|
| `YOUR_PASSWORD_HERE_AND_LEAVE_USERNAME_EMPTY` | Strong password for proxy authentication (username should remain empty) | `MySecureP@ssw0rd123!` |
| `YOUR_FULL_DOMAIN_HERE_FOR_EXAMPLE_DOCS_DOT_FACEBOOK_DOT_COM` | Full domain name (must match docker-compose.yml) | `proxy.yourdomain.com` |

**Replace in two locations:**
```yaml
auth:
  username:                                    # Leave empty
  password: YOUR_PASSWORD_HERE_AND_LEAVE_USERNAME_EMPTY  # Change to your password

tls:
  certFile: /etc/letsencrypt/live/YOUR_FULL_DOMAIN_HERE_FOR_EXAMPLE_DOCS_DOT_FACEBOOK_DOT_COM/fullchain.pem
  keyFile: /etc/letsencrypt/live/YOUR_FULL_DOMAIN_HERE_FOR_EXAMPLE_DOCS_DOT_FACEBOOK_DOT_COM/privkey.pem
```

### Configuration Summary

Here's a quick checklist of all values you need to replace:

- [ ] **Cloudflare API Token** in `gost/cloudflare.ini`
- [ ] **Email Address** in `docker-compose.yml`
- [ ] **Domain Name** in `docker-compose.yml` (1 occurrence)
- [ ] **Proxy Password** in `gost/config.yaml`
- [ ] **Domain Name** in `gost/config.yaml` (2 occurrences in cert paths)

## Deployment

### Step 1: Obtain SSL Certificate

First, run Certbot to obtain the SSL certificate via DNS validation:

```bash
docker compose pull
docker compose up certbot
```

Wait for the certificate to be issued. You should see a success message indicating the certificate was obtained.

### Step 2: Start All Services

```bash
docker compose up -d
```

This will start the GOST proxy server in detached mode.

### Step 3: Verify Services

```bash
# Check running containers
docker compose ps

# View logs
docker compose logs -f gost

# Check if port 443 is listening
sudo netstat -tulpn | grep 443
```

## Usage

### Client Configuration

Configure your GOST client or any HTTP/2 compatible proxy client with:

- **Server**: `https://YOUR_DOMAIN:443`
- **Protocol**: HTTP/2
- **Authentication**:
  - Username: (leave empty)
  - Password: `YOUR_PASSWORD`

### Example GOST Client Command

```bash
gost -L=:1080 -F=h2://YOUR_PASSWORD@YOUR_DOMAIN:443
```

This creates a local SOCKS5 proxy on port 1080 that forwards traffic through your server.

## Maintenance

### Renew SSL Certificate

Certificates from Let's Encrypt are valid for 90 days. To renew:

```bash
docker compose up certbot
docker compose restart gost
```

Consider setting up a cron job for automatic renewal:

```bash
# Edit crontab
crontab -e

# Add this line to renew every month
0 0 1 * * cd /path/to/gost-template && docker compose up certbot && docker compose restart gost
```

### Update Services

```bash
# Pull latest images
docker compose pull

# Recreate containers
docker compose up -d
```

### View Logs

```bash
# All services
docker compose logs -f

# Specific service
docker compose logs -f gost
docker compose logs -f certbot
```

### Stop Services

```bash
# Stop all services
docker compose down

# Stop and remove volumes
docker compose down -v
```

## Troubleshooting

### Certificate Not Obtained

- Verify your Cloudflare API token has DNS edit permissions
- Ensure the domain is properly configured in Cloudflare
- Check that the domain is not proxied (orange cloud) in Cloudflare DNS settings during initial setup

### GOST Service Won't Start

- Ensure the certificate was successfully obtained first
- Verify domain names match in both `docker-compose.yml` and `gost/config.yaml`
- Check logs: `docker compose logs gost`

### Port 443 Already in Use

```bash
# Find what's using port 443
sudo netstat -tulpn | grep 443

# Stop the conflicting service or change GOST port in docker-compose.yml
```

### Cannot Connect to Proxy

- Verify firewall allows port 443: `sudo ufw allow 443/tcp`
- Check if GOST is running: `docker compose ps`
- Verify password is correct
- Ensure client is using HTTP/2 protocol

## Security Recommendations

1. **Use Strong Passwords**: Generate a random, strong password for proxy authentication
2. **Keep System Updated**: Regularly update Ubuntu and Docker
3. **Enable Firewall**: Configure UFW to only allow necessary ports
4. **Monitor Logs**: Regularly check logs for suspicious activity
5. **Rotate API Tokens**: Periodically regenerate Cloudflare API tokens
6. **Disable Root Login**: Never allow root SSH access
7. **Use SSH Keys**: Disable password-based SSH authentication

## License

This project is provided as-is for educational and personal use.

## Contributing

Feel free to submit issues and pull requests to improve this template.

## Acknowledgments

- [GOST](https://github.com/ginuerzh/gost) - GO Simple Tunnel
- [Certbot](https://certbot.eff.org/) - Let's Encrypt client
- [Cloudflare](https://www.cloudflare.com/) - DNS and CDN services
