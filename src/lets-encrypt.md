# Генерация сертификата со включенным nginx

Создаем обычные сертификаты для домена по умолчанию

```
mkdir -p /etc/nginx/ssl
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/nginx/ssl/nginx.key -out /etc/nginx/ssl/nginx.crt
```

Настраиваем nginx:
`nano /etc/nginx/sites-available/default`

```
server {
	listen 80 default_server;
	listen [::]:80 default_server;
	server_name _;
	
	root /var/www/html;
	index index.html index.htm index.nginx-debian.html;
	
	location ~ /.well-known {
		allow all;
	}
}


server{
	listen 443 ssl default_server;
	root /var/www/html;
	
	ssl_certificate     /etc/nginx/ssl/nginx.crt;
	ssl_certificate_key /etc/nginx/ssl/nginx.key;
	
	location ~ /.well-known {
		allow all;
	}
}
```

Генерируем сертификат
```
letsencrypt certonly -a webroot --webroot-path=/var/www/html -d my_host --email my_email --text
```


# Генерация сертификата без nginx
```
letsencrypt certonly -d my_host --email my_email --text
```


# Генерирация dh4096.pem
```
openssl dhparam -out /etc/letsencrypt/live/my_host/dh4096.pem 4096
```


# Модификация файлов Nginx

Вставляем строчки в раздел http nginx.conf
```
	sendfile            on;
	tcp_nopush          on;
	tcp_nodelay         on;
	keepalive_timeout   70;
	types_hash_max_size 2048;
	
	
	ssl_session_cache   shared:SSL:10m;
	ssl_session_timeout 5m;
	ssl_protocols TLSv1 TLSv1.1 TLSv1.2; # Dropping SSLv3, ref: POODLE
	ssl_stapling on;
	
	# Cерверные шифры имеют больший приоритет, чем клиентские шифры
	ssl_prefer_server_ciphers on;
	
	# Разрешённые шифры
	ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:ECDHE-RSA-DES-CBC3-SHA:ECDHE-ECDSA-DES-CBC3-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA';
	#ssl_ciphers kEECDH+AES128:kEECDH:kEDH:-3DES:kRSA+AES128:kEDH+3DES:DES-CBC3-SHA:!RC4:!aNULL:!eNULL:!MD5:!EXPORT:!LOW:!SEED:!CAMELLIA:!IDEA:!PSK:!SRP:!SSLv2;
	#ssl_ciphers         HIGH:!aNULL:!MD5;
	
	
	# Разрешаем серверу для валидации сертификата прикреплять OCSP-ответы
	#ssl_stapling on;
	#ssl_stapling_verify on;
	#ssl_trusted_certificate /etc/pki/tls/certs/ca-bundle.crt;
	#resolver 8.8.8.8 [2001:4860:4860::8888];
	
	include             /etc/nginx/mime.types;
	default_type        application/octet-stream;
	
	client_max_body_size 256m;
	charset  utf-8;
	gzip  on;
	index   index.php index.html index.htm;
```


Для домена делаем:

```
server {
	listen 80;
	server_name my_host;
	rewrite ^(.*)$ https://$server_name$1 permanent;
}


server {
	listen       443 ssl;
	server_name  my_host;
	root         /var/www/html;
	
	ssl_certificate     /etc/letsencrypt/live/my_host/cert.pem;
	ssl_certificate_key /etc/letsencrypt/live/my_host/privkey.pem;
	ssl_dhparam /etc/letsencrypt/live/my_host/dh4096.pem;
	
	# Сайт доступен только по https
	#add_header Strict-Transport-Security 'max-age=604800';
	
	# Запретить показывать сайт во фрейме11
	add_header X-Frame-Options DENY;
	
	# Использовать отданный сервером Сontent-type, вместо автоматического его определения
	add_header X-Content-Type-Options nosniff;
	
	# Активировать XSS-защиту:
	add_header X-XSS-Protection "1; mode=block";	
	
	# http proxy to 10.0.0.1:80
	location / {
		proxy_pass                          http://10.0.0.1/;
		proxy_set_header Host               $host;
		proxy_set_header X-Real-IP          $remote_addr;
		proxy_set_header X-Forwarded-For    $remote_addr;
		proxy_set_header SERVER_PROTOCOL    $server_protocol;
		proxy_set_header SERVER_PORT        $server_port;
		proxy_set_header SERVER_NAME        $server_name;
		proxy_set_header REMOTE_USER        $remote_user;
		proxy_set_header REMOTE_PORT        $remote_port;
		proxy_set_header HTTPS              $https;
		proxy_set_header REQUEST_URI        $request_uri;
		proxy_set_header SCRIPT_NAME        /redmine;
		proxy_set_header QUERY_STRING       $query_string;
	}
```

# Обновление сертификата LetsEncrypt

```
letsencrypt renew -a webroot --webroot-path=/var/www/html
```

В Cron добавить строчку
```
23            03      1       *       *       letsencrypt renew -a webroot --webroot-path=/var/www/html && service nginx reload
```


# Материалы

1. [Nginx и https. Получаем класс А+](https://habrahabr.ru/post/252821/)
1. [Проверка качества защиты вашего сервера](https://www.ssllabs.com/ssltest/index.html)
1. [Создание самоподписанного сертификата](https://www.digitalocean.com/community/tutorials/how-to-create-an-ssl-certificate-on-nginx-for-ubuntu-14-04)
1. [How To Secure Nginx with Let's Encrypt on Ubuntu 16.04](https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-16-04)
