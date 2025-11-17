# TravianZ on Orange Pi 5 - Quick Setup Guide

This guide provides step-by-step instructions for running TravianZ on Orange Pi 5 (ARM64 architecture).

## Hardware Requirements

- **Orange Pi 5** (or Orange Pi 5 Plus)
- At least 4GB RAM (8GB recommended)
- 16GB+ microSD card or eMMC storage (SSD recommended for production)
- Network connection (Ethernet recommended)
- Power supply (5V/4A recommended)

## Operating System Preparation

### Recommended OS
- **Ubuntu 20.04/22.04 LTS** for Orange Pi 5
- **Debian 11/12** also works well

### Install Docker on Orange Pi 5

1. Update your system:
```bash
sudo apt-get update
sudo apt-get upgrade -y
```

2. Install Docker:
```bash
# Install required packages
sudo apt-get install -y ca-certificates curl gnupg lsb-release

# Add Docker's official GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Set up Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

3. Add your user to the docker group:
```bash
sudo usermod -aG docker $USER
newgrp docker
```

4. Verify installation:
```bash
docker --version
docker compose version
docker info | grep Architecture
# Should show: Architecture: aarch64
```

## TravianZ Installation

### 1. Clone the Repository
```bash
cd ~
git clone https://github.com/Shadowss/TravianZ.git
cd TravianZ
```

### 2. Configure Environment
```bash
cp .env.example .env
```

Edit `.env` if you want to change default passwords:
```bash
nano .env
```

Recommended settings for Orange Pi 5:
```env
MYSQL_ROOT_PASSWORD=your_secure_root_password
MYSQL_DATABASE=travian
MYSQL_USER=travianz
MYSQL_PASSWORD=your_secure_password
```

### 3. Start TravianZ
```bash
docker compose up -d
```

**First-time setup takes 5-10 minutes** as Docker downloads ARM64 images.

Monitor the startup:
```bash
docker compose logs -f
```

Press `Ctrl+C` to stop following logs.

### 4. Verify Services
```bash
docker compose ps
```

You should see three containers running:
- `travianz-web`
- `travianz-db`
- `travianz-phpmyadmin`

### 5. Access the Installation Wizard

Open a browser and navigate to:
```
http://<orange-pi-ip>:8080/install
```

To find your Orange Pi's IP address:
```bash
hostname -I | awk '{print $1}'
```

### 6. Complete Installation

Use these database settings in the installation wizard:

| Setting | Value |
|---------|-------|
| SQL Hostname | `db` |
| Port | `3306` |
| Username | `travianz` (or your custom value from .env) |
| Password | `travianzpass` (or your custom value from .env) |
| DB name | `travian` (or your custom value from .env) |
| Prefix | `s1_` |
| Type | `MYSQLi` |

## Performance Optimization for Orange Pi 5

### 1. Adjust MySQL Memory (Recommended)

Edit `docker-compose.yml` and modify the db service command:

```yaml
services:
  db:
    command: >
      --default-authentication-plugin=mysql_native_password
      --sql_mode=""
      --innodb_buffer_pool_size=512M
      --max_connections=100
```

For 4GB Orange Pi 5, use 256M-512M buffer pool.
For 8GB+ Orange Pi 5, use 1G-2G buffer pool.

Restart after changes:
```bash
docker compose down
docker compose up -d
```

### 2. Use Fast Storage

For best performance:
- Use **eMMC** instead of microSD card
- Or use **NVMe SSD** via PCIe (Orange Pi 5 Plus)
- Or use **USB 3.0 SSD** (Orange Pi 5)

### 3. Enable Swap (Optional, for 4GB models)

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

## Accessing Services

- **Game**: `http://<orange-pi-ip>:8080`
- **phpMyAdmin**: `http://<orange-pi-ip>:8081`
- **MySQL** (external): `<orange-pi-ip>:3306`

## Container Management

### View logs
```bash
docker compose logs -f web      # Web server logs
docker compose logs -f db       # Database logs
```

### Restart services
```bash
docker compose restart
```

### Stop services
```bash
docker compose down
```

### Update TravianZ
```bash
cd ~/TravianZ
git pull origin main
docker compose down
docker compose up -d --build
```

## Troubleshooting

### Issue: "Cannot connect to database"
**Solution**: Wait 1-2 minutes for MySQL to fully initialize on first start.
```bash
docker compose logs db
```

### Issue: "Platform mismatch" error
**Solution**: Verify Docker is using ARM64:
```bash
docker info | grep Architecture
# Should show: Architecture: aarch64
```

### Issue: Slow performance
**Solutions**:
1. Use SSD/eMMC instead of microSD
2. Reduce MySQL buffer pool size if you have limited RAM
3. Check available memory: `free -h`

### Issue: Permission errors
**Solution**:
```bash
docker exec -it travianz-web chown -R www-data:www-data /var/www/html
docker exec -it travianz-web chmod -R 777 /var/www/html/var
```

### Issue: Port already in use
**Solution**: Check what's using the port:
```bash
sudo netstat -tlnp | grep :8080
```

Change ports in `docker-compose.yml` if needed:
```yaml
ports:
  - "9080:80"  # Change 8080 to 9080
```

## Backup

### Backup database
```bash
docker exec travianz-db mysqldump -u root -p travian > backup_$(date +%Y%m%d).sql
```

### Backup all data
```bash
cd ~
tar -czf travianz_backup_$(date +%Y%m%d).tar.gz \
  --exclude='TravianZ/.git' \
  TravianZ/
```

## Remote Access

To access from other devices on your network, you may need to:

1. Configure Orange Pi 5 firewall:
```bash
sudo ufw allow 8080/tcp
sudo ufw allow 8081/tcp
sudo ufw allow 3306/tcp
```

2. Find your Orange Pi's IP:
```bash
hostname -I
```

3. Access from any device on the network:
```
http://<orange-pi-ip>:8080
```

## Additional Resources

- [Main Docker Documentation](DOCKER_README.md)
- [GitHub Issues](https://github.com/Shadowss/TravianZ/issues)
- [Orange Pi 5 Official Documentation](http://www.orangepi.org/html/hardWare/computerAndMicrocontrollers/service-and-support/Orange-Pi-5.html)

## Support

For issues specific to Orange Pi 5:
1. Check this guide's troubleshooting section
2. Review Docker logs: `docker compose logs`
3. Create an issue on GitHub with your Orange Pi 5 model and OS version

Happy gaming! 🎮
