# Node.js Installation and Setup

This guide covers the installation of Node.js and setting up an environment for running multiple Node.js applications from BitBucket repositories.

## 1. Install Node.js Dependencies

```bash
sudo apt update
sudo apt install -y build-essential curl git
```

## 2. Install Node.js using n Package Manager

```bash
# Install n package manager
curl -L https://raw.githubusercontent.com/tj/n/master/bin/n -o n
sudo bash n lts
sudo npm install -g n

# Install Node.js 14
sudo n 14

# Verify installation
node --version
npm --version
```

## 3. Install Global Packages

```bash
# Install PM2 for process management
sudo npm install -g pm2

# Install yarn as alternative package manager
sudo npm install -g yarn
```

## 4. Set Up SSH for BitBucket

```bash
# Generate SSH key
ssh-keygen -t ed25519 -C "your.email@example.com"

# Display the public key to add to BitBucket
cat ~/.ssh/id_ed25519.pub
```

Add this SSH key to BitBucket:
1. Go to BitBucket Settings → SSH Keys
2. Click "Add key"
3. Paste your public key

Test the connection:
```bash
ssh -T git@bitbucket.org
```

## 5. Set Up Applications Directory

```bash
# Create main applications directory
mkdir -p ~/apps

# Create a directory for environment files
mkdir -p ~/apps/config

# Create a directory for logs
mkdir -p ~/apps/logs
```

## 6. Clone and Set Up Applications

```bash
# Clone your applications
cd ~/apps
git clone git@bitbucket.org:your-username/app1.git
git clone git@bitbucket.org:your-username/app2.git

# Install dependencies for each app
cd app1
npm install

cd ../app2
npm install
```

## 7. Create Base PM2 Configuration

Create a PM2 ecosystem file that will manage all your Node.js applications:
```bash
nano ~/apps/ecosystem.config.js
```

There are two ways to handle environment variables:

### Option 1: Define Environment Variables in Ecosystem File
```javascript
module.exports = {
  apps: [
    {
      name: "app1",
      script: "./app1/app.js",
      instances: 2,
      exec_mode: "cluster",
      env: {
        NODE_ENV: "production",
        PORT: 3000,
        DB_HOST: "localhost",
        DB_PORT: 3306,
        DB_USER: "app1_user",
        DB_PASSWORD: "your_password"
      },
      env_production: {
        NODE_ENV: "production",
        PORT: 3000,
        DB_HOST: "production_host"
      },
      env_development: {
        NODE_ENV: "development",
        PORT: 3001,
        DB_HOST: "localhost"
      }
    },
    {
      name: "app2",
      script: "./app2/app.js",
      instances: 2,
      exec_mode: "cluster",
      env: {
        NODE_ENV: "production",
        PORT: 3001,
        DB_HOST: "localhost",
        DB_PORT: 3306
      }
    }
  ]
}
```

### Common Syntax Errors to Avoid:
```javascript
// ❌ WRONG - Using equals sign
env: {
  DB_PORT=3306,        // Wrong: uses =
  DB_HOST="localhost"  // Wrong: uses =
}

// ✅ CORRECT - Using colon
env: {
  DB_PORT: 3306,       // Correct: uses :
  DB_HOST: "localhost" // Correct: uses :
}
```

The error you're seeing is because:
1. JavaScript objects use colons (:) to separate keys and values
2. Equals signs (=) are for assignment, not object properties
3. Each property should end with a comma
4. String values need quotes, numbers don't

To fix your error:
```bash
# Open the ecosystem file
nano ~/apps/ecosystem.config.js

# Fix the syntax by replacing = with :
# Before:
DB_PORT=3306,
# After:
DB_PORT: 3306,
```

### Option 2: Load Environment Variables from .env Files (Recommended)
1. Create separate .env files for each app:
```bash
# Create env directory
mkdir -p ~/apps/config

# Create .env files for each app
nano ~/apps/config/app1.env
nano ~/apps/config/app2.env
```

2. Add your environment variables to the .env files:
```env
# ~/apps/config/app1.env
NODE_ENV=production
PORT=3000
DB_HOST=localhost
DB_USER=app1_user
DB_PASSWORD=your_password
# ... other app1 specific variables
```

3. Update ecosystem.config.js to use .env files:
```javascript
module.exports = {
  apps: [
    {
      name: "app1",
      script: "./app1/app.js",
      instances: 2,
      exec_mode: "cluster",
      env_file: "./config/app1.env"  // Path to app1's env file
    },
    {
      name: "app2",
      script: "./app2/app.js",
      instances: 2,
      exec_mode: "cluster",
      env_file: "./config/app2.env"  // Path to app2's env file
    }
  ]
}
```

### Using Different Environments
You can start apps with specific environment configurations:
```bash
# Start with default env
pm2 start ecosystem.config.js

# Start with production env
pm2 start ecosystem.config.js --env production

# Start with development env
pm2 start ecosystem.config.js --env development
```

### Environment Variable Priority
The priority order (highest to lowest) is:
1. Command line arguments
2. Environment-specific variables (env_production, env_development)
3. Variables in .env file (if using env_file)
4. Default env object in ecosystem.config.js
5. System environment variables

## 8. BitBucket Pipeline Setup

Create a pipeline configuration file in each application repository:

```bash
cd ~/apps/app1
nano bitbucket-pipelines.yml
```

Add this pipeline configuration:
```yaml
pipelines:
  branches:
    main:
      - step:
          name: Build and Test
          image: node:14
          caches:
            - node
          script:
            - npm install
            - npm test
      - step:
          name: Deploy to Production
          deployment: production
          script:
            - pipe: atlassian/ssh-run:0.4.1
              variables:
                SSH_USER: 'ubuntu'
                SERVER: '$SERVER_IP'
                COMMAND: |
                  cd ~/apps/app1
                  git pull origin main
                  npm install --production
                  pm2 restart app1

definitions:
  caches:
    node: node_modules

```

## 9. Server Deployment Key Setup

Generate a deployment key for the server:
```bash
# Generate deployment key
ssh-keygen -t ed25519 -C "server-deployment-key" -f ~/.ssh/deployment_key

# Display the public key to add to BitBucket repository
cat ~/.ssh/deployment_key.pub
```

Add this deployment key to your BitBucket repository:
1. Go to Repository Settings → Access keys
2. Add the public key with write access

## 10. BitBucket Repository Variables

Add these repository variables in BitBucket:
1. Go to Repository Settings → Repository variables
2. Add the following variables:
   - `SERVER_IP`: Your EC2 instance IP
   - `SSH_KEY`: Content of the private key (~/.ssh/deployment_key)

## 11. Managing Applications

Start all applications:
```bash
cd ~/apps
pm2 start ecosystem.config.js
```

Common PM2 commands:
```bash
# List all running applications
pm2 list

# Monitor all applications
pm2 monit

# View logs of all applications
pm2 logs

# View logs of specific application
pm2 logs app1

# Restart specific application
pm2 restart app1

# Restart all applications
pm2 restart all
```

## 12. Auto-start on System Boot

```bash
# Generate startup script
pm2 startup

# Save current process list
pm2 save
```

## Verification Steps

1. Check Node.js installation:
```bash
node --version
npm --version
```

2. Verify running applications:
```bash
pm2 list
```

3. Check application logs:
```bash
pm2 logs
```

4. Test BitBucket pipeline:
Make a commit to the main branch and verify that the pipeline runs successfully.

## Next Steps

After setting up Node.js and BitBucket pipelines, proceed to [Docker & Database Setup](03-docker-database-setup.md) to configure your database containers. 