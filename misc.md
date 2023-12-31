# linux general

## disable ipv6

```
echo 'net.ipv6.conf.all.disable_ipv6=1' | sudo tee -a /etc/sysctl.d/99-disable-ipv6.conf
echo 'net.ipv6.conf.default.disable_ipv6=1' | sudo tee -a /etc/sysctl.d/99-disable-ipv6.conf
echo 'net.ipv6.conf.lo.disable_ipv6=1' | sudo tee -a /etc/sysctl.d/99-disable-ipv6.conf
sudo sysctl -p /etc/sysctl.d/99-disable-ipv6.conf

sudo sed -i 's|GRUB_CMDLINE_LINUX=""|GRUB_CMDLINE_LINUX="ipv6.disable=1"|' /etc/default/grub
sudo update-grub
```

## install ufw

```
sudo apt -y install ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
yes | sudo ufw enable
```

sed -i '/<pattern>/s/^/#/g' file
sed -i '/<pattern>/s/^#//g' file

# install dockge

```
SERVICE=dockge
mkdir -p /opt/stacks /opt/$SERVICE
curl https://raw.githubusercontent.com/louislam/dockge/master/compose.yaml --output /opt/$SERVICE/compose.yaml
sed -i 's|- 5001|- 10.10.1.11:5001|' /opt/dockge/compose.yaml
docker compose --project-directory /opt/$SERVICE/ up -d
```

# install Cloudflare-Tunnel

```
SERVICE=cloudflared
mkdir -p /opt/stacks/$SERVICE

cat << 'EOF' > /opt/stacks/$SERVICE/compose.yaml
version: "3.3"
services:
  cloudflared:
    container_name: cloudflared
    restart: always
    network_mode: host
    labels:
      - com.centurylinklabs.watchtower.enable=true
    image: cloudflare/cloudflared:latest
    command: tunnel --no-autoupdate run --token
      eyJhIjoiMzdlMGI2MmNlOTU1MGFkMmM2Yjg2ODdjZTU0ZTE2MmMiLCJ0IjoiZGIwZWJkYmEtMTYwMi00NTMyLWE2ZjUtZDkwY2U0OGJlMTUwIiwicyI6Ik9USmtOV0k0Wm1FdE9HWmlOQzAwWlRFMkxXSm1NVEl0TVdKaVpHUmtPVE0zTTJaaCJ9
EOF

docker compose --project-directory /opt/stacks/$SERVICE/ up -d
unset SERVICE
```

# install watchtower

```
SERVICE=watchtower
mkdir -p /opt/stacks/$SERVICE

cat << 'EOF' > /opt/stacks/$SERVICE/compose.yaml
version: "3.3"
services:
  watchtower:
    restart: unless-stopped
    container_name: watchtower
    ports:
      - 10.10.1.12:8080:8080
    environment:
      - TZ=Africe/Cairo
      - WATCHTOWER_CLEANUP=true
      - WATCHTOWER_SCHEDULE=0 0 5 * * *
      - WATCHTOWER_NOTIFICATIONS=email
      - WATCHTOWER_NOTIFICATION_EMAIL_FROM=divepoint.logging@gmail.com
      - WATCHTOWER_NOTIFICATION_EMAIL_TO=divepoint@tn87.de
      - WATCHTOWER_NOTIFICATION_EMAIL_SERVER=smtp.gmail.com
      - WATCHTOWER_NOTIFICATION_EMAIL_SERVER_PORT=587
      - WATCHTOWER_NOTIFICATION_EMAIL_SERVER_USER=divepoint.logging@gmail.com
      - WATCHTOWER_NOTIFICATION_EMAIL_SERVER_PASSWORD=krnbadjfepyktcee
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    image: containrrr/watchtower
    command: --label-enable #--monitor-only
EOF

docker compose --project-directory /opt/stacks/$SERVICE/ up -d
unset SERVICE
```

# install guacamole

