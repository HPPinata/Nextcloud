# Nextcloud
Simple setup guide to Nextcloud in Docker-Compose

## 0. Preparation
System: Should work with any arm64 or x86_64 computer  
OS: OpenSUSE (others possible, just use the appropriate package manager)

Domains: 1 Zone with Wildcard CNAME (completely free) from [dynv6.com](https://dynv6.com)

On your router: Forward ports ``80/tcp``, ``443/tcp``, ``3478/tcp``, ``3478/udp``

Optional: You can use external storage (format it, then adapt the ``{echo} >> /etc/fstab`` line
with your ``%DRIVE%`` and ``%FS%`` and make sure it's no longer commented out)

Make sure to replace all occurences of these Variables:
```
%TIME%     your Timezone (eg. Europe/Berlin)
%KBOARD%   your Keyboard layout (eg. de)
%HNAME%    the hostname on your system
%DOMAIN%   your dynv6 zone
%ESPASS%   elasticsearch password
%TURNPASS% a secure password for coturn
%MAIL%     email adress for letsencrypt notifications
%TOKEN%    dynv6 update token (see zone-info)
%RP%       mariadb root password
%UP%       mariadb user password
%NA%       mariadb database name
%US%       mariadb user
```

## 1. Host-Setup
```
zypper in -y cron docker docker-compose firewalld

timedatectl set-timezone %TIME%
localectl set-keymap %KBOARD%
echo '%HNAME%' > /etc/hostname

firewall-cmd --permanent --new-service turnserver
firewall-cmd --permanent --service turnserver --add-port 3478/tcp --add-port 3478/udp
firewall-cmd --permanent --add-service http --add-service https --add-service turnserver
firewall-cmd --reload

mkdir -p /var/nextcloud/mount
#{ echo; echo '%DRIVE%  /var/nextcloud/mount  %FS%  defaults  0  0'; } >> /etc/fstab
#mount -a
```

## 2. Directories
```
cd /var/nextcloud
mkdir -p build/dynv6
mkdir -p build/nextcloud
```

## 3. config files
### 3.1 coturn
```
cat <<'EOL' > ./turnserver.conf
listening-port=3478
fingerprint
use-auth-secret
static-auth-secret=%TURNPASS%
realm=turn.%DOMAIN%
total-quota=0
bps-capacity=0
stale-nonce
no-multicast-peers
EOL
cat ./turnserver.conf
```

### 3.2 dynv6
```
cat <<'EOL' > build/dynv6/dyndns.bash
#!/bin/bash

ipv4=$(curl -4 api4.ipify.org)
ipv6=$(curl -6 api6.ipify.org)

for Z in ${ZONE[@]}; do
  dns4=$(dig -t A +short $Z)
  if [ "$ipv4" != "$dns4" ]; then
    curl -4 -L "https://ipv4.dynv6.com/api/update?zone=$Z&ipv4=auto&token=$TK"
  fi
done

for Z in ${ZONE[@]}; do
  dns6=$(dig -t AAAA +short $Z)
  if [ "$ipv6" != "$dns6" ]; then
    curl -6 -L "https://ipv6.dynv6.com/api/update?zone=$Z&ipv6=auto&ipv6prefix=auto&token=$TK"
  fi
done

sleep 300
EOL
cat build/dynv6/dyndns.bash
```
```
cat <<'EOL' > build/dynv6/Dockerfile
FROM alpine:latest

RUN apk add --no-cache bash curl bind-tools

ADD ./dyndns.bash /
RUN chmod +x ./dyndns.bash

CMD ["./dyndns.bash"]
EOL
cat build/dynv6/Dockerfile
```

### 3.3 caddy
```
set +H
phash=$(docker run --rm caddy:alpine caddy hash-password --plaintext %ESPASS%)
```
```
cat <<EOL > ./Caddyfile
{
  email    %MAIL%
  #acme_ca  https://acme-staging-v02.api.letsencrypt.org/directory
  #local_certs
}

https://%HNAME%, https://cloud.%DOMAIN% {
  redir /.well-known/carddav /remote.php/dav 301
  redir /.well-known/caldav /remote.php/dav 301
  reverse_proxy nextcloud-c:80
}

https://elastic.%DOMAIN% {
  basicauth * {
    elastic $phash
  }
  reverse_proxy elasticsearch:9200
}

https://office.%DOMAIN% {
  reverse_proxy collabora:9980
}
EOL
cat ./Caddyfile
```

### 3.4 nextcloud
```
cat <<'EOL' > build/nextcloud/php.ini
memory_limit=4G
upload_max_filesize=1000G
post_max_size=1000G
EOL
cat build/nextcloud/php.ini
```
```
cat <<'EOL' > build/nextcloud/Dockerfile
FROM nextcloud:apache

RUN export DEBIAN_FRONTEND=noninteractive; \
    apt update && apt full-upgrade -y; \
    apt install -y ffmpeg imagemagick tesseract-ocr tesseract-ocr-deu tesseract-ocr-eng

RUN sed -i 's+rights="none" pattern="PDF"+rights="read|write" pattern="PDF"+g' /etc/ImageMagick-6/policy.xml; \
    cat /etc/ImageMagick-6/policy.xml

ADD ./php.ini /usr/local/etc/php/conf.d/
EOL
cat build/nextcloud/Dockerfile
```

## 4. compose.yml
```
cat <<'EOL' > compose.yml
services:
  dynv6:
    build: ./build/dynv6
    container_name: dynv6
    network_mode: host
    environment:
      - ZONE=( %DOMAIN% )
      - TK=%TOKEN%
    restart: unless-stopped

  coturn:
    image: coturn/coturn:alpine
    container_name: coturn
    network_mode: host
    volumes:
      - ./turnserver.conf:/etc/coturn/turnserver.conf:ro
    restart: unless-stopped

  caddy:
    image: caddy:alpine
    container_name: caddy
    depends_on:
      - dynv6
    ports:
      - 80:80
      - 443:443
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
      - caddy_config:/config
    restart: unless-stopped

  mariadb:
    image: mariadb:latest
    container_name: mariadb
    command: --transaction-isolation=READ-COMMITTED --log-bin=binlog --binlog-format=ROW
    volumes:
      - db:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=%RP%
      - MYSQL_PASSWORD=%UP%
      - MYSQL_DATABASE=%NA%
      - MYSQL_USER=%US%
      - MARIADB_AUTO_UPGRADE=true
      - MARIADB_DISABLE_UPGRADE_BACKUP=true
    restart: unless-stopped

  redis:
    image: redis:alpine
    container_name: redis
    restart: unless-stopped

  nextcloud-c:
    build: ./build/nextcloud
    container_name: nextcloud-c
    depends_on:
      - coturn
      - caddy
      - redis
      - mariadb
      - elasticsearch
      - collabora
    volumes:
      - nextcloud:/var/www/html
      - ./config:/var/www/html/config
      - ./mount/data:/var/www/html/data
    environment:
      - TZ=%TIME%
      - MYSQL_PASSWORD=%UP%
      - MYSQL_DATABASE=%NA%
      - MYSQL_USER=%US%
      - MYSQL_HOST=mariadb
      - REDIS_HOST=redis
    restart: unless-stopped

  elasticsearch:
    image: bitnami/elasticsearch:8
    container_name: elasticsearch
    volumes:
      - elasticsearch:/bitnami/elasticsearch/data
    environment:
      - TZ=%TIME%
    restart: unless-stopped

  collabora:
    image: collabora/code:latest
    container_name: collabora
    depends_on:
      - caddy
    cap_add:
      - MKNOD
    environment:
      - aliasgroup1=https://cloud.%DOMAIN%:443
      - aliasgroup2=https://%HNAME%:443
      - extra_params=--o:ssl.enable=false --o:ssl.termination=true
    restart: unless-stopped

volumes:
  caddy_data:
  caddy_config:
  db:
  elasticsearch:
  nextcloud:
EOL
cat compose.yml
```
```
docker compose pull
docker compose build --pull
```
```
docker compose up -dV
```

## 5. Apps and Settings
### Apps:
  - Hub: Nextcloud Office and Talk
  - Multimedia: Preview Generator
  - Search: Full text search & –Elasticsearch Platform, –Files, –Files–Tesseract OCR

### Settings:
#### Collabora:
  - Custom server: https://office.%DOMAIN%

#### Talk:
  - Stun & Turn = turn.%DOMAIN%:3478
  - Secret = %TURNPASS%

#### Fulltext Search:
  - Adresse = https://elastic:%ESPASS%@elastic.%DOMAIN%
  - Index = cloud
  - Tokenizer = standard
  - Languages = eng,de

## 6. Config
### 6.1 PHP parameter info
```
cd /var/nextcloud
```
```
docker exec -u www-data nextcloud-c php occ config:system:set trusted_domains 0 --value %HNAME%
docker exec -u www-data nextcloud-c php occ config:system:set trusted_domains 1 --value cloud.%DOMAIN%
docker exec -u www-data nextcloud-c php -i
```

### 6.2 config.php
```
VAR=$(cat <<'EOL'
  'enable_previews' => true,
  'enabledPreviewProviders' =>
  array (
      'OC\Preview\BMP',
      'OC\Preview\GIF',
      'OC\Preview\JPEG',
      'OC\Preview\Krita',
      'OC\Preview\MarkDown',
      'OC\Preview\MP3',
      'OC\Preview\OpenDocument',
      'OC\Preview\PNG',
      'OC\Preview\TXT',
      'OC\Preview\XBitmap',
      'OC\Preview\Font',
      'OC\Preview\HEIC',
      'OC\Preview\Illustrator',
      'OC\Preview\Movie',
      'OC\Preview\MSOffice2003',
      'OC\Preview\MSOffice2007',
      'OC\Preview\MSOfficeDoc',
      'OC\Preview\PDF',
      'OC\Preview\Photoshop',
      'OC\Preview\Postscript',
      'OC\Preview\StarOffice',
      'OC\Preview\SVG',
      'OC\Preview\TIFF',
      'OC\Preview\EMF',
  ),
  'datadirectory'
EOL
)
VAR=$(VAR=${VAR@Q}; echo "${VAR:2:-1}")
sed -i "s+  'datadirectory'+$VAR+g" config/config.php
```
```
VAR=$(cat <<'EOL'
  'overwriteprotocol' => 'https',
  'default_phone_region' => 'DE',
  'dbname'
EOL
)
VAR=$(VAR=${VAR@Q}; echo "${VAR:2:-1}")
sed -i "s+  'dbname'+$VAR+g" config/config.php
```
```
cat config/config.php
```

## 7. AutoUpdate
```
cat <<'EOL' > update.bash
#!/bin/bash

cd /var/nextcloud
docker exec -u www-data nextcloud-c php occ maintenance:mode --on
sleep 5
docker compose down
docker compose pull
docker compose build --pull
docker compose up -dV
sleep 10
docker exec -u www-data nextcloud-c php occ maintenance:mode --off
sleep 5
docker exec -u www-data nextcloud-c php occ upgrade -n
docker system prune -a -f --volumes
zypper dup -y
reboot
EOL
cat update.bash
chmod +x update.bash
```

## 8. Root Cron
```
cat <<'EOL' | crontab -

*/5 * * * * docker exec -u www-data nextcloud-c php -f /var/www/html/cron.php
*/5 * * * * docker exec -u www-data nextcloud-c php occ files:scan --all
*/5 * * * * docker exec -u www-data nextcloud-c php occ fulltextsearch:index
0 6 * * * docker exec -u www-data nextcloud-c php occ preview:generate-all
3 3 * * 6 /var/nextcloud/update.bash
EOL
crontab -l
```
```
reboot
```
