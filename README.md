# ipbforum-nginx-conf

Nginx Configuration


Now that you have both Nginx and PHP-FPM up and running, we can move on to configuring the web server. First, let's make some small adjustments to /etc/nginx/nginx.conf,

user  nginx;
worker_processes  4;
 
error_log  /var/log/nginx/error.log error;
pid        /var/run/Nginx.pid;
 
 
events {
    worker_connections  1024;
}
 
 
http {
    include       mime.types;
    default_type  application/octet-stream;
    server_tokens off;
 
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
 
    access_log  /var/log/nginx/access.log  main;
 
    sendfile        on;
    #tcp_nopush     on;
 
    keepalive_timeout  30;
 
    #gzip  on;
 
    include conf.d/*.conf;
}

There's not much to say here. Increase worker_processes from the default of 1 to however many processor cores your server has. For example, if your server has a single quad core processor, set this value to 4. I've also changed the error_log directive to only log errors.

Now let's move on to configuring your IP.Board website. Use this as your base template for /etc/nginx/conf.d/ipboard.conf,

server {
    listen       80;
    server_name  yourdomain.com www.yourdomain.com;
    root         /srv/http/yourdomain.com/root;
 
    # Basic web server configuration.
    index        index.php
    #access_log   off;
    client_max_body_size  1G;
 
    # GZIP static content not processed by IPB.
    gzip  on;
    gzip_static on;
    gzip_http_version 1.1;
    gzip_vary on;
    gzip_comp_level 6;
    gzip_proxied any;
    gzip_types text/plain text/css application/json application/x-javascript application/xml application/xml+rss text/javascript application/javascript text/x-js;
    gzip_buffers 16 8k;
    gzip_disable "MSIE [1-6].(?!.*SV1)";
 
    # Set up rewrite rules.
    location / {
        try_files  $uri $uri/ /index.php;
    }
 
    # Stub and FPM Status
    location /server_status {
        stub_status on;
        allow 127.0.0.1;
        deny all;
    }
    location /fpm_status {
        allow 127.0.0.1;
        deny all;
        fastcgi_pass   unix:/var/run/php-fpm/ipboard.sock;
        fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
        include        /etc/nginx/fastcgi_params;
    }
 
    # Deny access to hidden files
    location ~ /. {
        deny  all;
    }
 
    # Mask fake admin directory
    location ~^/admin/(.*)$ {
        deny     all;
    }
 
    # Secure real admin directory
    location ~^(/nimda/).*(.php) {
        #allow         127.0.0.1;
        #deny          all;
        #auth_basic    "Restricted Area";
        #auth_basic_user_file $document_root/nimda/.htpasswd;
        fastcgi_pass   unix:/var/run/php-fpm/ipboard.sock;
        fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
        include        /etc/nginx/fastcgi_params;
    }
 
    # IP.Board PHP/CGI Protection
    location ~^(/uploads/).*(.php)$ {
        deny     all;
    }
    location ~^(/hooks/).*(.php)$ {
        deny     all;
    }
    location ~^(/cache/).*(.php)$ {
        deny     all;
    }
    location ~^(/screenshots/).*(.php)$ {
        deny     all;
    }
    location ~^(/downloads/).*(.php)$ {
        deny     all;
    }
    location ~^(/blog/).*(.php)$ {
        deny     all;
    }
    location ~^(/public/style_).*(.php)$ {
        deny     all;
    }
 
    # Caching directives for static files.
    location ~^(/uploads/profile/).*.(jpg|jpeg|gif|png)$ {
        access_log off;
        expires    1d;
    }
    location ~* ^.+.(jpg|jpeg|gif|css|png|js|ico|xml|htm|txt|swf|cur)$ {
        access_log off;
        expires    1w;
    }
 
    # Pass PHP scripts to php-fpm
    location ~ .php$ {
        fastcgi_pass   unix:/var/run/php-fpm/ipboard.sock;
        fastcgi_index  index.php;
        fastcgi_buffers 16 8k;
        fastcgi_buffer_size 16k;
        fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
        include        /etc/nginx/fastcgi_params;
    }
}
