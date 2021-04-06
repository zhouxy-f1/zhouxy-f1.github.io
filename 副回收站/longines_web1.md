```nginx
server {
    listen 80;
    server_name longines.cn;

    # [#857202 ] Can you forward Customer IP in access log ?
    real_ip_header X-Cluster-Client-Ip;
    set_real_ip_from 192.168.0.0/16;

    access_log logs/longines.cn.www.access.log main;
    error_log logs/longines.cn.www.error.log;

    more_set_headers "X-FSIN: 1";
    more_set_headers "X-FWK: M";
    more_set_headers "Strict-Transport-Security: max-age=300";
    #more_set_headers "Strict-Transport-Security: max-age=31536000";
    more_set_headers "X-Content-Type-Options: nosniff";
    more_set_headers "X-Frame-Options: SAMEORIGIN";
    more_set_headers "X-XSS-Protection: 1; mode=block";
    more_set_headers "Content-Security-Policy: default-src 'self' 'unsafe-inline' 'unsafe-eval'; img-src 'self' 'unsa
fe-inline' 'unsafe-eval' data: blob: https:; script-src 'self' 'unsafe-inline' 'unsafe-eval' https:; connect-src 'sel
f' 'unsafe-inline' 'unsafe-eval' https:; frame-src 'self' 'unsafe-inline' 'unsafe-eval' https: http:; media-src 'self
' 'unsafe-inline' 'unsafe-eval' https: http:";
    set_cookie_flag * secure;

    return 301 https://www.$server_name$request_uri;
}
server {
    listen 80;
    server_name longines.cn;

    # [#857202 ] Can you forward Customer IP in access log ?
    real_ip_header X-Cluster-Client-Ip;
    set_real_ip_from 192.168.0.0/16;

    access_log logs/longines.cn.www.access.log main;
    error_log logs/longines.cn.www.error.log;

    more_set_headers "X-FSIN: 1";
    more_set_headers "X-FWK: M";
    more_set_headers "Strict-Transport-Security: max-age=300";
    #more_set_headers "Strict-Transport-Security: max-age=31536000";
    more_set_headers "X-Content-Type-Options: nosniff";
    more_set_headers "X-Frame-Options: SAMEORIGIN";
    more_set_headers "X-XSS-Protection: 1; mode=block";
    set_cookie_flag * secure;


#1086180
    limit_req zone=one2 burst=10 nodelay;


    listen 80;
    server_name www.longines.cn staging.longines.cn;

    #proxy_ignore_client_abort on;
    # [#857202 ] Can you forward Customer IP in access log ?
    real_ip_header X-Cluster-Client-Ip;
    set_real_ip_from 192.168.0.0/16;

    access_log logs/longines.cn.www.access.log main;
    access_log logs/longines.cn.www.weblogmetric.log weblogmetric;
    error_log logs/longines.cn.www.error.log;

    set $MAGE_ROOT /var/www/sites/longines.cn/magento/www2;
    set $MAGE_RUN_CODE china_cn;
    set $MAGE_RUN_TYPE store;
    #add_header hostname web1 always;
    more_set_headers "X-FSIN: 1";
    more_set_headers "X-FWK: M";
    more_set_headers "Strict-Transport-Security: max-age=300";
    #more_set_headers "Strict-Transport-Security: max-age=31536000";
    more_set_headers "X-Content-Type-Options: nosniff";
    more_set_headers "X-Frame-Options: SAMEORIGIN";
    more_set_headers "X-XSS-Protection: 1; mode=block";
    set_cookie_flag * secure;


    #include conf.d/cache.cache;

    root $MAGE_ROOT/pub;
    index index.html index.php;

    error_page 404 403 = /errors/404.php;
    error_page 503 = /errors/503.php;

    #rewrite ^/(.*)/$ /$1 permanent;

    location / {
        #rewrite ^/checkout/cart/$ /checkout/cart permanent;
        #rewrite ^/customer/account/login/$ /customer/account/login permanent;
        location = /maintenance.html {
            return 503;
        }

        location ^~ /admin {
            deny all;
        }

        location ~* \.(woff|woff2|eot|ttf|svg|mp4|webm|jpg|jpeg|png|gif|ico|css|js)$ {
            expires 365d;
            break;
        }
        #location ~ ^/(watches(/.*)?$|watch-) {
        #    expires 1h;
        #    try_files @dummy @handler;
        #}

        try_files $uri $uri/ @handler;

        location ~\.php {
            if ($request_uri ~ "^/index\.php(/.*)?$") {
                return 301 /;
            }
            try_files @dummy @handler;
        }

        location /static/ {
            location ~ ^/static/version {
                rewrite ^/static/(version\d*/)?(.*)$ /static/$2 last;
            }
            if (!-f $request_filename) {
                rewrite ^/static/(version\d*/)?(.*)$ /static.php?resource=$2 last;
            }
        }

        #location /admin/ {
        #       deny all;
        #}

        location /media/ {
            try_files $uri $uri/ @handler;
            location ~ ^/media/(customer|downloadable|import)/ {
                deny all;
            }
            location ~ ^/media/theme_customization/.*\.xml {
                deny all;
            }
            location ~ ^/media/.*\.php {
                deny all;
            }
            if (!-f $request_filename) {
                rewrite / /get.php last;
            }
        }

        location ~ /\. {
            if (-f $request_filename) {
                return 403;
            }
            if (-d $request_filename) {
                return 403;
            }
            try_files @dummy @handler;
        }
    }

    location @handler {
        try_files $uri /index.php =404;
        fastcgi_pass 127.0.0.1:9001;
        fastcgi_buffers 1024 16k;
        fastcgi_buffer_size 64k;
        fastcgi_param MAGE_MODE production;
        fastcgi_param MAGE_RUN_CODE $MAGE_RUN_CODE;
        fastcgi_param MAGE_RUN_TYPE $MAGE_RUN_TYPE;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        fastcgi_param QUERY_STRING $query_string;
        fastcgi_param REQUEST_METHOD $request_method;
        fastcgi_param CONTENT_TYPE $content_type;
        fastcgi_param CONTENT_LENGTH $content_length;
        fastcgi_param SCRIPT_NAME $fastcgi_script_name;
        fastcgi_param REQUEST_URI $request_uri;
        fastcgi_param DOCUMENT_URI $document_uri;
        fastcgi_param DOCUMENT_ROOT $realpath_root;
        fastcgi_param SERVER_PROTOCOL $server_protocol;
        fastcgi_param REQUEST_SCHEME $scheme;
        #fastcgi_param HTTPS $https if_not_empty;
        fastcgi_param HTTPS on;
        fastcgi_param GATEWAY_INTERFACE CGI/1.1;
        fastcgi_param SERVER_SOFTWARE "";
        fastcgi_param REMOTE_ADDR $remote_addr;
        fastcgi_param REMOTE_PORT $remote_port;
        fastcgi_param SERVER_ADDR $server_addr;
        fastcgi_param SERVER_PORT $server_port;
        fastcgi_param SERVER_NAME $server_name;
        fastcgi_param REDIRECT_STATUS 200;
    }

#    location = / {
#        try_files @dummy @zendhandler;
#    }
#
#        root /var/www/sites/longines.cn/zend/www/site/www;
#        try_files $uri $uri/ @zendhandler;
#    }
#
#    location = /ggle-translate/translations.php {
#        try_files @dummy @zendhandler;
#    }
#
#    location @zendhandler {
#        more_set_headers "X-FWK: Z";
#        root /var/www/sites/longines.cn/zend/www/site/www;
#        try_files $uri /index.php =404;
#        fastcgi_pass 127.0.0.1:9001;
#        fastcgi_buffers 16 16k;
#        fastcgi_buffer_size 32k;
#        fastcgi_param APPLICATION_PATH $realpath_root;
#        fastcgi_param APPLICATION_ENV production;
#        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
#        fastcgi_param QUERY_STRING $query_string;
#        fastcgi_param REQUEST_METHOD $request_method;
#        fastcgi_param CONTENT_TYPE $content_type;
#        fastcgi_param CONTENT_LENGTH $content_length;
#        fastcgi_param SCRIPT_NAME $fastcgi_script_name;
#        fastcgi_param REQUEST_URI $request_uri;
#        fastcgi_param DOCUMENT_URI $document_uri;
#        fastcgi_param DOCUMENT_ROOT $realpath_root;
#        fastcgi_param SERVER_PROTOCOL $server_protocol;
#        fastcgi_param REQUEST_SCHEME $scheme;
#        #fastcgi_param HTTPS $https if_not_empty;
#        fastcgi_param HTTPS on;
#        fastcgi_param GATEWAY_INTERFACE CGI/1.1;
#        fastcgi_param SERVER_SOFTWARE "";
#        fastcgi_param REMOTE_ADDR $remote_addr;
#        fastcgi_param REMOTE_PORT $remote_port;
#        fastcgi_param SERVER_ADDR $server_addr;
#        fastcgi_param SERVER_PORT $server_port;
#        fastcgi_param SERVER_NAME $server_name;
#        fastcgi_param REDIRECT_STATUS 200;
#    }
#
    # 301 redirects
    location ~ "^/watches/the-longines-equestrian-collection/l6-130-0-89-2$" {
           return 301 /watches/equestrian/equestrian-collection;
    }
    location ~ "^/watches/dolcevita/(l\d-\d{3}-\d-\d{2}-\d)$" {
        return 301 /watch-longines-dolcevita-$1;
    }
    location ~ "^/watches/longines-dolcevita/(l\d-\d{3}-\d-\d{2}-\d)$" {
        return 301 /watch-longines-dolcevita-$1;
    }
    location ~ "^/watches/primaluna/(l\d-\d{3}-\d-\d{2}-\d)$" {
        return 301 /watch-longines-primaluna-$1;
    }
    location ~ "^/watches/longines-symphonette/(l\d-\d{3}-\d-\d{2}-\d)$" {
        return 301 /watch-longines-symphonette-$1;
    }
    location ~ "^/watches/grande-classique/(l\d-\d{3}-\d-\d{2}-\d)$" {
        return 301 /watch-la-grande-classique-de-longines-$1;
    }
    location ~ "^/watches/presence/(l\d-\d{3}-\d-\d{2}-\d)$" {
        return 301 /watch-presence-$1;
    }
    location ~ "^/watches/flagship/(l\d-\d{3}-\d-\d{2}-\d)$" {
        return 301 /watch-flagship-$1;
    }
    location ~ "^/watches/longines-lyre/(l\d-\d{3}-\d-\d{2}-\d)$" {
        return 301 /watch-longines-lyre-$1;
    }
    location ~ "^/watches/master-collection/(l\d-\d{3}-\d-\d{2}-\d)$" {
        return 301 /watch-the-longines-master-collection-$1;
    }
    location ~ "^/watches/the-longines-master-collection/(l\d-\d{3}-\d-\d{2}-\d)$" {
        return 301 /watch-the-longines-master-collection-$1;
    }
    location ~ "^/watches/conquest-classic/(l\d-\d{3}-\d-\d{2}-\d)$" {
        return 301 /watch-conquest-classic-$1;
    }
    location ~ "^/watches/st-imier-collection/(l\d-\d{3}-\d-\d{2}-\d)$" {
        return 301 /watch-the-longines-saint-imier-collection-$1;
    }
    location ~ "^/watches/elegant-collection/(l\d-\d{3}-\d-\d{2}-\d)$" {
        return 301 /watch-the-longines-elegant-collection-$1;
    }
    location ~ "^/watches/the-longines-elegant-collection/(l\d-\d{3}-\d-\d{2}-\d)$" {
        return 301 /watch-the-longines-elegant-collection-$1;
    }
    location ~ "^/watches/evidenza/(l\d-\d{3}-\d-\d{2}-\d)$" {
        return 301 /watch-longines-evidenza-$1;
    }
    location ~ "^/watches/the-longines-equestrian-collection/(l\d-\d{3}-\d-\d{2}-\d)$" {
        return 301 /watch-equestrian-$1;
    }
    location ~ "^/watches/hydroconquest/(l\d-\d{3}-\d-\d{2}-\d)$" {
        return 301 /watch-hydroconquest-$1;
    }
    location ~ "^/watches/conquest/(l\d-\d{3}-\d-\d{2}-\d)$" {
        return 301 /watch-conquest-$1;
    }
    location ~ "^/watches/heritage-collection/(l\d-\d{3}-\d-\d{2}-\d)$" {
        return 301 /watch-heritage-$1;
    }


    ### Redirection produits inexistants ###

    #location ~ "^/watch-the-longines-master-collection-l2-893-4-92-6$" {
    #    return 301 /watches/watchmaking-tradition/master-collection;
    #}

    ###
}
```

