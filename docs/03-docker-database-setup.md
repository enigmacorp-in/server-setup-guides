# Docker and Database Setup

This guide covers the installation of Docker and setting up MySQL and MongoDB containers.

## 1. Install Docker

First, install Docker and Docker Compose:

```bash
# Add Docker's official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Add Docker repository
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Add your user to docker group
sudo usermod -aG docker $USER

# Start and enable Docker
sudo systemctl start docker
sudo systemctl enable docker
```

## 2. Create Docker Network

Create a network for your containers:
```bash
docker network create app-network
```

## 3. MySQL Setup

### Configuration Guide
The MySQL setup involves three main configuration files:

1. **MySQL Configuration (`my.cnf`)**:
   - `character-set-server`: Sets UTF8MB4 encoding for proper Unicode support
   - `collation-server`: Defines how strings are compared and sorted
   - `default-authentication-plugin`: Uses native password authentication for better compatibility
   - `max_connections`: Maximum simultaneous connections (adjust based on your needs)

2. **Docker Compose File (`docker-compose.yml`)**:
   - Environment Variables:
     - `MYSQL_ROOT_PASSWORD`: [REQUIRED] Root superuser password
     - `MYSQL_DATABASE`: [REQUIRED] Name of your application database
     - `MYSQL_USER`: [REQUIRED] Application-specific user
     - `MYSQL_PASSWORD`: [REQUIRED] Password for application user
   - Ports:
     - "3306:3306": Maps container's MySQL port to host (change left number if host port 3306 is taken)
   - Volumes:
     - `./data`: Persistent storage for database files
     - `./conf.d`: Custom MySQL configurations

For detailed MySQL configuration options, refer to: https://dev.mysql.com/doc/refman/8.0/en/server-configuration-defaults.html

Create MySQL directory structure:
```bash
mkdir -p ~/docker/mysql/{data,conf.d}
```

Create MySQL configuration file:
```bash
cat << EOF > ~/docker/mysql/conf.d/my.cnf
[mysqld]
character-set-server=utf8mb4
collation-server=utf8mb4_unicode_ci
default-authentication-plugin=mysql_native_password
max_connections=1000
EOF
```

Create MySQL Docker Compose file:
```bash
cat << EOF > ~/docker/mysql/docker-compose.yml
version: '3.8'

services:
  mysql:
    image: mysql:8.0
    container_name: mysql
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: your_root_password
      MYSQL_DATABASE: your_database
      MYSQL_USER: your_user
      MYSQL_PASSWORD: your_password
    ports:
      - "3306:3306"
    volumes:
      - ./data:/var/lib/mysql
      - ./conf.d:/etc/mysql/conf.d
    networks:
      - app-network

networks:
  app-network:
    external: true
EOF
```

Start MySQL container:
```bash
cd ~/docker/mysql
docker compose up -d
```

## 4. MongoDB Setup

### Configuration Guide
MongoDB setup involves two main configuration files:

1. **MongoDB Configuration (`mongod.conf`)**:
   - `storage.dbPath`: Directory where MongoDB stores its data files
   - `net.bindIp`: Network interfaces MongoDB listens on (0.0.0.0 means all interfaces)
   - `security.authorization`: Enables role-based access control

2. **Docker Compose File (`docker-compose.yml`)**:
   - Environment Variables:
     - `MONGO_INITDB_ROOT_USERNAME`: [REQUIRED] MongoDB admin username
     - `MONGO_INITDB_ROOT_PASSWORD`: [REQUIRED] MongoDB admin password
   - Ports:
     - "27017:27017": Maps container's MongoDB port to host (change left number if host port 27017 is taken)
   - Volumes:
     - `./data`: Persistent storage for database files
     - `./config`: Custom MongoDB configurations

For detailed MongoDB configuration options, refer to: https://www.mongodb.com/docs/manual/reference/configuration-options/

Create MongoDB directory structure:
```bash
mkdir -p ~/docker/mongodb/{data,config}
```

Create MongoDB configuration file:
```bash
cat << EOF > ~/docker/mongodb/config/mongod.conf
storage:
  dbPath: /data/db
net:
  bindIp: 0.0.0.0
security:
  authorization: enabled
EOF
```

