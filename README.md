
# Hosting your .Net Applicaitons

## 1. Prerequisities
https://learn.microsoft.com/en-us/dotnet/core/install/linux-debian?tabs=dotnet10#debian-12

```sh
# Install bashed on your distro and version required
wget https://packages.microsoft.com/config/debian/12/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
rm packages-microsoft-prod.deb
```
```sh
# sdk
sudo apt-get update && \
  sudo apt-get install -y dotnet-sdk-10.0

# rutime
sudo apt-get update && \
  sudo apt-get install -y aspnetcore-runtime-10.0

# test
dotnet --version
```

## 2. Creating/running/publishing your applications
https://learn.microsoft.com/en-us/troubleshoot/developer/webapps/aspnetcore/practice-troubleshoot-linux/2-1-create-configure-aspnet-core-applications

```sh
# dotnet new <template_type> -n <project_name> -o <output_directory>
dotnet new webapp -n AspNetCoreDemo -o firstwebapp
cd firstwebapp
dotnet run


# Running the application uat
dotnet publish
sudo cp -a bin/Debug/net5.0/publish/ /var/merodotnet/firstwebapp/
dotnet /var/firstwebapp/AspNetCoreDemo.dll


# Running the application live
dotnet publish --configuration Release
sudo cp -a bin/Release/net5.0/publish/ /var/merodotnet/firstwebapp/
dotnet /var/firstwebapp/AspNetCoreDemo.dll
```

## 3. Background Service
```sh
# vim /etc/systemd/system/merodotent.service
[Unit]
Description=Example .NET Web API App running on Ubuntu

[Service]
WorkingDirectory=/var/merodotnet/publish
ExecStart=/usr/bin/dotnet /var/merodotnet/publish/AspNetCoreDemo.dll
Restart=always
# Restart service after 10 seconds if the dotnet service crashes:
RestartSec=10
KillSignal=SIGINT
SyslogIdentifier=dotnet-example
User=www-data
Environment=ASPNETCORE_ENVIRONMENT=Production
Environment=DOTNET_PRINT_TELEMETRY_MESSAGE=false

[Install]
WantedBy=multi-user.target
```

**Using Custom Env**
```sh
[Unit]
Description=MeroDotNet API
After=network.target

[Service]
Type=simple

WorkingDirectory=/var/merodotnet/publish

ExecStart=/usr/bin/dotnet /var/merodotnet/publish/AspNetCoreDemo.dll

Restart=always
RestartSec=5

User=www-data
Group=www-data

KillSignal=SIGINT
SyslogIdentifier=merodotnet-api

Environment=ASPNETCORE_ENVIRONMENT=Production
Environment=DOTNET_PRINT_TELEMETRY_MESSAGE=false

# Custom app variables
Environment=DB_HOST=localhost
Environment=DB_PORT=5432
Environment=JWT_SECRET=supersecret

# Load external env file
EnvironmentFile=/etc/merodotnet/merodotnet.env

# Optional custom config path
Environment=CONFIG_PATH=/etc/merodotnet/production.json

[Install]
WantedBy=multi-user.target
```

## 4. Hosting your applications
## 4.1 HTTP Hosting
https://learn.microsoft.com/en-us/troubleshoot/developer/webapps/aspnetcore/practice-troubleshoot-linux/2-2-install-nginx-configure-it-reverse-proxy#edit-the-configuration-file-by-using-vi

```nginx
  server {
    listen        80;
    server_name _;
    location / {
        proxy_pass         http://localhost:5000;
        proxy_http_version 1.1;
        proxy_set_header   Upgrade $http_upgrade;
        proxy_set_header   Connection $connection_upgrade;
        proxy_set_header   Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
    }
}
```

