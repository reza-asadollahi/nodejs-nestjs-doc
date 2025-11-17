Ubuntu server deployment for a NestJS app with Postgres and Redis
You’ve built the app—now let’s ship it. Below is a clean, end-to-end path to connect to your Ubuntu server, provision Postgres and Redis, configure environment variables, and deploy your NestJS app with PM2 and Nginx. It’s production-minded and easy to iterate on.

Server access and baseline setup
Connect via SSH
bash
ssh username@your_server_ip
# or with a key
ssh -i ~/.ssh/your_key.pem username@your_server_ip
Update system and install essentials
bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y build-essential git ufw curl
Configure firewall (UFW)
bash
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status
Install Node.js, PM2, and Nginx
Node.js (LTS) via NodeSource
bash
curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash -
sudo apt install -y nodejs
node -v
npm -v
PM2 for process management
bash
sudo npm i -g pm2
pm2 -v
Nginx for reverse proxy
bash
sudo apt install -y nginx
sudo systemctl enable nginx
sudo systemctl start nginx
Install and configure Postgres
Postgres server
bash
sudo apt install -y postgresql postgresql-contrib
sudo systemctl enable postgresql
sudo systemctl start postgresql
Create database, user, and permissions
bash
sudo -u postgres psql
sql
CREATE USER bs_user WITH PASSWORD 'your_strong_password';
CREATE DATABASE bs_database OWNER bs_user;
GRANT ALL PRIVILEGES ON DATABASE bs_database TO bs_user;
\q
Optional: allow local connections with password
Edit:

bash
sudo nano /etc/postgresql/*/main/pg_hba.conf
Change local auth to:

text
# Database administrative login by Unix domain socket
local   all             postgres                                peer
# App user
local   all             bs_user                                  md5
Then reload:

bash
sudo systemctl reload postgresql
Install and configure Redis
bash
sudo apt install -y redis-server
sudo nano /etc/redis/redis.conf
Set for production:

supervised systemd

requirepass your_strong_redis_password (optional but recommended)

bind 127.0.0.1

bash
sudo systemctl enable redis-server
sudo systemctl restart redis-server
redis-cli ping
If you set a password:

bash
redis-cli -a your_strong_redis_password ping
Fetch your app and set environment
Clone your repository
bash
cd /var/www
sudo mkdir your-app && sudo chown $USER:$USER your-app
cd your-app
git clone https://github.com/yourname/your-repo.git .
Install dependencies and build
bash
npm ci        # or npm install --frozen-lockfile if using yarn
npm run build
Environment variables (.env)
Create:

bash
nano .env
Example:

env
# App
PORT=3000
NODE_ENV=production

# Postgres
DB_HOST=localhost
DB_PORT=5432
DB_USERNAME=bs_user
DB_PASSWORD=your_strong_password
DB_DATABASE=bs_database
DB_SYNC=false

# Redis
REDIS_HOST=127.0.0.1
REDIS_PORT=6379
REDIS_PASSWORD=your_strong_redis_password
REDIS_TTL=300

# JWT, etc.
JWT_SECRET=change_me
Optional: run migrations
If you use TypeORM migrations:

bash
npm run typeorm:migration:run
# or your custom script: npm run migration:run
Start the app with PM2
Start and persist
bash
pm2 start dist/main.js --name your-app
pm2 status
pm2 logs your-app
pm2 save
pm2 startup systemd
# Follow the printed command to enable PM2 at boot
PM2 ecosystem file (recommended)
bash
nano ecosystem.config.js
js
module.exports = {
  apps: [
    {
      name: 'your-app',
      script: 'dist/main.js',
      env: {
        NODE_ENV: 'production',
        PORT: 3000,
      },
    },
  ],
};
Then:

bash
pm2 start ecosystem.config.js
pm2 save
Configure Nginx reverse proxy
Create site config
bash
sudo nano /etc/nginx/sites-available/your-app
nginx
server {
  listen 80;
  server_name your-domain.com;

  location / {
    proxy_pass http://127.0.0.1:3000;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';
    proxy_set_header Host $host;
    proxy_cache_bypass $http_upgrade;
  }

  # Optional: trust X-Forwarded headers
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  proxy_set_header X-Forwarded-Proto $scheme;
}
Enable and test
bash
sudo ln -s /etc/nginx/sites-available/your-app /etc/nginx/sites-enabled/your-app
sudo nginx -t
sudo systemctl reload nginx
Optional: SSL with Let’s Encrypt
bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d your-domain.com
sudo systemctl reload nginx
Health checks, logs, and maintenance
Check app port:

bash
ss -lntp | grep 3000
PM2 logs:

bash
pm2 logs your-app --lines 100
Nginx logs:

bash
sudo tail -f /var/log/nginx/access.log /var/log/nginx/error.log
Postgres status:

bash
sudo systemctl status postgresql
Redis status:

bash
sudo systemctl status redis-server
Security and production hardening
Users and permissions:

Run the app under a non-root user.

Use /var/www/your-app with restricted permissions.

Environment secrets:

Store .env with chmod 600.

Consider systemd environment files or PM2’s env if you avoid .env.

Database access:

Bind Postgres to localhost if the app is on the same server.

If remote DB, use SSL and firewall allowlist.

Redis:

Bind to 127.0.0.1, set requirepass, disable external access.

Auto-deploy:

Set up a deploy script or GitHub Actions that SSH into the server, pull, build, and reload PM2.

Zero-downtime deploy workflow
Pull latest:

bash
cd /var/www/your-app
git pull
Install and build:

bash
npm ci
npm run build
Run migrations (if any):

bash
npm run migration:run
Reload app:

bash
pm2 reload your-app
pm2 save
Troubleshooting quick hits
App not reachable via domain:

Check Nginx config and DNS A record.

curl -I http://127.0.0.1:3000 to confirm backend responds locally.

500 errors through Nginx:

Check app logs (pm2 logs), ensure the app listens on PORT and is healthy.

Database auth errors:

Verify .env credentials and Postgres pg_hba.conf.

psql -h localhost -U bs_user -d bs_database.

Redis auth errors:

Ensure REDIS_PASSWORD matches redis.conf.

redis-cli -a your_strong_redis_password ping.

Optional improvements
ConfigModule profiles: load .env.production vs .env.development.

Joi validation: enforce required envs at boot.

System metrics: install netdata or node_exporter + Prometheus.

Backups: pg_dump nightly and keep off-box.

bash
pg_dump -U bs_user -F c bs_database > /var/backups/bs_database_$(date +%F).dump
If you share your repo structure and current scripts, I can tailor the PM2 and Nginx config to match your exact setup, including SSL and multiple environments.
