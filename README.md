# SomaBox Deployment Guide - Remote Setup via SSH

## Step 1: Check Current System Status

SSH into the remote system and run these commands:
```bash
echo "=== Operating System ==="
cat /etc/os-release

echo ""
echo "=== System Check ==="
echo "Apache: $(apache2 -v 2>&1 || httpd -v 2>&1 || echo 'NOT INSTALLED')"
echo "Node.js: $(node -v 2>&1 || echo 'NOT INSTALLED')"
echo "npm: $(npm -v 2>&1 || echo 'NOT INSTALLED')"
echo "pm2: $(pm2 -v 2>&1 || echo 'NOT INSTALLED')"

echo ""
echo "=== What's Running on Port 80? ==="
sudo lsof -i :80 || echo "Nothing running"

echo ""
echo "=== Apache Status ==="
sudo systemctl status apache2 2>&1 || sudo systemctl status httpd 2>&1
```

**Note:** From the output, determine if the system uses `apache2` (Debian/Ubuntu) or `httpd` (RedHat/CentOS). Use the appropriate command throughout this guide.

---

## Step 2: Install Required Software

### Install Node.js and npm

**For Debian/Ubuntu:**
```bash
sudo apt update
sudo apt install nodejs npm -y
```

**For RedHat/CentOS:**
```bash
sudo yum install nodejs npm -y
# Or for newer versions
sudo dnf install nodejs npm -y
```

**Verify installation:**
```bash
node -v
npm -v
```

### Install PM2 globally
```bash
sudo npm install -g pm2
pm2 -v
```

### Ensure Apache is installed and running

**For Debian/Ubuntu:**
```bash
sudo apt install apache2 -y
sudo systemctl start apache2
sudo systemctl enable apache2
```

**For RedHat/CentOS:**
```bash
sudo yum install httpd -y
sudo systemctl start httpd
sudo systemctl enable httpd
```

---

## Step 3: Upload Project Files via FileZilla

1. **Connect via SFTP** in FileZilla using SSH credentials
2. **Navigate to:** `/var/www/html/`
3. **Upload folders:**
   - `somabox-frontend/`
   - `somabox-backend/`

**Set proper permissions after upload:**
```bash
cd /var/www/html/
sudo chown -R $USER:$USER somabox-frontend somabox-backend
chmod -R 755 somabox-frontend somabox-backend
```

---

## Step 4: Install Project Dependencies

### Backend
```bash
cd /var/www/html/somabox-backend
npm install
```

**Verify backend configuration:**
- Check that your backend is configured to run on **port 5000**
- Check `.env` file or config file for port settings

### Frontend
```bash
cd /var/www/html/somabox-frontend
npm install
npm run build  # Build Next.js for production
```

---

## Step 5: Configure Apache as Reverse Proxy

### Enable required Apache modules

**For Debian/Ubuntu:**
```bash
sudo a2enmod proxy
sudo a2enmod proxy_http
sudo a2enmod proxy_wstunnel
sudo systemctl restart apache2
```

**For RedHat/CentOS:**
```bash
# Modules are usually enabled by default, but verify:
sudo systemctl restart httpd
```

### Edit Apache configuration

**For Debian/Ubuntu:**
```bash
sudo nano /etc/apache2/sites-available/000-default.conf
```

**For RedHat/CentOS:**
```bash
sudo nano /etc/httpd/conf.d/somabox.conf
```

**Add this configuration inside `<VirtualHost *:80>` block:**
```apache
<VirtualHost *:80>
    ServerAdmin admin@somabox.local
    
    # Backend API - runs on port 5000
    ProxyPass /api http://localhost:5000/api
    ProxyPassReverse /api http://localhost:5000/api
    
    # Frontend - runs on port 3000
    ProxyPass / http://localhost:3000/
    ProxyPassReverse / http://localhost:3000/
    
    # WebSocket support (if needed)
    ProxyPass /ws ws://localhost:3000/ws
    ProxyPassReverse /ws ws://localhost:3000/ws
    
    ErrorLog ${APACHE_LOG_DIR}/somabox-error.log
    CustomLog ${APACHE_LOG_DIR}/somabox-access.log combined
</VirtualHost>
```

**For RedHat/CentOS, adjust log paths:**
```apache
ErrorLog /var/log/httpd/somabox-error.log
CustomLog /var/log/httpd/somabox-access.log combined
```

### Test and restart Apache

**For Debian/Ubuntu:**
```bash
sudo apache2ctl configtest
sudo systemctl restart apache2
```

**For RedHat/CentOS:**
```bash
sudo apachectl configtest
sudo systemctl restart httpd
```

---

## Step 6: Start Applications with PM2

### Start Backend
```bash
cd /var/www/html/somabox-backend
pm2 start npm --name "somabox-backend" -- start
# Or if you have a specific entry file:
# pm2 start server.js --name "somabox-backend"
```

### Start Frontend
```bash
cd /var/www/html/somabox-frontend
pm2 start npm --name "somabox-frontend" -- start
```

