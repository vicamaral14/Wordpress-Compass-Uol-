#!/bin/bash

# User Data Script - Instalação Completa WordPress + Docker + EFS (sem variáveis sensíveis)

# Log de instalação
exec > >(tee /var/log/wordpress-install.log) 2>&1

echo "=== 🚀 INICIANDO INSTALAÇÃO AUTOMATIZADA DO WORDPRESS ==="

# [1/8] Atualizar sistema
apt update -y
apt upgrade -y

# [2/8] Instalar dependências básicas
apt install -y apt-transport-https ca-certificates curl gnupg lsb-release nfs-common

# [3/8] Configurar EFS (Opcional)
mkdir -p /mnt/efs

# ⚠️ Substituir pelo ID/DNS correto via variável
EFS_ID="${EFS_ID:-fs-XXXXXXXX}"
EFS_REGION="${EFS_REGION:-us-east-1}"

EFS_MOUNTED=false
for i in {1..2}; do
    if timeout 30 mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=60,retrans=1,noresvport \
        $EFS_ID.efs.$EFS_REGION.amazonaws.com:/ /mnt/efs 2>/dev/null; then
        EFS_MOUNTED=true
        break
    else
        sleep 5
    fi
done

if [ "$EFS_MOUNTED" = true ]; then
    echo "$EFS_ID.efs.$EFS_REGION.amazonaws.com:/ /mnt/efs nfs4 defaults 0 0" >> /etc/fstab
    mkdir -p /mnt/efs/wordpress /mnt/efs/mysql-data
fi

# [4/8] Instalar Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
systemctl start docker
systemctl enable docker
usermod -aG docker ubuntu

# [6/8] Criar projeto WordPress
mkdir -p /home/ubuntu/wordpress-project
cd /home/ubuntu/wordpress-project

cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  wordpress:
    image: wordpress:latest
    container_name: wordpress
    restart: unless-stopped
    ports:
      - "80:80"
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: ${WORDPRESS_DB_USER}
      WORDPRESS_DB_PASSWORD: ${WORDPRESS_DB_PASSWORD}
      WORDPRESS_DB_NAME: ${WORDPRESS_DB_NAME}
    volumes:
      - wordpress_data:/var/www/html
    depends_on:
      - db
    networks:
      - wordpress_network

  db:
    image: mysql:8.0
    container_name: mysql
    restart: unless-stopped
    environment:
      MYSQL_DATABASE: ${WORDPRESS_DB_NAME}
      MYSQL_USER: ${WORDPRESS_DB_USER}
      MYSQL_PASSWORD: ${WORDPRESS_DB_PASSWORD}
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - wordpress_network

volumes:
  wordpress_data:
  db_data:

networks:
  wordpress_network:
    driver: bridge
EOF

# [7/8] Subir containers
docker compose up -d
