# Initial Security Setup

This guide covers the essential security steps to perform right after getting access to your EC2 instance running Ubuntu 24.04 LTS.

## 1. Update System Packages

First, update all system packages to their latest versions:

```bash
sudo apt update
sudo apt upgrade -y
```

## 2. Configure SSH Security

Edit the SSH configuration file:
```bash
sudo systemctl status ssh # Verify SSH service name
sudo nano /etc/ssh/sshd_config
```

Make these security-focused changes:
```
# Disable password authentication
PasswordAuthentication no

# Disable root login
PermitRootLogin no

# Use SSH Protocol 2
Protocol 2

# Specify which users can connect (replace 'ubuntu' with your username)
AllowUsers ubuntu
```

Restart SSH service:
```bash
sudo systemctl restart ssh
```

## 3. Install and Configure Fail2ban

Install Fail2ban to protect against brute force attacks:
```bash
sudo apt install -y fail2ban
```

Create a custom configuration:
```bash
# Create jail.local file
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local

# Create a custom config file for better upgradability
sudo mkdir -p /etc/fail2ban/jail.d
sudo nano /etc/fail2ban/jail.d/custom.conf
```

Add these settings to custom.conf:
```ini
[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
findtime = 300
bantime = 3600
```

Start and enable Fail2ban:
```bash
sudo systemctl start fail2ban
sudo systemctl enable fail2ban
```

## 4. Additional Security Measures

### Set Up Automatic Security Updates
```bash
# Install unattended-upgrades package
sudo apt install -y unattended-upgrades

# Install update-notifier-common for configuration
sudo apt install -y update-notifier-common

# Configure unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades

# Verify the configuration
sudo nano /etc/apt/apt.conf.d/50unattended-upgrades
```

### Secure Shared Memory
First, backup fstab:
```bash
sudo cp /etc/fstab /etc/fstab.backup
```

Then add to /etc/fstab:
```
tmpfs /run/shm tmpfs defaults,noexec,nosuid 0 0
```

## Verification Steps

1. Check SSH configuration:
   ```bash
   sudo sshd -t  # Test configuration syntax
   sudo systemctl status ssh  # Check service status
   ```

2. Check Fail2ban status:
   ```bash
   sudo systemctl status fail2ban
   sudo fail2ban-client status
   sudo fail2ban-client status sshd
   ```

3. Verify automatic updates:
   ```bash
   sudo systemctl status unattended-upgrades
   ```

## Security Best Practices for EC2

1. **EC2 Security Groups**
   - Keep your security group rules minimal and specific
   - Only open necessary ports (e.g., 22 for SSH, 80/443 for web traffic)
   - Use specific IP ranges where possible instead of 0.0.0.0/0
   - Regularly audit your security group rules

2. **IAM Best Practices**
   - Use IAM roles for EC2 instances instead of access keys
   - Follow the principle of least privilege
   - Regularly rotate credentials
   - Use MFA for IAM users

3. **Network Security**
   - Place instances in private subnets when possible
   - Use VPC flow logs for network monitoring
   - Consider using AWS WAF for web applications
   - Implement proper network segmentation

## Next Steps

After completing these security measures, proceed to [Node.js Installation](02-nodejs-setup.md) to set up your development environment. 