## 4.2 SSL
https://certbot.eff.org/instructions?ws=nginx&os=pip
```sh
# 1. Update server packages
sudo apt update

# 2. Install certbot and nginx plugin
sudo apt install -y certbot python3-certbot-nginx

# 3. Verify nginx is running
sudo systemctl status nginx

# 4. Test nginx configuration before generating SSL
sudo nginx -t

# 5. Allow HTTP + HTTPS through firewall (or iptables or other firewalls present)
sudo ufw allow 'Nginx Full'

# 6. Generate SSL certificate for domain
# Replace property.test.com with your actual domain
sudo certbot --nginx -d property.test.com

# 7. Generate SSL for multiple domains (optional)
sudo certbot --nginx \
  -d property.test.com \
  -d www.property.test.com

# 8. Check installed certificates
sudo certbot certificates

# 9. Verify certbot auto-renew timer status
sudo systemctl status certbot.timer

# 10. Enable certbot timer manually if not enabled
sudo systemctl enable certbot.timer
sudo systemctl start certbot.timer

# 11. Check certbot renewal schedule
systemctl list-timers | grep certbot

# 12. Test automatic renewal with dry-run
# This simulates renewal without changing certificate
sudo certbot renew --dry-run

# 13. Manually renew certificates
sudo certbot renew

# 14. Delete certificate (optional)
sudo certbot delete --cert-name property.test.com

# 15. Reinstall SSL config for existing certificate (optional)
sudo certbot install --cert-name property.test.com

# 16. Check DNS if SSL generation fails
dig property.test.com

# 17. Alternative DNS check
nslookup property.test.com

# 18. Reload nginx after config changes
sudo systemctl reload nginx

# 19. Restart nginx if needed
sudo systemctl restart nginx

```

**Notes:**
* *Let's Encrypt certificates expire every 90 days*
* *Certbot automatically renews certificates before expiry*
* *Renewal runs automatically using systemd timer*
* *HTTPS redirect is usually configured automatically by certbot*

## 5. Deploy and RollBack
https://www.w3schools.com/bash/

### Backup
```sh
#!/bin/bash

# =========================================================
# Deployment Script
# Publishes and deploys ASP.NET Core application
# =========================================================

PROJECT_DIR="{project-base-dir}"
PUBLISH_DIR="$PROJECT_DIR/bin/Release/net5.0/publish"
DEPLOY_DIR="/var/merodotnet/firstwebapp"
SERVICE_NAME="merodotent.service"

echo "Starting deployment..."

# Go to project directory
cd "$PROJECT_DIR" || exit 1

# Optional: Pull latest changes
# git pull

# Build & publish project
dotnet publish --configuration Release

# Create backup before deployment
./backup.sh

# Copy newly published files
sudo cp -r "$PUBLISH_DIR" "$DEPLOY_DIR"
echo "Application deployed successfully."

# Restart service
sudo systemctl restart "$SERVICE_NAME"
echo "Service restarted."
```

## 5.2 Resotre
```sh
#!/bin/bash

# =========================================================
# Restore Script
# Restores application from backup
# =========================================================

APP_DIR="/var/merodotnet/firstwebapp"
BACKUP_DIR="/var/merodotnet/firstwebapp.old"
SERVICE_NAME="merodotent.service"

echo "Starting restore process..."

# Check if backup exists
if [ ! -d "$BACKUP_DIR" ]; then
    echo "No backup found."
    exit 1
fi

# Remove broken/current deployment if it exists
if [ -d "$APP_DIR" ]; then
    rm -rf "$APP_DIR"
    echo "Current deployment removed."
fi

# Restore backup
mv "$BACKUP_DIR" "$APP_DIR"

echo "Backup restored successfully."

# Restart service
sudo systemctl restart "$SERVICE_NAME"

echo "Service restarted."
```

## 7. Auto Deploy
```sh
# Convert above shell to ansible script
```

## 8. Dockerize Applications
```sh
# Create docker for above with the requirements
```

## 6. Security
For the Dev team

**1. Key Encryption**
```sh
https://learn.microsoft.com/en-us/aspnet/core/security/data-protection/implementation/key-encryption-at-rest?view=aspnetcore-10.0
```
