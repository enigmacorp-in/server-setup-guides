# SSL Certificate Renewal Guide

This guide provides simple, practical steps for renewing SSL certificates when facing Nginx port conflicts.

## Quick SSL Certificate Renewal

### The Simple Method That Works

Most SSL certificate renewal issues are caused by Nginx processes blocking ports 80 and 443. Here's the straightforward solution:

#### Step 1: Stop Nginx

```bash
sudo systemctl stop nginx
```

#### Step 2: Check if All Nginx Processes Stopped

```bash
sudo lsof -i :80 -sTCP:LISTEN
sudo lsof -i :443 -sTCP:LISTEN
```

If you see any nginx processes still running, proceed to Step 3.

#### Step 3: Force Kill Any Remaining Nginx Processes

```bash
sudo pkill -f nginx
```

#### Step 4: Verify Ports Are Free

```bash
sudo lsof -i :80 -sTCP:LISTEN
sudo lsof -i :443 -sTCP:LISTEN
```

You should see no output (meaning ports are free).

#### Step 5: Renew SSL Certificates

```bash
sudo certbot renew
```

#### Step 6: Start Nginx (if needed)

Nginx usually starts automatically after renewal. If not, start it manually:

```bash
sudo systemctl start nginx
```

Verify it's running:

```bash
sudo systemctl status nginx
```

## That's It!

This simple process resolves 99% of SSL renewal issues. The key is ensuring all Nginx processes are completely stopped before running Certbot.

## Troubleshooting

### If Certbot Still Fails

Try the standalone method:

```bash
sudo systemctl stop nginx
sudo pkill -f nginx
sudo certbot renew --standalone
sudo systemctl start nginx
```

### Check Certificate Status

```bash
sudo certbot certificates
```

### Test Your SSL

```bash
curl -I https://your-domain.com
```

## Automatic Renewal Setup

Add this to your crontab for automatic renewals:

```bash
sudo crontab -e
```

Add this line:

```cron
0 2 * * * /usr/bin/certbot renew --quiet --pre-hook "systemctl stop nginx; pkill -f nginx" --post-hook "systemctl start nginx"
```

This will check for renewals daily at 2 AM and handle the nginx restart automatically.

---

**Note:** Always backup your configurations before making changes. 