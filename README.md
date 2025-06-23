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
