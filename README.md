# DEV 14 [Optional]: Деплой приложений. ASP.NET

## 1. Установка пакета SDK для .NET

```bash
wget https://packages.microsoft.com/config/ubuntu/20.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
rm packages-microsoft-prod.deb
```

```bash
sudo apt-get update; \
  sudo apt-get install -y apt-transport-https && \
  sudo apt-get update && \
  sudo apt-get install -y dotnet-sdk-6.0
```

## 2. Установка MS SQL Server

```bash
wget -qO- https://packages.microsoft.com/keys/microsoft.asc | sudo apt-key add -
sudo add-apt-repository "$(wget -qO- https://packages.microsoft.com/config/ubuntu/20.04/mssql-server-2019.list)"
sudo apt-get update && sudo apt-get install -y mssql-server
```

```bash
sudo /opt/mssql/bin/mssql-conf setup
systemctl status mssql-server --no-pager
```

## 3. Получение и сборка приложения

Приложение опубликуем в папке /var/www, поэтому установим заодно nginx:

```bash
apt install -y nginx

git clone https://github.com/dotnet-architecture/eShopOnWeb
cd ~/eShopOnWeb/src/Web/
```

### 3.1 Настройка СУБД

```bash
dotnet tool install --global dotnet-ef

nano appsettings.json
```

```json
"CatalogConnection": "Server=localhost;Database=Microsoft.eShopOnWeb.CatalogDb;User Id=SA;Password=PASSWORD_FROM_SQL_SERVER",
"IdentityConnection": "Server=localhost;Database=Microsoft.eShopOnWeb.Identity;User Id=SA;Password=PASSWORD_FROM_SQL_SERVER;"
```

### 3.2 Миграции

```bash
dotnet restore
dotnet tool restore

dotnet ef database update -c catalogcontext -p ../Infrastructure/Infrastructure.csproj -s Web.csproj
dotnet ef database update -c appidentitydbcontext -p ../Infrastructure/Infrastructure.csproj -s Web.csproj
```

### 3.3 Сборка приложения

```bash
export ASPNETCORE_URLS=http://localhost:8080

dotnet publish Web.csproj -c Release -o /var/www -r linux-x64 --self-contained
chown -R www-data:www-data /var/www
```

## 4. Настройка Nginx

```bash
nano /etc/nginx/sites-available/es1305-www-1.devops.rebrain.srwx.net
```

```bash
server {
    listen        80 default_server;
    server_name   es1305-www-1.devops.rebrain.srwx.net;
    location / {
        proxy_pass         http://127.0.0.1:8080;
        proxy_http_version 1.1;
        proxy_set_header   Upgrade $http_upgrade;
        proxy_set_header   Connection keep-alive;
        proxy_set_header   Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
    }
}
```

```bash
ln -s /etc/nginx/sites-available/es1305-www-1.devops.rebrain.srwx.net /etc/nginx/sites-enabled/
rm /etc/nginx/sites-enabled/default

nginx -t
nginx -s reload
```

## 5. Настройка сервиса systemd

```bash
nano /etc/systemd/system/eshoponweb.service
```

```bash
[Unit]
Description=MS .NET Web App (eShopOnWeb)

[Service]
WorkingDirectory=/var/www
ExecStart=/usr/bin/bash -c 'cd /var/www; ./Web'
Restart=always
# Restart service after 10 seconds if the dotnet service crashes:
RestartSec=10
KillSignal=SIGINT
SyslogIdentifier=dotnet
User=www-data
Environment=ASPNETCORE_URLS=http://localhost:8080
Environment=DOTNET_PRINT_TELEMETRY_MESSAGE=false

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable eshoponweb
systemctl start eshoponweb
systemctl status eshoponweb
```

## 6. Настройка HTTPS

```bash
apt install certbot python3-certbot-nginx
certbot --nginx -d es1305-www-1.devops.rebrain.srwx.net

nano /etc/nginx/sites-available/es1305-www-1.devops.rebrain.srwx.net
```

```bash
server {
    server_name   es1305-www-1.devops.rebrain.srwx.net;
    location / {
        proxy_pass         http://127.0.0.1:8080;
        proxy_http_version 1.1;
        proxy_set_header   Upgrade $http_upgrade;
        proxy_set_header   Connection keep-alive;
        proxy_set_header   Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
    }

    listen 443 ssl http2;
    ssl_certificate /etc/letsencrypt/live/es1305-www-1.devops.rebrain.srwx.net/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/es1305-www-1.devops.rebrain.srwx.net/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

}

server {
    if ($host = es1305-www-1.devops.rebrain.srwx.net) {
        return 301 https://$host$request_uri;
    }

    listen        80 default_server;
    server_name   es1305-www-1.devops.rebrain.srwx.net;
    return 404;
}
```

```bash
nginx -t
nginx -s reload
```
