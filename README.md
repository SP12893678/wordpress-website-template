wordpress web server set up guide
---
※確定環境已有docker、docker-compose
※由於資安問題，若無下方提及檔案，需自行建立
### 架設前須設定部分 

#### STEP1
修改`docker/.env`
```
MYSQL_ROOT_PASSWORD=你的mysql root密碼
MYSQL_USER=你的mysql使用者名稱
MYSQL_PASSWORD=你的mysql使用者密碼

HOST_EMAIL=你的網站email
HOST_DOMAIN=你的網域domain
WILDCARD_DOMAIN=你的萬用網域domain
```

##### 可選

`docker/config/uploads.ini` 為wordpress上傳設定
`nginx/server/fastcgi-cache.conf` 為快取設定
`nginx/server/static-files.conf` 為靜態資源快取設定
`nginx/gzip.conf ` 為gzip設定

#### STEP2

##### 若dns使用cloudflare
請前去申請個人的存取token (需擁有修改dns權限)
得到token後，再`nginx/ssl/cloudflare.ini`, 將token貼上your_token位置
```
dns_cloudflare_api_token = your_token
```

修改`docker/docker-composeLE.yml`，如下：
```yml
version: '3.1'

services:
    certbot:
        image: certbot/dns-cloudflare
        command: certonly --expand -d ${HOST_DOMAIN} -d ${WILDCARD_DOMAIN} --preferred-challenges dns --dns-cloudflare --dns-cloudflare-credentials /secrets/cloudflare.ini  --email ${HOST_EMAIL} --agree-tos --server https://acme-v02.api.letsencrypt.org/directory
        volumes:
        - ../nginx/certbot/conf:/etc/letsencrypt
        - ../nginx/certbot/logs:/var/log/letsencrypt
        - ../nginx/certbot/data:/var/www/certbot
        - ../nginx/ssl:/secrets

```

##### 若dns使用google cloud dns
請前去申請 cerbot專用的 憑證服務帳戶(需擁有修改dns權限)
建立完畢後，下載該服務帳戶的金鑰google.json，將內容直接複製到`nginx/ssl/google.json`

修改`docker/docker-composeLE.yml`，如下：
```yml
version: '3.1'

services:
    certbot:
        image: certbot/dns-google
        command: certonly --expand -d ${HOST_DOMAIN} -d ${WILDCARD_DOMAIN} --preferred-challenges dns --dns-google --dns-google-credentials /secrets/google.json --email ${HOST_EMAIL} --agree-tos --server https://acme-v02.api.letsencrypt.org/directory
        volumes:
        - ../nginx/certbot/conf:/etc/letsencrypt
        - ../nginx/certbot/logs:/var/log/letsencrypt
        - ../nginx/certbot/data:/var/www/certbot
        - ../nginx/ssl:/secrets

```

下載好的google.json格式如下：
```json
{
  "type": "...",
  "project_id": "...",
  "private_key_id": "...",
  "private_key": "...",
  "client_email": "...",
  "client_id": "...",
  "auth_uri": "...",
  "token_uri": "...",
  "auth_provider_x509_cert_url": "...",
  "client_x509_cert_url": "..."
}
```

#### STEP3
第一次架設請先進行狀況一方式

---
#### 狀況一
如果網域尚未有SSL憑證, 請先將`nginx/server-prod-step1.conf.example`內容複製到`nginx/server.conf`如下,(PS.記得更改server_name為你的網域)
```
server {
     listen [::]:80;
     listen 80;

     server_name your_domain.com;

     location ~ /.well-known/acme-challenge {
         allow all; 
         root /var/www/certbot;
     }
} 
```

上述皆設定完畢後，到docker資料夾開啟終端機，輸入下方指令：
PS.SSL憑證過程需等待1-2分鐘
```
docker-compose -f docker-composeLE.yml
```

若成功，再執行下方指令來運行自動憑證作業
```
docker-compose up -d
```


