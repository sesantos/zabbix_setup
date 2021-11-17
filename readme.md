# Download docker-compose 
curl -L https://github.com/docker/compose/releases/download/v2.1.0/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose
ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose 
sudo chmod +x /usr/local/bin/docker-compose

# Clone Zabbix-Docker repo
git clone https://github.com/zabbix/zabbix-docker.git

mkdir -p zbx_env/usr/share/zabbix/modules

# Create a Zabbix instance
docker-compose -f docker-compose_v3_ubuntu_mysql_latest.yaml up -d

# Disable TCP Checksum offload
docker network ls | grep zabbix-docker_srlinux-mgmt
ip a | grep 5885ebe438ec | grep UP | grep veth| awk '{print $2}' | cut -f1 -d"@" | while read a; do ethtool -K $a tx off; done;