```
SERVICE=guacamole
mkdir -p /opt/stacks/$SERVICE

cat << 'EOF' > /opt/stacks/$SERVICE/compose.yaml
version: "3.3"
services:
  guacamole:
    container_name: guacamole
    restart: always
    ports:
      - 10.10.1.13:8080:8080
    volumes:
      - ./config:/config
    environment:
      - EXTENSIONS=auth-totp
      - TZ=Africa/Cairo
    image: maxwaldorf/guacamole:1.5.0
EOF

docker compose --project-directory /opt/stacks/$SERVICE/ up -d
unset SERVICE


## create self-signed certificate
#docker exec npm openssl req -x509 -newkey rsa:4096 -keyout /data/custom_ssl/dp.lcl.key.pem -out /data/custom_ssl/dp.lcl.cert.pem -sha256 -days 3650 -nodes -subj "/C=XX/ST=StateName/L=CityName/O=CompanyName/OU=CompanySectionName/CN=dp.lcl" -addext "subjectAltName=DNS:dp.lcl,DNS:*.dp.lcl"
#docker exec npm cat /data/custom_ssl/dp.lcl.cert.pem


# install Nextcloud-AIO

SERVICE=nextcloud_aio
mkdir -p /opt/stacks/$SERVICE

cat << 'EOF' > /opt/stacks/$SERVICE/compose.yaml
version: "3.3"
services:
  all-in-one:
    init: true
    container_name: nextcloud-aio-mastercontainer
    restart: always
    ports:
      - 10.10.1.15:8080:8080
    environment:
      - APACHE_PORT=11000
      - APACHE_IP_BINDING=0.0.0.0
      - SKIP_DOMAIN_VALIDATION=true
      - NEXTCLOUD_MOUNT=/hdd/
    volumes:
      - mastercontainer:/mnt/docker-aio-config
      - /var/run/docker.sock:/var/run/docker.sock:ro
    image: nextcloud/all-in-one:latest
    networks:
      - nextcloud-aio
volumes:
  mastercontainer:
networks:
  nextcloud-aio:
    name: nextcloud-aio
    driver: bridge
EOF

docker compose --project-directory /opt/stacks/$SERVICE/ up -d
docker network connect nextcloud-aio npm
echo go to aio.dp.lcl and setup nextcloud
# wait for nextcloud to be started and set phone region
nextcloud-phone-region=EG
until /usr/bin/docker exec --user www-data nextcloud-aio-nextcloud php occ config:system:set default_phone_region --value='$nextcloud-phone-region' &>/dev/null; do
  sleep 1;
done;
# get nextcloud-aio password
curl -Lk https://aio.dp.lcl:8080 # needed to generate aio-password
until docker exec -it nextcloud-aio-mastercontainer cat /mnt/docker-aio-config/data/configuration.json | grep password | cut -d'"' -f 4; do
  sleep 1;
done;
# enable external files app
docker exec --user www-data nextcloud-aio-nextcloud php occ app:enable files_external
# allow self signed certificates for office
/usr/bin/docker exec --user www-data nextcloud-aio-nextcloud php occ config:app:set richdocuments disable_certificate_verification --value='yes'
# create npm proxy host for nextcloud
nextcloud-domain=cld.dp.lcl
docker exec npm curl --location --request POST 'http:/localhost:81/api/nginx/proxy-hosts' \
  --header "Authorization: Bearer $npm_token" \
  --header "Content-Type: application/json" \
  --data '{
    "domain_names": [
        "$nextcloud-domain"
    ],
    "forward_host": "nextcloud-aio-apache",
    "forward_port": 11000,
    "access_list_id": 0,
    "certificate_id": 0,
    "ssl_forced": 0,
    "caching_enabled": 0,
    "block_exploits": 1,
    "advanced_config": "listen 443 ssl http2;\r\ninclude conf.d/include/ssl-ciphers.conf;\r\ninclude conf.d/include/force-ssl.conf;\r\nssl_certificate /data/custom_ssl/dp.lcl.cert.pem;\r\nssl_certificate_key /data/custom_ssl/dp.lcl.key.pem;\r\n\r\nclient_body_buffer_size 512k;\r\nproxy_read_timeout 86400s;\r\nclient_max_body_size 0;",
    "meta": {
        "letsencrypt_agree": false,
        "dns_challenge": false,
        "nginx_online": true,
        "nginx_err": null
    },
    "allow_websocket_upgrade": 1,
    "http2_support": 0,
    "forward_scheme": "http",
    "enabled": 1,
    "locations": [],
    "hsts_enabled": 0,
    "hsts_subdomains": 0
}'

unset SERVICE


# install samba

SERVICE=samba
mkdir -p /opt/stacks/$SERVICE

cat << 'EOF' > /opt/stacks/$SERVICE/compose.yaml
version: "3.3"
services:
  samba:
    restart: unless-stopped
    container_name: samba
    ports:
      - 10.10.1.19:137-138:137-138/udp
      - 10.10.1.19:139:139
      - 10.10.1.19:445:445
    volumes:
      - ./share:/share
    image: dperson/samba
    command: -p -u "tino;asdasd" -s "public;/share/public;yes;no;yes" -s
      "tino;/share/tino;no;no;no;tino"
EOF

mkdir /opt/stacks/$SERVICE/share
docker compose --project-directory /opt/stacks/$SERVICE/ up -d
unset SERVICE





docker run \
--init \
--sig-proxy=false \
--name nextcloud-aio-mastercontainer \
--restart always \
--publish 10.10.1.15:8080:8080 \
--env APACHE_PORT=11000 \
--env APACHE_IP_BINDING=0.0.0.0 \
--env SKIP_DOMAIN_VALIDATION=true \
--volume nextcloud_aio_mastercontainer:/mnt/docker-aio-config \
--volume /var/run/docker.sock:/var/run/docker.sock:ro \
nextcloud/all-in-one:latest


docker network create npm
docker run -d \
  --name npm \
  --restart=always \
  --network=npm \
  --publish 80:80 \
  --publish 81:81 \
  --publish 443:443 \
  --volume /etc/localtime:/etc/localtime:ro \
  --volume npm_data:/data \
  --volume npm_letsencrypt:/etc/letsencrypt \
  --env DISABLE_IPV6=true \
  jc21/nginx-proxy-manager:latest



```
