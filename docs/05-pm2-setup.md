# PM2 Process Manager Setup

This guide covers the installation and configuration of PM2 for managing Node.js applications.

## 1. Install PM2

Install PM2 globally using npm:
```bash
npm install -g pm2
```

## 2. Basic PM2 Configuration

Create a PM2 ecosystem file:
```bash
cd ~/apps
pm2 ecosystem
```

Edit the ecosystem file:
```bash
nano ecosystem.config.js
```

Add this configuration:
```javascript
module.exports = {
  apps: [{
    name: "main-app",
    script: "app.js",
    watch: false,
    instances: "max",
    exec_mode: "cluster",
    env: {
      NODE_ENV: "development",
    },
    env_production: {
      NODE_ENV: "production",
    },
    max_memory_restart: "1G",
    error_file: "logs/err.log",
    out_file: "logs/out.log",
    log_file: "logs/combined.log",
    time: true
  }]
}
```

## 3. PM2 Process Management

Start application:
```bash
# Start with ecosystem file
pm2 start ecosystem.config.js

# Or start directly
pm2 start app.js --name "main-app" --watch
```

Common PM2 commands:
```bash
# List all processes
pm2 list

# Monitor processes
pm2 monit

# Show logs
pm2 logs

# Show specific app logs
pm2 logs main-app

# Restart application
pm2 restart main-app

# Stop application
pm2 stop main-app

# Delete application from PM2
pm2 delete main-app
```

## 4. PM2 Startup Script

Generate startup script:
```bash
# Generate startup script
pm2 startup

# Save current process list
pm2 save
```

## 5. PM2 Monitoring

Enable PM2 monitoring:
```bash
# Install PM2 monitoring module
pm2 install pm2-logrotate

# Configure log rotation
pm2 set pm2-logrotate:max_size 10M
pm2 set pm2-logrotate:retain 7
pm2 set pm2-logrotate:compress true
```

## 6. Advanced PM2 Configuration

Create a more detailed ecosystem file for multiple applications:
```javascript
module.exports = {
  apps: [
    {
      name: "api-service",
      script: "./api/app.js",
      instances: "max",
      exec_mode: "cluster",
      watch: true,
      ignore_watch: ["node_modules", "logs"],
      env: {
        NODE_ENV: "development",
        PORT: 3000
      },
      env_production: {
        NODE_ENV: "production",
        PORT: 3000
      },
      max_memory_restart: "1G",
      error_file: "logs/api-err.log",
      out_file: "logs/api-out.log",
      log_file: "logs/api-combined.log",
      time: true,
      merge_logs: true,
      log_date_format: "YYYY-MM-DD HH:mm:ss Z"
    },
    {
      name: "worker-service",
      script: "./workers/worker.js",
      instances: 2,
      exec_mode: "cluster",
      watch: false,
      env: {
        NODE_ENV: "development"
      },
      env_production: {
        NODE_ENV: "production"
      },
      max_memory_restart: "500M"
    }
  ],
  deploy: {
    production: {
      user: "ubuntu",
      host: "your-server-ip",
      ref: "origin/main",
      repo: "git@github.com:username/repository.git",
      path: "/home/ubuntu/apps",
      "post-deploy": "npm install && pm2 reload ecosystem.config.js --env production"
    }
  }
}
```

## 7. PM2 Deployment Setup

Configure deployment:
```bash
# Setup deployment
pm2 deploy ecosystem.config.js production setup

# Deploy application
pm2 deploy ecosystem.config.js production
```

## 8. PM2 Monitoring Dashboard

Install PM2 web interface:
```bash
# Install PM2 web interface
pm2 install pm2-web

# Access dashboard at http://localhost:9615
```

## 9. PM2 Log Management

Configure log management:
```bash
# Create logs directory
mkdir -p ~/apps/logs

# Set up log rotation configuration
pm2 install pm2-logrotate
pm2 set pm2-logrotate:rotateInterval '0 0 * * *'
pm2 set pm2-logrotate:dateFormat 'YYYY-MM-DD_HH-mm-ss'
pm2 set pm2-logrotate:max_size '10M'
pm2 set pm2-logrotate:retain '7'
pm2 set pm2-logrotate:compress true
```

## Verification Steps

1. Check PM2 processes:
```bash
pm2 list
```

2. Monitor resource usage:
```bash
pm2 monit
```

3. Check startup script:
```bash
systemctl status pm2-${USER}
```

## Common Issues and Solutions

1. **Process Memory Issues**
   ```bash
   # Set memory limit for specific app
   pm2 start app.js --max-memory-restart 500M
   ```

2. **Process Not Starting**
   ```bash
   # Check detailed logs
   pm2 logs app-name --lines 100
   ```

3. **Startup Script Issues**
   ```bash
   # Regenerate startup script
   pm2 unstartup
   pm2 startup
   ```

## Best Practices

1. **Process Naming**
   - Use descriptive names for processes
   - Include environment in name for multiple environments

2. **Memory Management**
   - Set appropriate memory limits
   - Monitor memory usage regularly

3. **Logging**
   - Use log rotation
   - Implement proper error logging
   - Regular log cleanup

4. **Clustering**
   - Use cluster mode for CPU-intensive applications
   - Set appropriate number of instances

5. **Monitoring**
   - Regular monitoring of processes
   - Set up alerts for critical issues
   - Use PM2 monitoring dashboard

## Next Steps

Your server setup is now complete! Here's what you can do next:

1. Deploy your Node.js application
2. Set up monitoring and alerts
3. Configure backup strategies
4. Implement CI/CD pipelines 