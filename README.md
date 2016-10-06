## Configurar vhost en nginx con https.

### Crear un archivo en `/etc/nginx/sites-available/[PROJECT_NAME]`

```
server {
    listen 80;
    listen [::]:80;
    server_name [HOST_NAME];
    server_tokens off;

    root /home/deploy/[PROJECT_NAME]_[BRANCH_NAME];
    index index.php index.html;

    access_log  /var/log/nginx/[PROJECT_NAME]_access.log;
    error_log   /var/log/nginx/[PROJECT_NAME]_error.log;

    location / {
        return 301 https://$http_host$request_uri;
    }

    include /etc/nginx/letsencrypt.conf;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name [HOST_NAME];
    server_tokens off;

    root /home/deploy/[PROJECT_NAME]_[BRANCH_NAME];
    index index.php index.html;

    # comment these two lines until a certificate has been created
    ssl_certificate /etc/letsencrypt/live/[HOST_NAME]/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/[HOST_NAME]/privkey.pem;

    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_dhparam /etc/ssl/certs/dhparam.pem;

    access_log  /var/log/nginx/[PROJECT_NAME]_access.log;
    error_log   /var/log/nginx/[PROJECT_NAME]_error.log;

    location = /favicon.ico {
        log_not_found off;
        access_log off;
    }

    location = /robots.txt {
        allow all;
        log_not_found off;
        access_log off;
    }

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        try_files $uri /index.php =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass unix:/var/run/php5-fpm.sock;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~* \.(js|css|png|jpg|jpeg|gif|ico)$ {
        expires max;
        log_not_found off;
    }

    gzip              on;
    gzip_disable      "msie6";
    gzip_buffers      16 8k;
    gzip_comp_level   4;
    gzip_http_version 1.0;
    gzip_min_length   1280;
    gzip_types        text/plain text/css application/x-javascript text/xml application/xml application/xml+rss text/javascript image/x-icon image/bmp;
    gzip_vary         on;
}
```

### Crear un enlace simbolico
``` sh
ln -s /etc/nginx/sites-available/[PROJECT_NAME] /etc/nginx/sites-enables/[PROJECT_NAME]
```

### Reiniciar nginx

``` sh
service nginx reload
```

### Configurar certificado con certbot

Crear un archivo en /root/certbot-config/[PROJECT_NAME].ini y cambiar `example.com` por el o los dominios a crear certificados

```
# use the webroot authenticator. 
authenticator = webroot
# the following path needs to be served by our webserver
# to validate our domains
webroot-path = /var/www/letsencrypt

# generate certificates for the specified domains.
domains = example.com, www.example.com

# register certs with the following email address
email = ingeniouskey@gmail.com

# use a 4096 bit RSA key instead of 2048
rsa-key-size = 4096
```

### Agregar una línea nueva a `/etc/cron.weekly/renew-ssl-certificates`

``` sh
/root/certbot-auto certonly -c /root/certbot-config/[PROJECT_NAME].ini --renew-by-default
```

### Crear el certificado por primera vez:

``` sh
/root/certbot-auto certonly -c /root/certbot-config/[PROJECT_NAME].ini
```

### Pasos finales.

* Descomentar las ĺineas de `ssl_certificate` y `ssl_certificate_key` del archivo `/etc/nginx/sites-available/[PROJECT_NAME]`
* Reiniciar nginx: `service nginx reload`
