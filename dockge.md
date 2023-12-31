# Dockge

# install dockge

mkdir -p /opt/{stacks,dockge}
cd /opt/dockge
curl https://raw.githubusercontent.com/louislam/dockge/master/compose.yaml --output compose.yaml
docker compose up -d