### Verify processes are running
```bash
pm2 list
pm2 logs  # Check logs for any errors
```

**You should see both processes running with status "online"**

---

## Step 7: Configure PM2 to Start on Boot

### Save current PM2 process list
```bash
pm2 save
```

### Generate startup script
```bash
pm2 startup
```

**Important:** PM2 will output a command that looks like this:
```bash
sudo env PATH=$PATH:/usr/bin /usr/lib/node_modules/pm2/bin/pm2 startup systemd -u username --hp /home/username
```

**Copy and run that exact command.** It configures PM2 to start automatically on system reboot.

### Verify startup configuration
```bash
sudo systemctl status pm2-$USER
```

---

## Step 8: Test Everything

### Check PM2 processes
```bash
pm2 list
pm2 logs --lines 50
```

### Check if ports are listening
```bash
sudo lsof -i :3000  # Should show node (frontend)
sudo lsof -i :5000  # Should show node (backend)
sudo lsof -i :80    # Should show apache2/httpd
```

### Test from browser
1. Connect to the WiFi
2. Open browser and navigate to the device IP address
3. You should see your Next.js frontend
4. Test API calls to `/api/*` endpoints

### Test reboot persistence
```bash
sudo reboot
```

After reboot, SSH back in and check:
```bash
pm2 list  # Should show both processes running
sudo systemctl status apache2  # or httpd - should be active
```

---

## Step 9: Useful PM2 Commands for Management
```bash
# View logs
pm2 logs somabox-frontend
pm2 logs somabox-backend

# Restart applications
pm2 restart somabox-frontend
pm2 restart somabox-backend
pm2 restart all

# Stop applications
pm2 stop somabox-frontend
pm2 stop somabox-backend

# Delete from PM2 (stop and remove)
pm2 delete somabox-frontend
pm2 delete somabox-backend

# Monitor resources
pm2 monit

# Show detailed info
pm2 show somabox-frontend
```

---

## Troubleshooting

### Frontend not accessible
```bash
pm2 logs somabox-frontend
cd /var/www/html/somabox-frontend
npm run build  # Rebuild if needed
pm2 restart somabox-frontend
```

### Backend API not responding
```bash
pm2 logs somabox-backend
# Check backend port configuration
cat /var/www/html/somabox-backend/.env
pm2 restart somabox-backend
```

### Apache proxy not working
```bash
# Check Apache error logs
# Debian/Ubuntu:
sudo tail -f /var/log/apache2/somabox-error.log
# RedHat/CentOS:
sudo tail -f /var/log/httpd/somabox-error.log

# Test Apache config
sudo apache2ctl configtest  # or apachectl configtest
```

### PM2 not starting on boot
```bash
# Regenerate startup script
pm2 unstartup
pm2 startup
# Run the command it outputs
pm2 save
```

---

## Quick Reference: Full Installation Script
```bash
#!/bin/bash
# Run this on a fresh system

# Install Node.js, npm, Apache
sudo apt update && sudo apt install nodejs npm apache2 -y  # Debian/Ubuntu
# OR
# sudo yum install nodejs npm httpd -y  # RedHat/CentOS

# Install PM2
sudo npm install -g pm2

# Enable Apache modules (Debian/Ubuntu)
sudo a2enmod proxy proxy_http proxy_wstunnel
sudo systemctl restart apache2

# Set permissions
cd /var/www/html/
sudo chown -R $USER:$USER somabox-frontend somabox-backend

# Install dependencies
cd somabox-backend && npm install && cd ..
cd somabox-frontend && npm install && npm run build && cd ..

# Start with PM2
cd somabox-backend && pm2 start npm --name "somabox-backend" -- start && cd ..
cd somabox-frontend && pm2 start npm --name "somabox-frontend" -- start && cd ..

# Configure startup
pm2 save
pm2 startup  # Run the command it outputs

echo "Installation complete! Configure Apache reverse proxy next."
```

---

## Notes

- **Firewall:** Ensure port 80 is open if using a firewall
- **SELinux:** On RedHat/CentOS, if SELinux is enabled, you may need to configure it to allow Apache to proxy:
```bash
  sudo setsebool -P httpd_can_network_connect 1
```
- **Updates:** To update the application, upload new files via FileZilla, run `npm install` if dependencies changed, rebuild frontend with `npm run build`, then `pm2 restart all`

---

**Deployment Status Checklist:**
- [ ] Apache installed and running
- [ ] Node.js and npm installed
- [ ] PM2 installed globally
- [ ] Project files uploaded to `/var/www/html/`
- [ ] Dependencies installed (`npm install`)
- [ ] Frontend built (`npm run build`)
- [ ] Apache reverse proxy configured
- [ ] Both apps running in PM2 (`pm2 list` shows "online")
- [ ] PM2 startup configured (`pm2 startup` + `pm2 save`)
- [ ] Tested from browser
- [ ] Tested after reboot