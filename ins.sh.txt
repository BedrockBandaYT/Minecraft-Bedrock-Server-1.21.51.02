#!/bin/bash

# Colors for output
GREEN='\033[0;32m'
NC='\033[0m'

echo -e "${GREEN}Updating system...${NC}"
sudo apt update -y && sudo apt upgrade -y

echo -e "${GREEN}Installing dependencies...${NC}"
sudo apt install -y curl unzip tar git docker.io docker-compose

echo -e "${GREEN}Starting & Enabling Docker...${NC}"
sudo systemctl start docker
sudo systemctl enable docker

echo -e "${GREEN}Creating directories for Pterodactyl...${NC}"
cd /pterodactyl/panel 

echo -e "${GREEN}Downloading and configuring Docker Compose...${NC}"
cat > /pterodactyl/panel/docker-compose.yml <<EOL
version: '3.8'

x-common:

  database:

    &db-environment

    MYSQL_PASSWORD: &db-password "sgyy"
    MYSQL_ROOT_PASSWORD: "sgyt"

  panel:

    &panel-environment

    APP_URL: "https://pterodactyl.example.com"
    APP_TIMEZONE: "UTC"
    APP_SERVICE_AUTHOR: "noreply@example.com"
    TRUSTED_PROXIES: "*"

  mail:

    &mail-environment

    MAIL_FROM: "noreply@example.com"
    MAIL_DRIVER: "smtp"
    MAIL_HOST: "mail"
    MAIL_PORT: "1025"
    MAIL_USERNAME: ""
    MAIL_PASSWORD: ""
    MAIL_ENCRYPTION: "true"

services:

  database:
    image: mariadb:10.5
    restart: always
    command: --default-authentication-plugin=mysql_native_password
    volumes:
      - "/srv/pterodactyl/database:/var/lib/mysql"
    environment:
      <<: *db-environment
      MYSQL_DATABASE: "panel"
      MYSQL_USER: "pterodactyl"

  cache:
    image: redis:alpine
    restart: always

  panel:
    image: ghcr.io/pterodactyl/panel:latest
    restart: always
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - database
      - cache
    volumes:
      - "/srv/pterodactyl/var/:/app/var/"
      - "/srv/pterodactyl/nginx/:/etc/nginx/http.d/"
      - "/srv/pterodactyl/certs/:/etc/letsencrypt/"
      - "/srv/pterodactyl/logs/:/app/storage/logs"
    environment:
      <<: [*panel-environment, *mail-environment]
      DB_PASSWORD: *db-password
      APP_ENV: "production"
      APP_ENVIRONMENT_ONLY: "false"
      CACHE_DRIVER: "redis"
      SESSION_DRIVER: "redis"
      QUEUE_DRIVER: "redis"
      REDIS_HOST: "cache"
      DB_HOST: "database"
      DB_PORT: "3306"

networks:
  default:
    ipam:
      config:
        - subnet: 172.20.0.0/16
EOL

echo -e "${GREEN}Starting Pterodactyl with Docker Compose...${NC}"
docker-compose up -d

echo -e "${GREEN}Installation Complete! Access your panel at https://pterodactyl.example.com${NC}"
