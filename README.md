# ProjetoLinuxCompass

Instalação e configuração servidor web (Nginx):
sudo apt update
sudo apt install nginx
cd /var/www
sudo mkdir teste
cd teste
sudo touch index.html
sudo vi index.html
cd /etc/nginx/sites-enabled
sudo "${EDITOR:-vi}" teste
server {
       listen 81;
       listen [::]:81;

       server_name example.ubuntu.com;

       root /var/www/teste;
       index index.html;

       location / {
               try_files $uri $uri/ =404;
       }
}
sudo service nginx restart
sudo systemctl edit nginx
[Service]
Restart=always
RestartSec=3
sudo systemctl daemon-reexec
sudo systemctl restart nginx

cd /home/ubuntu
touch monitora_site.sh
nano monitora_site.sh

#!/bin/bash

# --- VARIÁVEIS DE CONFIGURAÇÃO ---
SITE_URL="http://18.222.81.222:81" # Altere para o seu domínio, IP e porta. Ex: "http://seusite.com:81"
LOG_FILE="/var/log/monitoramento.log"
DISCORD_WEBHOOK_URL="SUA_WEBHOOK_DO_DISCORD_AQUI" # Substitua pela sua URL do webhook do Discord/Telegram/Slack.

# --- FUNÇÕES ---

# Função para enviar notificação (adaptável para Discord, Telegram, Slack)
send_notification() {
  MESSAGE="$1"
  /usr/bin/curl -H "Content-Type: application/json" -X POST -d "{\"content\":\"$MESSAGE\"}" "$DISCORD_WEBHOOK_URL"
}

# Função para instalar a tarefa no crontab
install_cron_job() {
  if ! /usr/bin/crontab -l 2>/dev/null | /usr/bin/grep -q "/home/ubuntu/monitora_site.sh"; then
    (/usr/bin/crontab -l 2>/dev/null; echo "* * * * * /bin/bash /home/ubuntu/monitora_site.sh >> /dev/null 2>&1") | /usr/bin/crontab -
    echo "Tarefa cron adicionada para 'monitora_site.sh' em /home/ubuntu/."
  else
    echo "Tarefa cron para 'monitora_site.sh' já existe."
  fi
}

# --- LÓGICA PRINCIPAL DO SCRIPT ---

# Garante que o arquivo de log exista e com as permissões corretas (para o usuário root que executa o cron)
# Este comando 'touch' pode falhar se as permissões do diretório /var/log/ forem muito restritivas,
# mas 'root' geralmente tem acesso.
/usr/bin/touch "$LOG_FILE"

if [[ "$1" == "install-cron" ]]; then
  install_cron_job
  exit 0
fi

if /usr/bin/curl -s -o /dev/null -w "%{http_code}" "$SITE_URL" | /usr/bin/grep -q "200"; then
  STATUS="DISPONÍVEL"
  echo "$(date): Site $SITE_URL está $STATUS." | /usr/bin/tee -a "$LOG_FILE"
else
  STATUS="INDISPONÍVEL"
  MESSAGE="ALERTA: Site $SITE_URL está $STATUS!"
  echo "$(date): $MESSAGE" | /usr/bin/tee -a "$LOG_FILE"
  send_notification "$MESSAGE"
fi

chmod +x /home/ubuntu/monitora_site.sh
/home/ubuntu/monitora_site.sh install-cron
