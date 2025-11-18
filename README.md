#!/bin/bash

# Script de Instalação Corrigido - UniFi Controller v6
# Para Ubuntu 24.04 com problemas de dependências
# Versão: 2.0

set -e

# Cores para output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

log_info() {
    echo -e "${GREEN}[INFO]${NC} $1"
}

log_warn() {
    echo -e "${YELLOW}[WARN]${NC} $1"
}

log_error() {
    echo -e "${RED}[ERROR]${NC} $1"
}

# Resolver dependências específicas do UniFi
install_unifi_dependencies() {
    log_info "Instalando dependências específicas do UniFi..."
    
    # Instalar dependências faltantes
    sudo apt install -y binutils jsvc
    
    # Para o MongoDB, vamos usar uma versão compatível
    log_info "Configurando MongoDB compatível..."
    
    # Parar MongoDB atual se estiver rodando
    sudo systemctl stop mongod 2>/dev/null || true
    
    # Remover MongoDB atual se existir
    sudo apt remove --purge -y mongodb-org* 2>/dev/null || true
    sudo apt autoremove -y
    
    # Instalar MongoDB compatível (versão 4.4)
    wget -qO - https://www.mongodb.org/static/pgp/server-4.4.asc | sudo apt-key add -
    
    echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/4.4 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-4.4.list
    
    sudo apt update
    sudo apt install -y mongodb-org
    
    # Corrigir permissões do MongoDB
    sudo mkdir -p /data/db
    sudo chown -R mongodb:mongodb /data/db
    
    # Iniciar MongoDB
    sudo systemctl start mongod
    sudo systemctl enable mongod
    
    # Verificar se MongoDB está rodando
    if sudo systemctl is-active --quiet mongod; then
        log_info "MongoDB instalado e rodando"
    else
        log_error "Falha ao iniciar MongoDB"
        sudo systemctl status mongod
        exit 1
    fi
}

# Instalar UniFi com tratamento de erros
install_unifi_corrected() {
    local UNIFI_VERSION="6.5.55"
    local UNIFI_URL="https://dl.ui.com/unifi/${UNIFI_VERSION}/unifi_sysvinit_all.deb"
    
    log_info "Instalando UniFi Controller versão ${UNIFI_VERSION}..."
    
    # Baixar pacote
    cd /tmp
    if wget -q "$UNIFI_URL"; then
        log_info "Pacote UniFi baixado com sucesso"
    else
        log_error "Falha ao baixar pacote UniFi"
        exit 1
    fi
    
    # Tentar instalar forçando dependências
    log_info "Instalando pacote UniFi..."
    
    # Primeira tentativa normal
    if sudo dpkg -i unifi_sysvinit_all.deb; then
        log_info "Instalação bem-sucedida"
    else
        log_warn "Problemas com dependências, tentando corrigir..."
        sudo apt install -f -y
    fi
    
    # Verificar se a instalação foi bem-sucedida
    if dpkg -l | grep -q unifi; then
        log_info "UniFi instalado com sucesso"
    else
        log_error "Falha na instalação do UniFi"
        exit 1
    fi
    
    # Limpar arquivo
    rm -f unifi_sysvinit_all.deb
}

# Método alternativo se o anterior falhar
install_unifi_alternative() {
    log_info "Usando método alternativo de instalação..."
    
    # Instalar via repositório alternativo
    wget -q -O /tmp/unifi.key https://dl.ui.com/unifi/unifi-repo.key
    sudo cp /tmp/unifi.key /etc/apt/trusted.gpg.d/unifi.asc
    
    echo 'deb [ signed-by=/etc/apt/trusted.gpg.d/unifi.asc ] https://www.ui.com/downloads/unifi/debian stable ubiquiti' | sudo tee /etc/apt/sources.list.d/unifi.list
    
    sudo apt update
    sudo apt install -y unifi
}

# Corrigir serviço UniFi
fix_unifi_service() {
    log_info "Configurando serviço UniFi..."
    
    # Parar serviço se estiver rodando
    sudo systemctl stop unifi 2>/dev/null || true
    
    # Corrigir permissões
    sudo chown -R unifi:unifi /usr/lib/unifi/*
    sudo chmod -R 755 /usr/lib/unifi
    
    # Configurar Java correto
    sudo update-alternatives --config java 2>/dev/null || true
    
    # Recarregar daemons
    sudo systemctl daemon-reload
    
    # Iniciar serviço
    sudo systemctl start unifi
    sleep 10
    
    # Verificar status
    local max_attempts=5
    local attempt=1
    
    while [ $attempt -le $max_attempts ]; do
        if sudo systemctl is-active --quiet unifi; then
            log_info "UniFi Controller está rodando!"
            return 0
        fi
        log_warn "Tentativa $attempt: UniFi ainda não está rodando, aguardando..."
        sleep 10
        ((attempt++))
    done
    
    log_error "UniFi não iniciou após $max_attempts tentativas"
    log_info "Verificando logs..."
    sudo journalctl -u unifi -n 20 --no-pager
    return 1
}

# Script principal corrigido
main_corrected() {
    clear
    echo "========================================="
    echo "  Instalador Corrigido UniFi Controller"
    echo "  Resolvendo problemas de dependências"
    echo "========================================="
    echo
    
    log_info "Iniciando instalação corrigida..."
    
    # Atualizar sistema
    sudo apt update && sudo apt upgrade -y
    
    # Instalar Java
    sudo apt install -y openjdk-17-jre-headless wget curl
    
    # Instalar dependências específicas
    install_unifi_dependencies
    
    # Tentar instalação normal
    if install_unifi_corrected; then
        log_info "Instalação principal bem-sucedida"
    else
        log_warn "Método principal falhou, tentando alternativo..."
        install_unifi_alternative
    fi
    
    # Configurar firewall
    log_info "Configurando firewall..."
    sudo ufw allow 22/tcp
    sudo ufw allow 8080/tcp
    sudo ufw allow 8443/tcp
    sudo ufw allow 8880/tcp
    sudo ufw allow 8843/tcp
    sudo ufw allow 3478/udp
    sudo ufw allow 10001/udp
    sudo ufw --force enable
    
    # Corrigir serviço
    if fix_unifi_service; then
        local IP_ADDRESS=$(hostname -I | awk '{print $1}')
        echo
        log_info "=== INSTALAÇÃO CONCLUÍDA ==="
        echo
        echo -e "${GREEN}UniFi Controller instalado com sucesso!${NC}"
        echo
        echo "Acesse a interface web em:"
        echo -e "${YELLOW}https://${IP_ADDRESS}:8443${NC}"
        echo
        echo "A primeira inicialização pode levar alguns minutos..."
        echo
        echo "Para verificar status: sudo systemctl status unifi"
        echo "Para ver logs: sudo tail -f /var/log/unifi/server.log"
    else
        log_error "Instalação encontrou problemas"
        log_info "Tentando diagnóstico..."
        
        # Diagnóstico
        echo
        log_info "=== DIAGNÓSTICO ==="
        echo "Java: $(java -version 2>&1 | head -1)"
        echo "MongoDB: $(sudo systemctl status mongod --no-pager | grep Active)"
        echo "UniFi: $(sudo systemctl status unifi --no-pager | grep Active)"
        echo "Porta 8443: $(ss -tlnp | grep 8443 || echo 'Não ouvindo')"
    fi
}

# Executar
main_corrected

--------------------------------------

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
