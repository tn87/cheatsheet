apt update
apt -y install python3 python3-dev python3-setuptools python3-pip python3-ldap libmariadb-dev-compat ldap-utils libldap2-dev dnsutils memcached libmemcached-dev poppler-utils default-jre sqlite3 nginx

#### seafile 11

pip3 install --break-system-packages --timeout=3600 django==3.2._ future==0.18._ mysqlclient==2.1._ \
 pymysql pillow==10.0._ pylibmc captcha==0.4 markupsafe==2.0.1 jinja2 sqlalchemy==2.0.18 \
 psd-tools django-pylibmc django_simple_captcha==0.5._ djangosaml2==1.5._ pysaml2==7.2._ pycryptodome==3.16._ cffi==1.15.1 python-ldap==3.4.3 lxml

#### seafile 10

pip3 install --break-system-packages --timeout=3600 django==3.2._ future==0.18._ mysqlclient==2.1._ \
 pymysql pillow==9.3._ pylibmc captcha==0.4 markupsafe==2.0.1 jinja2 sqlalchemy==1.4.3 \
 psd-tools django-pylibmc django_simple_captcha==0.5._ pycryptodome==3.16._ cffi==1.15.1 lxml

mkdir /opt/seafile
cd /opt/seafile

useradd seafile -m -s /bin/bash
chown -R seafile: /opt/seafile

#chpasswd << 'END'
#seafile:P@ssw0rd
#END

su seafile -c 'wget -O seafile-pro-server_10.0.11_x86-64_Ubuntu.tar.gz "https://download.seafile.com/d/6e5297246c/files/?p=%2Fpro%2Fseafile-pro-server_10.0.11_x86-64_Ubuntu.tar.gz&dl=1"'
su seafile -c 'tar xf seafile-pro-server_10.0.11_x86-64_Ubuntu.tar.gz'
cd seafile-pro-server-10.0.11

su seafile -c 'echo -e "\nHome-Cloud\ncld.tn.lcl\n\n\n\n" | ./setup-seafile.sh'
cat << 'EOF' >> ../conf/seahub_settings.py
CACHES = {
'default': {
'BACKEND': 'django_pylibmc.memcached.PyLibMCCache',
'LOCATION': '127.0.0.1:11211',
},
}
EOF

cat << 'EOF' >> ../conf/seafile.conf
[memcached]
memcached_options = --SERVER=<the IP of Memcached Server> --POOL-MIN=10 --POOL-MAX=100
EOF
systemctl start nginx
systemctl enable nginx

touch /etc/nginx/sites-available/seafile.conf
rm /etc/nginx/sites-{enabled,available}/default
