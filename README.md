# ProjetoLinuxCompass: Servidor Web Nginx com Monitoramento na AWS EC2

Este repositório documenta a configuração de um servidor web Nginx em uma instância EC2 da AWS, incluindo a criação da infraestrutura de rede (VPC), o deploy de uma página HTML personalizada e um script de monitoramento com notificações via webhook.

[cite_start]O projeto segue as etapas propostas pelo desafio "Configuração de Servidor Web com Monitoramento" da Turma de PB ABR 2025 - Programa de Bolsas DevSecOps[cite: 24, 25, 26].

## Tabela de Conteúdo

1.  [Visão Geral do Projeto](#1-visão-geral-do-projeto)
2.  [Etapa 1: Configuração do Ambiente na AWS](#2-etapa-1-configuração-do-ambiente-na-aws)
    * [Criar VPC e Sub-redes](#criar-vpc-e-sub-redes)
    * [Criar Internet Gateway](#criar-internet-gateway)
    * [Configurar Tabela de Rotas Públicas](#configurar-tabela-de-rotas-públicas)
    * [Criar Instância EC2](#criar-instância-ec2)
    * [Configurar Security Group](#configurar-security-group)
    * [Acessar a Instância via SSH](#acessar-a-instância-via-ssh)
    * [**Solução de Problemas: Erro de Conexão SSH / "Failed to connect"**](#solução-de-problemas-erro-de-conexão-ssh--failed-to-connect)
    * [**Dica: Como Manter o IP Público Fixo (Elastic IP)**](#dica-como-manter-o-ip-público-fixo-elastic-ip)
3.  [Etapa 2: Configuração do Servidor Web (Nginx)](#3-etapa-2-configuração-do-servidor-web-nginx)
    * [Instalar Nginx](#instalar-nginx)
    * [Criar Página HTML Personalizada](#criar-página-html-personalizada)
    * [Configurar Nginx para Servir a Página](#configurar-nginx-para-servir-a-página)
    * [Configurar Nginx para Reinício Automático (systemd)](#configurar-nginx-para-reinício-automático-systemd)
    * [**Solução de Problemas: "Network is unreachable" ao instalar Nginx**](#solução-de-problemas-network-is-unreachable-ao-instalar-nginx)
    * [**Solução de Problemas: Nginx Falha ao Iniciar / "unexpected {"**](#solução-de-problemas-nginx-falha-ao-iniciar--unexpected)
    * [**Solução de Problemas: Nginx Apresenta Página Padrão**](#solução-de-problemas-nginx-apresenta-página-padrão)
    * [**Solução de Problemas: Erro ao Parar Nginx / "Unit nginx.service does not exist"**](#solução-de-problemas-erro-ao-parar-nginx--unit-nginxservice-does-not-exist)
4.  [Etapa 3: Monitoramento e Notificações](#4-etapa-3-monitoramento-e-notificações)
    * [Criar Script de Monitoramento](#criar-script-de-monitoramento)
    * [Configurar Cron Job](#configurar-cron-job)
    * [**Solução de Problemas: Notificação Discord Não Funciona**](#solução-de-problemas-notificação-discord-não-funciona)
5.  [Etapa 4: Testes e Documentação](#5-etapa-4-testes-e-documentação)

---

## 1. Visão Geral do Projeto

[cite_start]Este projeto tem como objetivo desenvolver e testar habilidades em Linux, AWS e automação, configurando um ambiente de servidor web monitorado[cite: 25, 26]. As principais etapas incluem:
* [cite_start]Configuração de ambiente de rede (VPC, sub-redes, Internet Gateway). [cite: 7]
* [cite_start]Deploy de uma instância EC2 com Ubuntu. [cite: 8, 33, 34]
* [cite_start]Instalação e configuração de um servidor web Nginx para exibir uma página HTML personalizada. [cite: 10, 11, 40, 41, 42]
* [cite_start]Criação de um script Bash para monitorar a disponibilidade do site a cada minuto, com logs e envio de notificações via webhook (Discord/Telegram/Slack) em caso de indisponibilidade. [cite: 13, 14, 50, 51, 52, 53]
* [cite_start]Testes de implementação e documentação no GitHub. [cite: 16, 17, 61, 62, 63, 64, 65]

## 2. Etapa 1: Configuração do Ambiente na AWS

[cite_start]Nesta etapa, configuramos a infraestrutura de rede na AWS para nossa instância EC2. [cite: 6, 27]

### [cite_start]Criar VPC e Sub-redes [cite: 7, 29]

1.  No console da AWS, navegue até o serviço **VPC**.
2.  Clique em "VPCs" e depois em "Criar VPC".
3.  **Nome:** `minha-vpc-nginx`
4.  **Bloco CIDR IPv4:** `10.0.0.0/16`
5.  Clique em **Criar VPC**.
6.  [cite_start]Em "Sub-redes", clique em "Criar sub-rede" e crie 4 sub-redes: [cite: 7, 30, 31]
    * [cite_start]**publica-subnet-az1:** VPC `minha-vpc-nginx`, AZ `sa-east-1a`, CIDR `10.0.1.0/24` [cite: 30]
    * [cite_start]**publica-subnet-az2:** VPC `minha-vpc-nginx`, AZ `sa-east-1b`, CIDR `10.0.2.0/24` [cite: 30]
    * [cite_start]**privada-subnet-az1:** VPC `minha-vpc-nginx`, AZ `sa-east-1a`, CIDR `10.0.3.0/24` [cite: 31]
    * [cite_start]**privada-subnet-az2:** VPC `minha-vpc-nginx`, AZ `sa-east-1b`, CIDR `10.0.4.0/24` [cite: 31]

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

### [cite_start]Criar Instância EC2 [cite: 8, 33, 34, 35]

1.  No console da AWS, navegue até o serviço **EC2**.
2.  Clique em "Instâncias" e depois em "Executar instâncias".
3.  [cite_start]**Escolha uma AMI:** "Ubuntu Server 22.04 LTS (HVM), SSD Volume Type". [cite: 34]
4.  **Escolha o Tipo de Instância:** `t2.micro` (gratuito).
5.  **Configurar Detalhes da Instância:**
    * **Rede:** `minha-vpc-nginx`
    * [cite_start]**Sub-rede:** `publica-subnet-az1` (ou `publica-subnet-az2`). [cite: 35]
    * **Atribuir IP público automaticamente:** **Habilitar**.
6.  **Adicionar Armazenamento:** Padrão (8 GiB).
7.  **Adicionar Tags:** `Nome: NginxServer` (opcional).
8.  [cite_start]**Configurar Grupo de Segurança:** [cite: 36]

### [cite_start]Configurar Security Group [cite: 36]

1.  **Crie um novo Security Group:**
    * **Nome do grupo de segurança:** `nginx-sg`
    * **Descrição:** `Security Group para Nginx server`
    * [cite_start]**Regras de entrada (Inbound Rules):** [cite: 36]
        * **Tipo: SSH**, **Porta: 22**, **Fonte: Meu IP** (recomendado para SSH) ou `0.0.0.0/0`.
        * **Tipo: HTTP**, **Porta: 80**, **Fonte: 0.0.0.0/0** (para acesso público ao site).
    * **Regras de saída (Outbound Rules):**
        * Certifique-se de que há uma regra padrão: **Tipo: All traffic**, **Destino: 0.0.0.0/0**. (Isso é crucial para que a instância possa acessar a internet para atualizações e downloads).
2.  **Revisar e Executar:** Escolha ou crie um novo par de chaves (Key Pair) e faça o download do arquivo `.pem`. [cite_start]Guarde-o em segurança. [cite: 37]

### [cite_start]Acessar a Instância via SSH [cite: 37]

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

## [cite_start]3. Etapa 2: Configuração do Servidor Web (Nginx) [cite: 9]

[cite_start]Nesta etapa, instalamos e configuramos o Nginx para servir nossa página HTML. [cite: 10, 40]

### [cite_start]Instalar Nginx [cite: 10, 40]

1.  **Atualize os pacotes do sistema:**
    ```bash
    sudo apt update
    ```
2.  **Instale o Nginx:**
    ```bash
    sudo apt install nginx
    ```

### [cite_start]Criar Página HTML Personalizada [cite: 11, 41]

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
### [cite_start]Configurar Nginx para Servir a Página [cite: 42]

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

        server_name SEU_IP_PUBLICO_EC2; # Ex: 18.222.81.222, ou seu domínio (ex: example.ubuntu.com)

        root /var/www/teste;  # Caminho para sua página HTML
        index index.html;     # Nome do arquivo principal

        location / {
            try_files $uri $uri/ =404;
        }
    }
    ```
5.  **Reinicie o Nginx para aplicar as alterações:**
    ```bash
    sudo systemctl restart nginx
    ```
6.  **Verifique o status do Nginx:**
    ```bash
    sudo systemctl status nginx
    ```
    Deve mostrar `Active: active (running)`.

### [cite_start]Configurar Nginx para Reinício Automático (systemd) [cite: 45]

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

## [cite_start]4. Etapa 3: Monitoramento e Notificações [cite: 12]

[cite_start]Nesta etapa, criamos um script para monitorar o site e enviar alertas. [cite: 13, 48, 49]

### [cite_start]Criar Script de Monitoramento [cite: 13, 14, 50, 51]

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
      # Removendo -s -o /dev/null -w "%{http_code}" para ver erros mais claros se houver.
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
    * [cite_start]**ATENÇÃO:** Altere `SITE_URL` para o IP **fixo** da sua instância (Elastic IP) e a porta correta (80 ou 81). [cite: 52, 53]
    * [cite_start]**ATENÇÃO:** Substitua `SUA_WEBHOOK_DO_DISCORD_AQUI` pela URL real do seu webhook do Discord. [cite: 53, 57]

3.  **Dê permissão de execução ao script:**
    ```bash
    chmod +x /home/ubuntu/monitora_site.sh
    ```
4.  Instale a tarefa cron para rodar o script a cada minuto:
    ```bash
    /home/ubuntu/monitora_site.sh install-cron
    ```
## [cite_start]5. Etapa 4: Testes e Documentação [cite: 15, 16]

### [cite_start]Testes da Implementação [cite: 16, 61]

1.  **Acessibilidade do Site:**
    * Abra seu navegador e digite `http://SEU_IP_PUBLICO_OU_ELASTIC_IP` (ou com a porta se for diferente de 80).
    * [cite_start]Verifique se sua página HTML personalizada é exibida corretamente. [cite: 62]
2.  [cite_start]**Teste de Monitoramento (Parada do Nginx):** [cite: 63]
    * No terminal da sua instância EC2, pare o Nginx: `sudo systemctl stop nginx`.
    * Aguarde 1-2 minutos (o tempo do cron).
    * [cite_start]Verifique o `/var/log/monitoramento.log` para ver as mensagens de "ALERTA: Site INDISPONÍVEL!". [cite: 64]
    * [cite_start]Verifique se a notificação chegou no seu canal do Discord (ou Telegram/Slack). [cite: 64]
    * Inicie o Nginx novamente: `sudo systemctl start nginx`. O script deve registrar que o site voltou a ficar "DISPONÍVEL".

### [cite_start]Documentação no GitHub [cite: 17, 65]

Este arquivo `README.md` serve como a documentação do projeto, explicando:
* [cite_start]A configuração do ambiente AWS. [cite: 66]
* [cite_start]A instalação e configuração do servidor web Nginx. [cite: 67]
* [cite_start]O funcionamento do script de monitoramento. [cite: 68]
* [cite_start]Os testes e a validação da solução. [cite: 69]

---
