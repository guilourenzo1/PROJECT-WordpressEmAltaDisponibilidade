# Projeto - Wordpress - Em Alta Disponibilidade

Este projeto hospeda um site **Wordpress** utilizando ferramentas **AWS** como: **VPCs, EFS, RDS, Instâncias EC2, Application Load Balancer e Auto Scaling Group** a fim de simular um ambiente de produção real, no qual interrupções não podem causar indisponibilidade da aplicação. Além disso, o uso de serviços gerenciados permite focar na lógica de implantação e escalabilidade, sem a necessidade de gerenciar manualmente servidores de banco de dados ou sistemas de arquivos distribuídos.

---

## Sumário

1. 

---
## Criação da VPC

• Entre na aba de VPC na AWS  
• Criar VPC  
• VPC e muito mais  
• Nome do projeto: projeto-wordpress  
• 2 AZs  
• 2 Sub-redes públicas  
• 4 Sub-redes privadas  
• 1 NAT Gateway por AZ  
• Gateway do S3  

---

## Criação dos Security Groups

### • Security Group do Load Balancer
        • Nome: SG ALB
        • Regra de entrada: Todo o tráfego -> 0.0.0.0/0   
        • Regra de saída: Todo o tráfego -> 0.0.0.0/0

### • Security Group da EC2
        • Nome: SG EC2
        • Regra de entrada 1: HTTP -> 80 -> SG ALB 
        • Regra de entrada 2: HTTPs -> 443 -> SG ALB 
        • Regra de saída: Todo o tráfego -> 0.0.0.0/0

### • Security Group do RDS
        • Nome: SG RDS
        • Regra de entrada: MySQL/Aurora -> 3306 -> SG EC2
        • Regra de saída: Todo o tráfego -> 0.0.0.0/0

### • Security Group do EFS
        • Nome: SG EFS
        • Regra de entrada: NFS -> 2049 -> SG EC2
        • Regra de saída: Todo o tráfego -> 0.0.0.0/0
---

## Criação do RDS

• Entre na aba Aurora and RDS na AWS  
• Crias Banco de dados  
• My SQL  
• Nível gratuito  
• Identificador da instância: projeto-wordpressDB  
• Coloque um nome para o usuário e defina uma senha (guarde-a)  
• Configuração da instância: db.t3.micro  
• Security Group do RDS  
• Configurações adicionais -> Nome do banco de dados -> wordpress  

<sub> _O que não foi citado deixe como padrão_ </sub>

---


## Criação do EFS

• Entre na aba EFS na AWS
• Nome -> projeto-wordpressEFS
• Regional
• Próximo ->
• Zona de disponibilidade -> us-east-1a e us-east-1b
• ID da sub-rede -> sub-nets públicas  
• Grupos de segurança -> SG NFS  

<sub> _O que não foi citado deixe como padrão_ </sub>


---

## Criação Launch Template

• Entre na aba da EC2 na AWS  
• Modelo de execução  
• Nome: projeto-wordpresslt  
• Início rápido -> Ubuntu -> Versão 22.04 (gratuíta)  
• Tipo de instância: t2.micro  
• Grupo de segurança: SG EC2  
• Atribua um ip público  

### Script do Launch Template

