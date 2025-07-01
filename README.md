# ProjetoLinuxCompass: Servidor Web Nginx com Monitoramento na AWS EC2

Este repositório documenta a configuração de um servidor web Nginx em uma instância EC2 da AWS, incluindo a criação da infraestrutura de rede (VPC), o deploy de uma página HTML personalizada e um script de monitoramento com notificações via webhook.

O projeto segue as etapas propostas pelo desafio "Configuração de Servidor Web com Monitoramento" da Turma de PB ABR 2025 - Programa de Bolsas DevSecOps.

## Tabela de Conteúdo

1.  [Visão Geral do Projeto](#1-visão-geral-do-projeto)
2.  [Etapa 1: Configuração do Ambiente na AWS](#2-etapa-1-configuração-do-ambiente-na-aws)
    * [Criar VPC e Sub-redes](#criar-vpc-e-sub-redes)
    * [Criar Internet Gateway](#criar-internet-gateway)
    * [Configurar Tabela de Rotas Públicas](#configurar-tabela-de-rotas-públicas)
    * [Criar Instância EC2](#criar-instância-ec2)
    * [Configurar Security Group](#configurar-security-group)
    * [Acessar a Instância via SSH](#acessar-a-instância-via-ssh)
3.  [Etapa 2: Configuração do Servidor Web (Nginx)](#3-etapa-2-configuração-do-servidor-web-nginx)
    * [Instalar Nginx](#instalar-nginx)
    * [Criar Página HTML Personalizada](#criar-página-html-personalizada)
    * [Configurar Nginx para Servir a Página](#configurar-nginx-para-servir-a-página)
    * [Configurar Nginx para Reinício Automático (systemd)](#configurar-nginx-para-reinício-automático-systemd)
4.  [Etapa 3: Monitoramento e Notificações](#4-etapa-3-monitoramento-e-notificações)
    * [Criar Script de Monitoramento](#criar-script-de-monitoramento)
5.  [Etapa 4: Testes e Documentação](#5-etapa-4-testes-e-documentação)

---

## 1. Visão Geral do Projeto

Este projeto tem como objetivo desenvolver e testar habilidades em Linux, AWS e automação, configurando um ambiente de servidor web monitorado. As principais etapas incluem:
* Configuração de ambiente de rede (VPC, sub-redes, Internet Gateway).
* Deploy de uma instância EC2 com Ubuntu.
* Instalação e configuração de um servidor web Nginx para exibir uma página HTML personalizada.
* Criação de um script Bash para monitorar a disponibilidade do site a cada minuto, com logs e envio de notificações via webhook (Discord/Telegram/Slack) em caso de indisponibilidade.
* Testes de implementação e documentação no GitHub.

## 2. Etapa 1: Configuração do Ambiente na AWS

Nesta etapa, configuramos a infraestrutura de rede na AWS para nossa instância EC2.

### Criar VPC e Sub-redes

1.  No console da AWS, navegue até o serviço **VPC**.
2.  Clique em "VPCs" e depois em "Criar VPC".
3.  **Nome:** `minha-vpc-nginx`
4.  **Bloco CIDR IPv4:** `10.0.0.0/16`
5.  Clique em **Criar VPC**.
6.  Em "Sub-redes", clique em "Criar sub-rede" e crie 4 sub-redes:
    * **publica-subnet-az1:** VPC `minha-vpc-nginx`, AZ `sa-east-1a`, CIDR `10.0.1.0/24`
    * **publica-subnet-az2:** VPC `minha-vpc-nginx`, AZ `sa-east-1b`, CIDR `10.0.2.0/24`
    * **privada-subnet-az1:** VPC `minha-vpc-nginx`, AZ `sa-east-1a`, CIDR `10.0.3.0/24`
    * **privada-subnet-az2:** VPC `minha-vpc-nginx`, AZ `sa-east-1b`, CIDR `10.0.4.0/24`

### Criar Internet Gateway

1.  No painel da VPC, clique em "Gateways de Internet".
2.  Clique em "Criar gateway da Internet".
3.  **Nome:** `minha-vpc-nginx-igw`
4.  Clique em **Criar gateway da Internet**.
5.  Selecione o IGW criado, clique em **Ações** e depois em **Anexar à VPC**. Selecione `minha-vpc-nginx`.

### Configurar Tabela de Rotas Públicas

1.  No painel da VPC, clique em "Tabelas de rotas".
2.  Crie uma nova tabela de rotas:
    * **Nome:** `minha-vpc-nginx-public-rt`
    * **VPC:** `minha-vpc-nginx`
    * Clique em **Criar tabela de rotas**.
3.  Selecione `minha-vpc-nginx-public-rt`.
4.  Na aba **Rotas**, clique em **Editar rotas**, depois em **Adicionar rota**:
    * **Destino:** `0.0.0.0/0`
    * **Alvo:** **Internet Gateway**, selecione `minha-vpc-nginx-igw`.
    * Clique em **Salvar rotas**.
5.  Na aba **Associações de sub-redes**, clique em **Editar associações de sub-redes**.
6.  Marque `publica-subnet-az1` e `publica-subnet-az2`.
7.  Clique em **Salvar associações**.

### Criar Instância EC2

1.  No console da AWS, navegue até o serviço **EC2**.
2.  Clique em "Instâncias" e depois em "Executar instâncias".
3.  **Escolha uma AMI:** "Ubuntu Server 22.04 LTS (HVM), SSD Volume Type".
4.  **Escolha o Tipo de Instância:** `t2.micro` (gratuito).
5.  **Configurar Detalhes da Instância:**
    * **Rede:** `minha-vpc-nginx`
    * **Sub-rede:** `publica-subnet-az1` (ou `publica-subnet-az2`).
    * **Atribuir IP público automaticamente:** **Habilitar**.
6.  **Adicionar Armazenamento:** Padrão (8 GiB).
7.  **Adicionar Tags:**

### Configurar Security Group

1.  **Crie um novo Security Group:**
    * **Nome do grupo de segurança:** `nginx-sg`
    * **Descrição:** `Security Group para Nginx server`
    * **Regras de entrada (Inbound Rules):**
        * **Tipo: SSH**, **Porta: 22**, **Fonte: Meu IP** (recomendado para SSH) ou `0.0.0.0/0`.
        * **Tipo: HTTP**, **Porta: 80**, **Fonte: 0.0.0.0/0** (para acesso público ao site).
    * **Regras de saída (Outbound Rules):**
        * Certifique-se de que há uma regra padrão: **Tipo: All traffic**, **Destino: 0.0.0.0/0**. (Isso é crucial para que a instância possa acessar a internet para atualizações e downloads).
2.  **Revisar e Executar:** Escolha ou crie um novo par de chaves (Key Pair) e faça o download do arquivo `.pem`. Guarde-o em segurança.

### Acessar a Instância via SSH

1.  **Obtenha o IP Público da Instância:** No console EC2, selecione sua instância e copie o **IP IPv4 público** ou o **DNS IPv4 público**.
2.  **Abra o Terminal (Windows PowerShell/CMD, Linux/macOS Terminal).**
3.  **Navegue até a pasta onde salvou o arquivo `.pem`:**
    ```bash
    cd /caminho/para/sua/pasta/com/a/chave
    ```
4.  **Defina as permissões corretas para o arquivo `.pem`:**
    * **Linux/macOS:** `chmod 400 nome-da-sua-chave.pem`
    * **Windows (Gráfico):** Clique direito > Propriedades > Segurança > Avançadas > Desabilitar Herança > Converter... > Remova todos exceto seu usuário > Edite seu usuário para ter apenas "Leitura".
5.  **Conecte-se via SSH:**
    ```bash
    ssh -i "nome-da-sua-chave.pem" ubuntu@SEU_IP_PUBLICO_EC2
    ```
    (Use `ubuntu` para AMIs Ubuntu. Para Amazon Linux, use `ec2-user`).

## 3. Etapa 2: Configuração do Servidor Web (Nginx)

Nesta etapa, instalamos e configuramos o Nginx para servir nossa página HTML.

### Instalar Nginx

1.  **Atualize os pacotes do sistema:**
    ```bash
    sudo apt update
    ```
2.  **Instale o Nginx:**
    ```bash
    sudo apt install nginx
    ```

### Criar Página HTML Personalizada

1.  Crie um diretório para seu site e a página `index.html`:
    ```bash
    cd /var/www/
    sudo mkdir teste
    cd teste
    sudo touch index.html
    sudo vi index.html
    ```
2.  Dentro do `vi` (ou `nano`), insira seu código HTML. Exemplo:
    ```html
    <!DOCTYPE html>
    <html>
    <head>
        <title>Meu Primeiro Site Nginx na AWS</title>
        <style>
            body { font-family: Arial, sans-serif; text-align: center; margin-top: 50px; background-color: #f0f0f0; }
            h1 { color: #333; }
            p { color: #666; }
        </style>
    </head>
    <body>
        <h1>Olá do Projeto Linux Compass!</h1>
        <p>Esta é a minha página web customizada servida pelo Nginx na AWS EC2.</p>
        <p>Data e Hora: <script>document.write(new Date().toLocaleString());</script></p>
    </body>
    </html>
    ```

### Configurar Nginx para Servir a Página

1.  **Desabilite o site padrão do Nginx** para evitar conflitos e garantir que sua configuração seja usada:
    ```bash
    sudo unlink /etc/nginx/sites-enabled/default
    ```
2.  **Crie e edite o arquivo de configuração do seu site:**
    ```bash
    sudo nano /etc/nginx/sites-enabled/teste
    ```
    (Note: É comum criar em `/etc/nginx/sites-available/` e depois linkar para `sites-enabled`, mas criar diretamente em `sites-enabled` também funciona para um único site).
3.  **Insira a seguinte configuração:**
    ```nginx
    server {
        listen 80;      # Porta padrão HTTP
        listen [::]:80; # Para IPv6

        # Se você estiver usando um Elastic IP ou um domínio, coloque-o aqui.
        # Caso contrário, o Nginx responderá para o IP da instância.
        server_name SEU_IP_PUBLICO_EC2; # Ex: 18.222.81.222, ou seu domínio (ex: example.ubuntu.com)

        root /var/www/teste;  # Caminho para sua página HTML
        index index.html;     # Nome do arquivo principal

        location / {
            try_files $uri $uri/ =404;
        }
    }
    ```
4.  **Teste a sintaxe do Nginx:**
    ```bash
    sudo nginx -t
    ```
    Deve retornar `syntax is ok` e `test is successful`.
5.  **Reinicie o Nginx para aplicar as alterações:**
    ```bash
    sudo systemctl restart nginx
    ```
6.  **Verifique o status do Nginx:**
    ```bash
    sudo systemctl status nginx
    ```
    Deve mostrar `Active: active (running)`.

### Configurar Nginx para Reinício Automático (systemd)

Para garantir que o Nginx reinicie automaticamente em caso de falha:

1.  **Edite a unidade de serviço do Nginx:**
    ```bash
    sudo systemctl edit nginx
    ```
    Isso abrirá um novo editor (geralmente `nano` ou `vi`) para um arquivo de "override" para o Nginx.
2.  **Adicione as seguintes linhas:**
    ```ini
    [Service]
    Restart=always
    RestartSec=3
    ```
    Isso fará com que o Nginx tente reiniciar sempre que parar, esperando 3 segundos antes de tentar novamente.
3.  **Salve e saia do editor.**
4.  **Recarregue a configuração do `systemd` para aplicar as mudanças:**
    ```bash
    sudo systemctl daemon-reload
    ```
5.  **Reinicie o Nginx para que a nova configuração de restart seja aplicada:**
    ```bash
    sudo systemctl restart nginx
    ```

## 4. Etapa 3: Monitoramento e Notificações

Nesta etapa, criamos um script para monitorar o site e enviar alertas.

### Criar Script de Monitoramento

1.  Crie o arquivo do script:
    ```bash
    cd /home/ubuntu
    touch monitora_site.sh
    nano monitora_site.sh
    ```
2.  **Insira o código do script:**
    ```bash
    #!/bin/bash

    # --- VARIÁVEIS DE CONFIGURAÇÃO ---
    SITE_URL="http://SEU_IP_PUBLICO_OU_ELASTIC_IP:80" # ATUALIZE para o seu IP fixo (Elastic IP) e porta.
    LOG_FILE="/var/log/monitoramento.log"
    DISCORD_WEBHOOK_URL="SUA_WEBHOOK_DO_DISCORD_AQUI" # SUBSTITUA pela sua URL do webhook do Discord/Telegram/Slack.

    # --- FUNÇÕES ---

    # Função para enviar notificação (adaptável para Discord, Telegram, Slack)
    send_notification() {
      MESSAGE="$1"
      # O -s (silent) suprime a barra de progresso, -o /dev/null descarta a saída,
      # -w (write out) permite formatar a saída. No entanto, para webhooks,
      # não precisamos da saída, apenas que a requisição seja feita.
      /usr/bin/curl -H "Content-Type: application/json" -X POST -d "{\"content\":\"$MESSAGE\"}" "$DISCORD_WEBHOOK_URL"
    }

    # Função para instalar a tarefa no crontab
    install_cron_job() {
      if ! /usr/bin/crontab -l 2>/dev/null | /usr/bin/grep -q "/home/ubuntu/monitora_site.sh"; then
        # Adiciona a tarefa cron para rodar a cada 1 minuto
        (/usr/bin/crontab -l 2>/dev/null; echo "* * * * * /bin/bash /home/ubuntu/monitora_site.sh >> /dev/null 2>&1") | /usr/bin/crontab -
        echo "Tarefa cron adicionada para 'monitora_site.sh' em /home/ubuntu/."
      else
        echo "Tarefa cron para 'monitora_site.sh' já existe."
      fi
    }

    # --- LÓGICA PRINCIPAL DO SCRIPT ---

    # Garante que o arquivo de log exista e com as permissões corretas
    /usr/bin/touch "$LOG_FILE"
    /usr/bin/chmod 644 "$LOG_FILE" # Garante permissões de leitura para outros usuários, útil para debugging

    # Se o script for chamado com "install-cron", apenas instala a tarefa cron e sai.
    if [[ "$1" == "install-cron" ]]; then
      install_cron_job
      exit 0
    fi

    # Realiza a requisição HTTP e verifica o código de status
    HTTP_CODE=$(/usr/bin/curl -s -o /dev/null -w "%{http_code}" "$SITE_URL")

    if [[ "$HTTP_CODE" == "200" ]]; then
      STATUS="DISPONÍVEL"
      echo "$(date): Site $SITE_URL está $STATUS." | /usr/bin/tee -a "$LOG_FILE"
    else
      STATUS="INDISPONÍVEL"
      MESSAGE="ALERTA: Site $SITE_URL está $STATUS! (HTTP Code: $HTTP_CODE)"
      echo "$(date): $MESSAGE" | /usr/bin/tee -a "$LOG_FILE"
      send_notification "$MESSAGE"
    fi
    ```
    * **ATENÇÃO:** Altere `SITE_URL` para o IP **fixo** da sua instância (Elastic IP) e a porta correta (80 ou 81).
    * **ATENÇÃO:** Substitua `SUA_WEBHOOK_DO_DISCORD_AQUI` pela URL real do seu webhook do Discord.

3.  **Dê permissão de execução ao script:**
    ```bash
    chmod +x /home/ubuntu/monitora_site.sh
    ```
4.  Instale a tarefa cron para rodar o script a cada minuto:
    ```bash
    /home/ubuntu/monitora_site.sh install-cron
    ```

## 5. Etapa 4: Testes e Documentação

### Testes da Implementação

1.  **Acessibilidade do Site:**
    * Abra seu navegador e digite `http://SEU_IP_PUBLICO_OU_ELASTIC_IP` (ou com a porta se for diferente de 80).
    * Verifique se sua página HTML personalizada é exibida corretamente.
2.  **Teste de Monitoramento (Parada do Nginx):**
    * No terminal da sua instância EC2, pare o Nginx: `sudo systemctl stop nginx`.
    * Aguarde 1-2 minutos (o tempo do cron).
    * Verifique o `/var/log/monitoramento.log` para ver as mensagens de "ALERTA: Site INDISPONÍVEL!".
    * Verifique se a notificação chegou no seu canal do Discord (ou Telegram/Slack).
    * Inicie o Nginx novamente: `sudo systemctl start nginx`. O script deve registrar que o site voltou a ficar "DISPONÍVEL".

### Documentação no GitHub

Este arquivo `README.md` serve como a documentação do projeto, explicando:
* A configuração do ambiente AWS.
* A instalação e configuração do servidor web Nginx.
* O funcionamento do script de monitoramento.
* Os testes e a validação da solução.
