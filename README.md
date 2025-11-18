#!/bin/bash
# ==================================================================
# Script: Instala UniFi Network Controller + MongoDB 7.0 no Ubuntu 24.04
# Testado e funcionando em novembro 2025
# ==================================================================

set -e  # Para o script em caso de erro

echo "=== Atualizando sistema ==="
apt update && apt upgrade -y

echo "=== Instalando dependências básicas ==="
apt install -y ca-certificates apt-transport-https gnupg curl wget lsb-release

echo "=== Removendo repositórios antigos do MongoDB (se existirem) ==="
rm -f /etc/apt/sources.list.d/mongodb-org*.list
rm -f /usr/share/keyrings/mongodb-*.gpg

echo "=== Configurando MongoDB 7.0 (repositório jammy — funciona perfeitamente no 24.04) ==="
# Baixa a chave oficial do MongoDB 7.0
curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc -o /tmp/mongodb-server-7.0.asc

# Importa e converte a chave
gpg --import /tmp/mongodb-server-7.0.asc
gpg --export 9685A6BC3C33C2B9 | gpg --dearmor -o /usr/share/keyrings/mongodb-server-7.0.gpg
chmod 644 /usr/share/keyrings/mongodb-server-7.0.gpg
rm -f /tmp/mongodb-server-7.0.asc

# Adiciona o repositório (jammy = funciona, noble ainda dá erro de assinatura)
cat > /etc/apt/sources.list.d/mongodb-org-7.0.list <<EOF
deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse
EOF

echo "=== Instalando MongoDB 7.0 ==="
apt update
apt install -y mongodb-org

systemctl start mongod
systemctl enable mongod
echo "MongoDB instalado e rodando"

echo "=== Adicionando repositório oficial da Ubiquiti (UniFi) ==="
wget -qO- https://dl.ui.com/unifi/unifi-repo.gpg | gpg --dearmor -o /usr/share/keyrings/ubiquiti-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/ubiquiti-archive-keyring.gpg] https://www.ui.com/downloads/unifi/debian stable ubiquiti" > /etc/apt/sources.list.d/100-ubnt-unifi.list

echo "=== Instalando UniFi Network Controller ==="
apt update
apt install -y unifi

systemctl start unifi
systemctl enable unifi

echo "=== Instalação concluída com sucesso! ==="
echo ""
echo "Acesse o UniFi Controller em:"
echo "https://$(hostname -I | awk '{print $1}'):8443"
echo "(ignore o aviso de certificado — é normal na primeira vez)"
echo ""
echo "MongoDB está rodando na porta 27017"
echo "UniFi está rodando na porta 8443"
echo ""
echo "Dica: se o startup do UniFi for muito lento em VM, instale:"
echo "   apt install -y haveged"
echo ""

exit 0