• Clique na lista suspensa de Detalhes avançados
• Cole esse script user-data ao fim da tela
<details>
  <summary> Clique aqui para ver o script user-data </summary>

  ```bash
#!/bin/bash
 
# User Data Script - Instalação Completa WordPress + Docker + EFS
# Este script automatiza toda a configuração necessária para uma nova instância EC2
 
# Log de instalação
exec > >(tee /var/log/wordpress-install.log) 2>&1
 
echo "=== 🚀 INICIANDO INSTALAÇÃO AUTOMATIZADA DO WORDPRESS ==="
echo "Data/Hora: $(date)"
echo "Hostname: $(hostname)"
echo "IP Privado: $(curl -s http://169.254.169.254/latest/meta-data/local-ipv4 2>/dev/null || echo 'N/A')"
echo "IP Público: $(curl -s http://169.254.169.254/latest/meta-data/public-ipv4 2>/dev/null || echo 'N/A')"
echo "Região: $(curl -s http://169.254.169.254/latest/meta-data/placement/region 2>/dev/null || echo 'N/A')"
echo "AZ: $(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone 2>/dev/null || echo 'N/A')"
echo "========================================================"
 
# [1/8] Atualizar sistema
echo "[1/8] 📦 Atualizando sistema..."
apt update -y
apt upgrade -y
echo "✅ Sistema atualizado"
 
# [2/8] Instalar dependências básicas
echo "[2/8] 🔧 Instalando dependências..."
apt install -y apt-transport-https ca-certificates curl gnupg lsb-release nfs-common
echo "✅ Dependências instaladas"
 
# [3/8] Configurar EFS
echo "[3/8] 💾 Configurando EFS (opcional)..."
mkdir -p /mnt/efs
 
EFS_ID="<SEU EFS_ID>"
EFS_REGION="us-east-1"
 
EFS_MOUNTED=false
for i in {1..2}; do
    if timeout 30 mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=60,retrans=1,noresvport \
       ${EFS_ID}.efs.${EFS_REGION}.amazonaws.com:/ /mnt/efs 2>/dev/null; then
        echo "✅ EFS montado com sucesso na tentativa $i"
        EFS_MOUNTED=true
        break
    else
        echo "⚠️ Tentativa $i de montagem do EFS falhou, continuando..."
        sleep 5
    fi
done
 
if [ "$EFS_MOUNTED" = true ]; then
    echo "${EFS_ID}.efs.${EFS_REGION}.amazonaws.com:/ /mnt/efs nfs4 nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=60,retrans=1,noresvport 0 0" >> /etc/fstab
    mkdir -p /mnt/efs/wordpress
    echo "✅ EFS configurado"
else
    echo "⚠️ EFS não disponível - continuando com volume local"
fi
 
# [4/8] Instalar Docker
echo "[4/8] 🐳 Instalando Docker..."
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
 
# Configurar Docker
systemctl start docker
systemctl enable docker
DEFAULT_USER=$(whoami)
usermod -aG docker $DEFAULT_USER
echo "✅ Docker instalado: $(docker --version)"
 
# [5/8] Docker Compose já incluído
echo "[5/8] 🔨 Docker Compose já incluído"
echo "✅ Docker Compose disponível: $(docker compose version)"
 
# [6/8] Criar projeto WordPress
echo "[6/8] 📁 Criando projeto WordPress..."
mkdir -p /home/ubuntu/wordpress-project
cd /home/ubuntu/wordpress-project
chown -R ubuntu:ubuntu /home/ubuntu/wordpress-project
 
# Criar docker-compose.yml
echo "Criando docker-compose.yml..."
cat > /home/ubuntu/wordpress-project/docker-compose.yml << EOF
version: '3.8'
 
services:
  wordpress:
    image: wordpress:latest
    container_name: wordpress
    restart: unless-stopped
    ports:
      - "80:80"
    environment:     
      WORDPRESS_DB_HOST: <SEU_DB_HOST>:3306
      WORDPRESS_DB_USER: <SEU_DB_USER>
      WORDPRESS_DB_PASSWORD: <SUA_SENHA>
      WORDPRESS_DB_NAME: <SEU_DB_NAME>
    volumes:
EOF

# Definir volume dependendo do EFS
if [ "$EFS_MOUNTED" = true ]; then
    echo "      - /mnt/efs/wordpress:/var/www/html" >> /home/ubuntu/wordpress-project/docker-compose.yml
else
    echo "      - wordpress_data:/var/www/html" >> /home/ubuntu/wordpress-project/docker-compose.yml
fi

cat >> /home/ubuntu/wordpress-project/docker-compose.yml << 'EOF'
    networks:
      - wordpress_network
 
volumes:
  wordpress_data:
 
networks:
  wordpress_network:
    driver: bridge
EOF
 
echo "✅ Docker Compose configurado"
 
# [7/8] Iniciar containers
echo "[7/8] 🚀 Iniciando containers..."
cd /home/ubuntu/wordpress-project
docker compose up -d
 
# Aguardar containers iniciarem
echo "Aguardando containers iniciarem..."
sleep 20
 
# Verificar container
if docker ps --format '{{.Names}}' | grep -q wordpress; then
    echo "✅ WordPress container ativo"
else
    echo "❌ WordPress não subiu"
fi
 
# [8/8] Finalização
PUBLIC_IP=$(curl -s http://169.254.169.254/latest/meta-data/public-ipv4 2>/dev/null || echo 'IP-DA-INSTANCIA')
 
echo ""
echo "========================================================"
echo "🎉 INSTALAÇÃO CONCLUÍDA"
echo "🌍 WordPress URL: http://$PUBLIC_IP"
echo "📋 Logs: /var/log/wordpress-install.log"
echo "========================================================"

  ```
</details>
<sub> _O que não foi citado deixe como padrão_ </sub>

---

## Criação do Target Group
• Entre na aba da EC2 na AWS -> Grupos de Destino
• Nome -> projeto-wordpressTG  
• Selecione a VPC que foi criada  
• Código de Sucesso -> 200-399  

<sub> _O que não foi citado deixe como padrão_ </sub>

---

## Criação do Application Load Balancers
• Entre na aba da EC2 na AWS -> Load Balancers  
• Criar Load Balancers -> Selecione Application Load Balancer  
• Nome -> projeto-wordpressALB  
• Selecione a VPC criada  
• Zona de disponibilidade e sub-redes -> Marque as duas check boxes -> sub-redes públicas
• Grupo de segurança -> SG ALB  
• Grupo de destino -> projeto-wordpressTG  

<sub> _O que não foi citado deixe como padrão_ </sub>

---

## Criação Auto Scaling Group
• Nome -> projeto-wordpressASG  
• Modelo de execução -> projeto-wordpresslt  
• Selecione a VPC que foi criada  
• Zonas de disponibilidades e sub-redes -> Todas as sub-redes privadas  
• Anexar a um balanceador de carga existente -> Escolha entre seus grupos de destino de balanceador de carga -> Target group que foi criado  
• Capacidade desejada -> 2  
• Selecione a opção "Política de dimensionamento com monitoramento do objetivo"  

<sub> _O que não foi citado deixe como padrão_ </sub>

---

<img width="1601" height="381" alt="image" src="https://github.com/user-attachments/assets/b0769ca9-c85a-47b1-8dd8-1b59d4125f3c" />

• Esta é a mensagem de retorno desejada  
<sub> Para checar vá em Load Balancer -> Mapa de recursos  

---

## Site Worpress funcionando
• Site Worpress funcionando via AWS:
<img width="1919" height="920" alt="image" src="https://github.com/user-attachments/assets/d6c8aaaa-8b43-4e98-9101-95e26a61a3f6" />





