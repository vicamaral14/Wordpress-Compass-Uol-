#!/bin/bash
set -e

# ================================
# Variáveis (substitua ou recupere via SSM/Secrets Manager)
# ================================
EFS_DNS_NAME="<EFS_DNS_NAME>.efs.<REGION>.amazonaws.com"
DB_HOST="<DB_HOST>.rds.<REGION>.amazonaws.com"
DB_NAME="<DB_NAME>"
DB_USER="<DB_USER>"
DB_PASSWORD="<DB_PASSWORD>"
EFS_MOUNT_DIR="/mnt/efs"
PROJECT_DIR="${EFS_MOUNT_DIR}/wordpress"

# ================================
# Atualizar pacotes
# ================================
apt update -y
apt upgrade -y

# ================================
# Instalar pacotes necessários
# ================================
apt install -y nfs-common docker.io docker-compose

# ================================
# Criar diretório do EFS e montar
# ================================
mkdir -p ${EFS_MOUNT_DIR}
mount -t nfs4 -o nfsvers=4.1 ${EFS_DNS_NAME}:/ ${EFS_MOUNT_DIR}

# Garantir montagem automática no reboot
echo "${EFS_DNS_NAME}:/ ${EFS_MOUNT_DIR} nfs4 defaults,_netdev 0 0" >> /etc/fstab

# ================================
# Criar diretório do projeto
# ================================
mkdir -p ${PROJECT_DIR}

# ================================
# Criar docker-compose.yml no EFS
# ================================
cat <<EOF > ${PROJECT_DIR}/docker-compose.yml
version: '3.8'

services:
  db:
    image: mysql:8.0
    container_name: wp-db
    restart: always
    environment:
      MYSQL_DATABASE: \${DB_NAME}
      MYSQL_USER: \${DB_USER}
      MYSQL_PASSWORD: "\${DB_PASSWORD}"
      MYSQL_ROOT_PASSWORD: "\${DB_PASSWORD}"
    volumes:
      - db_data:/var/lib/mysql

  wordpress:
    image: wordpress:latest
    container_name: wp-app
    depends_on:
      - db
    restart: always
    ports:
      - "80:80"
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: \${DB_USER}
      WORDPRESS_DB_PASSWORD: "\${DB_PASSWORD}"
      WORDPRESS_DB_NAME: \${DB_NAME}
    volumes:
      - wp_data:/var/www/html

volumes:
  db_data:
    driver: local
    driver_opts:
      type: "nfs"
      o: "addr=\${EFS_DNS_NAME},nfsvers=4.1,rw"
      device: ":/db"
  wp_data:
    driver: local
    driver_opts:
      type: "nfs"
      o: "addr=\${EFS_DNS_NAME},nfsvers=4.1,rw"
      device: ":/wordpress"
EOF

# ================================
# Subir containers do WordPress e MySQL
# ================================
docker-compose -f ${PROJECT_DIR}/docker-compose.yml up -d
