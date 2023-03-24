# Nextcloud
Simple setup guide to Nextcloud in Docker-Compose

## 0. Preparation
System: Should work with any arm64 or x86_64 computer  
OS: OpenSUSE (others possible, just use the appropriate package manager)

Domains: 1 Zone with Wildcard CNAME (completely free) from [dynv6.com](https://dynv6.com)

You can use external storage (format it, then adapt the ``{echo} >> /etc/fstab`` line
with your %DRIVE%and %FS% and make sure it's no longer commented out)

Make sure to replace the Variables:
```
%DOMAIN%
```

## 1. Host-Setup
```
zypper in -y cron docker docker-compose

timedatectl set-timezone %TIME%

mkdir -p /var/nextcloud/mount
#{ echo; echo '%DRIVE%  /var/nextcloud/mount  %FS%  defaults  0  0'; } >> /etc/fstab
reboot
```

XXXX
## 2. Directories
```
cd /var/nextcloud
mkdir -p build/coturn
mkdir -p build/caddy
mkdir -p build/nextcloud
```

## 3. Dockerfiles
### 3.1 coturn
```
cat <<'EOL' > build/coturn/turnserver.conf
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
cat build/coturn/turnserver.conf
```
```
cat <<'EOL' > build/coturn/Dockerfile
FROM coturn/coturn:alpine

ADD ./turnserver.conf /etc/coturn/
EOL
cat build/coturn/Dockerfile
```

### 3.2 dynv6
```
cat <<'EOL' > build/dynv6/dyndns.bash
#!/bin/bash

OLD4=$(cat tempaddr4)
NEW4=$(curl api.ipify.org)
if [ "$OLD4" != "$NEW4" ]; then
  for Z in ${ZONE[@]}; do
    curl -4 -L "https://ipv4.dynv6.com/api/update?zone=$Z&ipv4=auto&token=$TK"
  done
fi

OLD6=$(cat tempaddr6)
NEW6=$(curl api64.ipify.org)
if [ "$OLD6" != "$NEW6" ]; then
  for Z in ${ZONE[@]}; do
    curl -6 -L "https://ipv6.dynv6.com/api/update?zone=$Z&ipv6=auto&ipv6prefix=auto&token=$TK"
  done
fi

echo $NEW4 > tempaddr4
echo $NEW6 > tempaddr6
sleep 300
EOL
cat build/dynv6/dyndns.bash
```
```
cat <<'EOL' > build/dynv6/Dockerfile
FROM alpine:latest

RUN apk add --no-cache curl bash

ADD ./dyndns.bash /
RUN chmod +x ./dyndns.bash

CMD ["./dyndns.bash"]
EOL
cat build/dynv6/Dockerfile
```

### 3.3 caddy
```
cat <<'EOL' > build/caddy/Caddyfile
{
  email    %MAIL%
  key_type p384
  #acme_ca  https://acme-staging-v02.api.letsencrypt.org/directory
  #local_certs
}

%HNAME%, cloud.%DOMAIN% {
  redir /.well-known/carddav /remote.php/dav 301
  redir /.well-known/caldav /remote.php/dav 301
  reverse_proxy localhost:8080
}

%HNAME%:9980, office.%DOMAIN% {
  reverse_proxy localhost:8880
}
EOL
cat build/caddy/Caddyfile
```
```
cat <<'EOL' > build/caddy/Dockerfile
FROM caddy:alpine

ADD ./Caddyfile /etc/caddy/Caddyfile
EOL
cat build/caddy/Dockerfile
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
    apt update && apt full-upgrade -y && apt install -y ffmpeg imagemagick

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
    privileged: true
    network_mode: host
    environment:
      - ZONE=( %DOMAIN% )
      - TK=%TOKEN%
    restart: unless-stopped

  coturn-container:
    build: ./build/coturn
    container_name: coturn-container
    privileged: true
    network_mode: host
    restart: unless-stopped

  caddy-container:
    build: ./build/caddy
    container_name: caddy-container
    privileged: true
    network_mode: host
    volumes:
      - caddy_data:/data
      - caddy_config:/config
    restart: unless-stopped

  mariadb-container:
    image: mariadb:latest
    container_name: mariadb-container
    privileged: true
    command: --transaction-isolation=READ-COMMITTED --log-bin=binlog --binlog-format=ROW
    networks:
      - network
    volumes:
      - db:/var/lib/mysql:rw
    environment:
      - MYSQL_ROOT_PASSWORD=%RP%
      - MYSQL_PASSWORD=%UP%
      - MYSQL_DATABASE=%NA%
      - MYSQL_USER=%US%
      - MARIADB_AUTO_UPGRADE=true
      - MARIADB_DISABLE_UPGRADE_BACKUP=true
    restart: unless-stopped

  redis-container:
    image: redis:alpine
    container_name: redis-container
    privileged: true
    networks:
      - network
    restart: unless-stopped

  nextcloud-container:
    build: ./build/nextcloud
    container_name: nextcloud-container
    privileged: true
    depends_on:
      - coturn-container
      - caddy-container
      - redis-container
      - mariadb-container
      - collabora-container
    networks:
      - network
    ports:
      - 8080:80
    volumes:
      - nextcloud:/var/www/html:rw
      - ./config:/var/www/html/config:rw
      - ./mount/data:/var/www/html/data:rw
      - /etc/localtime:/etc/localtime:ro
    environment:
      - MYSQL_PASSWORD=%UP%
      - MYSQL_DATABASE=%NA%
      - MYSQL_USER=%US%
      - MYSQL_HOST=mariadb-container
      - REDIS_HOST=redis-container
    restart: unless-stopped

  collabora-container:
    image: collabora/code:latest
    container_name: collabora-container
    privileged: true
    depends_on:
      - caddy-container
    networks:
      - network
    ports:
      - 8880:9980
    cap_add:
      - MKNOD
    environment:
      - domain=cloud.%DOMAIN%|%HNAME%
      - extra_params=--o:ssl.enable=false --o:ssl.termination=true
    restart: unless-stopped

volumes:
  caddy_data:
  caddy_config:
  db:
  nextcloud:

networks:
  network:
EOL
cat compose.yml
```
```
docker-compose pull
docker-compose build --pull
```
```
docker-compose up -dV
```

## 5. Apps and Settings
### Apps:
  - Hub: Nextcloud Office and Talk
  - Multimedia: Preview Generator

### Settings:
#### Collabora:
  - Custom server: https://office.%DOMAIN%

#### Talk:
  - Stun = turn.%DOMAIN%:3478
  - Turn = turn.%DOMAIN%
  - Secret = %TURNPASS%

## 6. Config
### 6.1 PHP parameter info
```
cd /var/nextcloud
```
```
docker exec -u www-data nextcloud-container php occ config:system:set trusted_domains 0 --value %HNAME%
docker exec -u www-data nextcloud-container php occ config:system:set trusted_domains 1 --value cloud.%DOMAIN%
docker exec -u www-data nextcloud-container php -i
```

### 6.2 config.php
```
VAR=$(cat <<'EOL'
  'enable_previews' => true,
  'enabledPreviewProviders' =>
  array (
      'OC\Preview\PNG',
      'OC\Preview\JPEG',
      'OC\Preview\GIF',
      'OC\Preview\BMP',
      'OC\Preview\XBitmap',
      'OC\Preview\MP3',
      'OC\Preview\TXT',
      'OC\Preview\MarkDown',
      'OC\Preview\OpenDocument',
      'OC\Preview\Krita',
      'OC\Preview\Illustrator',
      'OC\Preview\HEIC',
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
      'OC\Preview\Font',
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
docker exec -u www-data nextcloud-container php occ maintenance:mode --on
sleep 5
docker-compose down
docker-compose pull
docker-compose build --pull
docker-compose up -dV
sleep 10
docker exec -u www-data nextcloud-container php occ maintenance:mode --off
sleep 5
docker exec -u www-data nextcloud-container php occ upgrade -n
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
*/5 * * * * docker exec -u www-data nextcloud-container php -f /var/www/html/cron.php

*/5 * * * * /var/nextcloud/address.bash

0 6 * * * docker exec -u www-data nextcloud-container php occ preview:generate-all

3 3 * * 6 /var/nextcloud/update.bash
EOL
crontab -l
```
```
reboot
```
