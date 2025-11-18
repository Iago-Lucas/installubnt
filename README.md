#!/bin/bash
# Script de correção rápida para UniFi

echo "Corrigindo dependências do UniFi..."

# Instalar dependências faltantes
sudo apt update
sudo apt install -y binutils jsvc

# Corrigir instalação do UniFi
sudo apt install -f -y

# Parar serviços
sudo systemctl stop unifi 2>/dev/null || true
sudo systemctl stop mongod 2>/dev/null || true

# Reinstalar MongoDB compatível
sudo apt remove --purge -y mongodb-org*
wget -qO - https://www.mongodb.org/static/pgp/server-4.4.asc | sudo apt-key add -
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/4.4 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-4.4.list
sudo apt update
sudo apt install -y mongodb-org

# Configurar MongoDB
sudo systemctl start mongod
sudo systemctl enable mongod

# Reinstalar UniFi
sudo dpkg -i /tmp/unifi_sysvinit_all.deb
sudo apt install -f -y

# Iniciar UniFi
sudo systemctl start unifi

echo "Correção aplicada! Verifique o status:"
echo "sudo systemctl status unifi"
echo "Acesse: https://$(hostname -I | awk '{print $1}'):8443"
