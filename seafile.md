# Seafile

#### sources

https://manual.seafile.com/docker/pro-edition/deploy_seafile_pro_with_docker/

## install seafile pro via docker compose

```
# install seafile

subdomain=cld
admin_mail=admin@d1ve.xyz
admin_pw=$(pwgen -s -1 14)
mysql_pw=$(pwgen -s -1 14)
echo "Seafile Password: $admin_pw"

SERVICE=seafile
mkdir -p /opt/stacks/$SERVICE

docker login docker.seadrive.org -u seafile -p zjkmid6rQibdZ=uJMuWS

cat << EOF > /opt/stacks/$SERVICE/compose.yaml
version: "3.3"
services:
  db:
    image: mariadb:10.11
    container_name: seafile-mysql
    environment:
      - MYSQL_ROOT_PASSWORD=$mysql_pw
      - MYSQL_LOG_CONSOLE=true
      - MARIADB_AUTO_UPGRADE=1
    volumes:
      - ./db:/var/lib/mysql
    networks:
      - seafile

  memcached:
    image: memcached:1.6.18
    container_name: seafile-memcached
    entrypoint: memcached -m 256
    networks:
      - seafile

  elasticsearch:
    image: elasticsearch:8.6.2
    container_name: seafile-elasticsearch
    environment:
      - discovery.type=single-node
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
      - "xpack.security.enabled=false"
    mem_limit: 2g
    volumes:
      - ./elasticsearch:/usr/share/elasticsearch/data
    networks:
      - seafile

  seafile:
    image: docker.seadrive.org/seafileltd/seafile-pro-mc:latest
    container_name: seafile
    volumes:
      - ./data:/shared
    environment:
      - DB_HOST=db
      - DB_ROOT_PASSWD=$mysql_pw
      - TIME_ZONE=Africa/Cairo
      - SEAFILE_ADMIN_EMAIL=$admin_mail
      - SEAFILE_ADMIN_PASSWORD=$admin_pw
      - SEAFILE_SERVER_LETSENCRYPT=false
      - SEAFILE_SERVER_HOSTNAME=$subdomain.$domain
    depends_on:
      - db
      - memcached
      - elasticsearch
    networks:
      - seafile
      - npm

networks:
  seafile:
    name: seafile
  npm:
    external: true
EOF

mkdir -p /opt/stacks/$SERVICE/{data,elasticsearch,db}
chmod g+w /opt/stacks/$SERVICE/elasticsearch

docker compose --project-directory /opt/stacks/$SERVICE/ up -d


## create seafile host

host='seafile'
port=80
forward='http'

cert_id=`docker exec npm curl --location --request GET 'http://localhost:81/api/nginx/certificates' \
  --header "Authorization: Bearer $npm_token" \
  --header 'accept: application/json' | grep $domain | cut -d'"' -f 3`
docker exec npm curl --location --request POST 'http:/localhost:81/api/nginx/proxy-hosts' \
  --header "Authorization: Bearer $npm_token" \
  --header "Content-Type: application/json" \
  --data '{
    "domain_names": [
        "'$subdomain.$domain'"
    ],
    "forward_host": "'$host'",
    "forward_port": '$port',
    "access_list_id": 0,
    "certificate_id": '${cert_id:1:-1}',
    "ssl_forced": 1,
    "caching_enabled": 0,
    "block_exploits": 1,
    "advanced_config": "",
    "meta": {
        "letsencrypt_agree": false,
        "dns_challenge": false,
        "nginx_online": true,
        "nginx_err": null
    },
    "allow_websocket_upgrade": 1,
    "http2_support": 1,
    "forward_scheme": "'$forward'",
    "enabled": 1,
    "locations": [],
    "hsts_enabled": 1,
    "hsts_subdomains": 0
}'
```
