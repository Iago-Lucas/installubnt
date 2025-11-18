#!/bin/bash
# Instalador rápido UniFi
sudo apt update && sudo apt upgrade -y
sudo apt install -y wget openjdk-17-jre-headless
wget -qO - https://www.mongodb.org/static/pgp/server-6.0.asc | sudo apt-key add -
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/6.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-6.0.list
sudo apt update && sudo apt install -y mongodb-org
sudo systemctl start mongod && sudo systemctl enable mongod
cd /tmp && wget https://dl.ui.com/unifi/6.5.55/unifi_sysvinit_all.deb
sudo dpkg -i unifi_sysvinit_all.deb && sudo apt install -f -y
sudo systemctl start unifi && sudo systemctl enable unifi
echo "Instalação completa! Acesse: https://$(hostname -I | awk '{print $1}'):8443"