---
#### 狀況二
若網域已有SSL憑證, 但wordpress尚未使用多站點，請先將`nginx/server-prod-step2.conf.example`內容複製到`nginx/server.conf`如下, 接著使用`docker exec nginx nginx -s reload`, 將重新載入server.conf運行, 即可瀏覽網站

PS.記得更改server_name為你的網域,以及所有your_domain.com字眼都須更改
```
server {
    listen [::]:80;
    listen 80;

    server_name your_domain.com;

    location ~ /.well-known/acme-challenge {
        allow all; 
        root /var/www/certbot;
    }

    # redirect http to https www
    return 301 https://your_domain.com$request_uri;
}

server {
    listen [::]:443 ssl http2;
    listen 443 ssl http2;

    server_name your_domain.com;
    index index.php;
    root /var/www/html;
    
    # SSL code
    ssl_certificate /etc/nginx/ssl/live/your_domain.com/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/live/your_domain.com/privkey.pem;

    include conf.d/server/fastcgi-cache.conf;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ [^/]\.php(/|$) {
        fastcgi_split_path_info ^(.+?\.php)(/.*)$;
        fastcgi_pass wordpress:9000;
        fastcgi_index index.php;
        fastcgi_param  SCRIPT_FILENAME   $document_root$fastcgi_script_name;
        include fastcgi_params;

        if (!-f $document_root$fastcgi_script_name) {
            return 404;
        }
    }

    include conf.d/server/static-files.conf;
}
```

---
#### 狀況三
若網域已有SSL憑證, 且wordpress已使用多站點，則將`nginx/wordpress-subdomain.conf.example`內容複製到`nginx/server.conf`如下, 接著使用`docker exec nginx nginx -s reload`, 將重新載入server.conf運行,即可瀏覽網站

```
map $http_host $blogid {
    default       -999;
 
    #Ref: https://wordpress.org/extend/plugins/nginx-helper/
    #include /var/www/wordpress/wp-content/plugins/nginx-helper/map.conf ;
 
}
 
    
server {
    listen [::]:80;
    listen 80;

    server_name your_domain.com;

    location ~ /.well-known/acme-challenge {
        allow all; 
        root /var/www/certbot;
    }

    # redirect http to https www
    return 301 https://www.your_domain.com$request_uri;
}


server {
    listen [::]:80;
    listen 80;

    server_name *.your_domain.com;

    location ~ /.well-known/acme-challenge {
        allow all; 
        root /var/www/certbot;
    }

    # redirect http to https www
    return 301 https://$server_name$request_uri;
}

server {
    listen [::]:443 ssl http2;
    listen 443 ssl http2;

    server_name your_domain.com *.your_domain.com;
    index index.php;
    root /var/www/html;
    
    # SSL code
    ssl_certificate /etc/nginx/ssl/live/your_domain.com/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/live/your_domain.com/privkey.pem;

    include conf.d/server/fastcgi-cache.conf;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ [^/]\.php(/|$) {
        fastcgi_split_path_info ^(.+?\.php)(/.*)$;
        fastcgi_pass wordpress:9000;
        fastcgi_index index.php;
        fastcgi_param  SCRIPT_FILENAME   $document_root$fastcgi_script_name;
        include fastcgi_params;

        if (!-f $document_root$fastcgi_script_name) {
            return 404;
        }
    }

    include conf.d/server/static-files.conf;

    #WPMU Files
        location ~ ^/files/(.*)$ {
                try_files /wp-content/blogs.dir/$blogid/$uri /wp-includes/ms-files.php?file=$1 ;
                access_log off; log_not_found off;      expires max;
        }
 
    #WPMU x-sendfile to avoid php readfile()
    location ^~ /blogs.dir {
        internal;
        alias /var/www/example.com/htdocs/wp-content/blogs.dir;
        access_log off;     log_not_found off;      expires max;
    }
 
    #add some rules for static content expiry-headers here
}
```

