# nginx proxy manager

#### sources

https://documenter.getpostman.com/view/4475423/2s93mBvyLZ#d0e63f29-3617-4717-b27d-23a8da953ec7
https://github.com/NginxProxyManager/nginx-proxy-manager/tree/develop/backend/schema/endpoints

## setup docker via compose

```
# install nginx proxy manager

SERVICE=npm
mkdir -p /opt/stacks/$SERVICE

cat << 'EOF' > /opt/stacks/$SERVICE/compose.yaml
version: "3.3"
services:
  nginx-proxy-manager:
    container_name: npm
    restart: always
    networks:
      - npm
    ports:
      - 10.10.200.1:80:80
      - 10.10.200.1:443:443
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
    environment:
      - DISABLE_IPV6=true
    image: jc21/nginx-proxy-manager:latest

networks:
  npm:
    name: npm
EOF

mkdir -p /opt/stacks/$SERVICE/{npm_data,npm_letsencrypt}
docker compose --project-directory /opt/stacks/$SERVICE/ up -d


#### get api token

npm_token=`docker exec npm curl --location --request POST 'http://localhost:81/api/tokens'  \
  --header "Content-Type: application/json" \
  --data '{
    "identity":"admin@example.com",
    "secret":"changeme"
}' | cut -d'"' -f 4`


#### get an certificate

domain=d1ve.xyz
le_mail='admin@srvadm.de'
cf_token='xxxxx'
docker exec npm curl --location --request POST 'http:/localhost:81/api/nginx/certificates' \
  --header "Authorization: Bearer $npm_token" \
  --header "Content-Type: application/json" \
  --data '{
    "domain_names": [
      "'$domain'",
      "*.'$domain'"
    ],
    "meta": {
      "letsencrypt_email":"'$le_mail'",
      "dns_challenge":true,
      "dns_provider":"cloudflare",
      "dns_provider_credentials":"dns_cloudflare_api_token = '$cf_token'",
      "letsencrypt_agree":true
    },
    "provider":"letsencrypt"
  }'


## create npm host

subdomain=npm
host='localhost'
port=81
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
    "hsts_enabled": 0,
    "hsts_subdomains": 0
}'
```

# get api token

```
npm_token=`docker exec npm curl --location --request POST 'http://localhost:81/api/tokens'  \
  --header "Content-Type: application/json" \
  --data '{
    "identity":"admin@example.com",
    "secret":"changeme"
}' | cut -d'"' -f 4`
```

# edit admin details

```
docker exec npm curl --location --request PUT 'http://localhost:81/api/users/1' \
  --header "Authorization: Bearer $npm_token" \
  --header 'accept: application/json' \
  --data-raw '{
    "name": "new-Admin",
    "nickname": "newadmin",
    "email": "newadmin@example.com"
}'
```

# list certificates

```
docker exec npm curl --location --request GET 'http://localhost:81/api/nginx/certificates' \
  --header "Authorization: Bearer $npm_token" \
  --header 'accept: application/json'
```

# change admin password

```
docker exec npm curl --location --request PUT 'http://localhost:81/api/users/1/auth' \
  --header "Authorization: Bearer $npm_token" \
  --header "Content-Type: application/json" \
  --data-raw '{
    "type":"password",
    "current":"changeme",
    "secret":"newpassword"
}'
```

# get proxy hosts

```
docker exec npm curl --location --request GET 'http://localhost:81/api/nginx/proxy-hosts' \
  --header "Authorization: Bearer $npm_token" \
  --header 'accept: application/json'
```

# delete proxy host

```
docker exec npm curl --location --request DELETE 'http://localhost:81/api/nginx/proxy-hosts/{proxy_host_id}' \
  --header "Authorization: Bearer $npm_token" \
  --header 'accept: application/json'
```

## get admin user information

```
docker exec npm curl --location --request GET 'http://localhost:81/api/users/1' \
  --header "Authorization: Bearer $npm_token" \
  --header 'accept: application/json'
unset SERVICE
```

## how to create a self signed certificate

```
docker exec npm openssl req -x509 -newkey rsa:4096 -keyout /data/custom_ssl/key.pem -out /data/custom_ssl/cert.pem -sha256 -days 3650 -nodes -subj "/C=XX/ST=StateName/L=CityName/O=CompanyName/OU=CompanySectionName/CN=dp.lcl" -addext "subjectAltName=DNS:domain.local,DNS:*.domain.local"
docker exec npm cat /data/custom_ssl/cert.pem
```