Create MongoDB Docker Compose file:
```bash
cat << EOF > ~/docker/mongodb/docker-compose.yml
version: '3.8'

services:
  mongodb:
    image: mongo:4.4
    container_name: mongodb
    restart: unless-stopped
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: your_root_password
    ports:
      - "27017:27017"
    volumes:
      - ./data:/data/db
      - ./config:/etc/mongo
    command: ["mongod", "--config", "/etc/mongo/mongod.conf"]
    networks:
      - app-network

networks:
  app-network:
    external: true
EOF
```

Start MongoDB container:
```bash
cd ~/docker/mongodb
docker compose up -d
```

## 5. Database Security

### MySQL Security
```bash
# Connect to MySQL container
docker exec -it mysql mysql -u root -p

# Change root password
ALTER USER 'root'@'%' IDENTIFIED BY 'new_secure_password';

# Create application user with all necessary privileges
CREATE USER 'app_user'@'%' IDENTIFIED BY 'app_password';
GRANT CREATE, SELECT, INSERT, UPDATE, DELETE, REFERENCES ON your_database.* TO 'app_user'@'%';
FLUSH PRIVILEGES;

# Verify privileges
SHOW GRANTS FOR 'app_user'@'%';
```

### MongoDB Security
```bash
# Connect to MongoDB container
docker exec -it mongodb mongosh -u admin -p

# Create application database and user
use your_database
db.createUser({
  user: "app_user",
  pwd: "app_password",
  roles: [{ role: "readWrite", db: "your_database" }]
})
```

## 6. Backup Setup

### Configuration Guide
The backup script (`backup-databases.sh`) contains several important parameters:

1. **Backup Location**:
   - `BACKUP_DIR`: Directory where backups are stored, organized by date
   - Format: `~/backups/YYYYMMDD/`

2. **MySQL Backup Parameters**:
   - Uses `mysqldump` with `--all-databases` flag to backup everything
   - Required: Update `your_root_password` with actual MySQL root password

3. **MongoDB Backup Parameters**:
   - Uses `mongodump` to create BSON format backups
   - Required: Update `your_root_password` with actual MongoDB admin password
   - Backup is stored temporarily in container at `/data/backup/`

4. **Backup Retention**:
   - Backups are compressed into `.tar.gz` format
   - Original uncompressed files are deleted after compression
   - Consider adding rotation logic to remove old backups

5. **Cron Schedule**:
   - Default: `0 0 * * *` (runs daily at midnight)
   - Modify the schedule based on your backup needs
   - Format: `minute hour day month weekday`

For more about cron scheduling, visit: https://crontab.guru

Create backup script:
```bash
mkdir -p ~/scripts
cat << EOF > ~/scripts/backup-databases.sh
#!/bin/bash

# Backup directory
BACKUP_DIR=~/backups/$(date +%Y%m%d)
mkdir -p $BACKUP_DIR

# MySQL backup
docker exec mysql mysqldump -u root -p"your_root_password" --all-databases > $BACKUP_DIR/mysql_backup.sql

# MongoDB backup
docker exec mongodb mongodump --username admin --password "your_root_password" --out /data/backup/
docker cp mongodb:/data/backup/ $BACKUP_DIR/mongodb_backup

# Compress backups
cd $BACKUP_DIR
tar czf databases_backup_$(date +%Y%m%d).tar.gz *
rm -rf mysql_backup.sql mongodb_backup
EOF

chmod +x ~/scripts/backup-databases.sh
```

Add to crontab for daily backups:
```bash
# Add to crontab -e
0 0 * * * ~/scripts/backup-databases.sh
```

## Verification Steps

1. Check Docker containers:
```bash
docker ps
docker network ls
```

2. Test MySQL connection:
```bash
docker exec -it mysql mysql -u root -p -e "SELECT VERSION();"
```

3. Test MongoDB connection:
```bash
docker exec -it mongodb mongosh -u admin -p
```

## Next Steps

After setting up the databases, proceed to [Nginx Configuration](04-nginx-setup.md) to configure the reverse proxy. 