# Config_nginx_reverse-proxy_modsecurity

### Buoc 1 : install nginx, mở ufw cho nginx(enable ufw) ufw status
### Buoc 2 : checking webserver systemctl status nginx (systemctl restart nginx)
### Buoc 3 : check lại bằng curl -i localhost:80 (22 - ssh, 443 - https) 
### Buoc 4 : install modules modsecurity on Ubuntu  :

    apt-get install -y git build-essential libpcre3 libpcre3-dev libssl-dev libtool autoconf apache2-dev libxml2-dev libcurl4-openssl-dev automake pkgconf
    

### Buoc 5 : clone gói từ github(gói này trên offical) : git clone -b nginx_refactoring https://github.com/SpiderLabs/ModSecurity.git

    cd ModSecurity
    ./autogen.sh
    ./configure --enable-standalone-module --disable-mlogc
    make

### Buoc 6 : Compile Nginx

    cd /usr/src
    wget https://nginx.org/download/nginx-1.10.3.tar.gz
    tar -zxvf nginx-1.10.3.tar.gz && rm -f nginx-1.10.3.tar.gz

    cd nginx-1.10.3/
    ./configure --user=nginx --group=nginx --add-module=/usr/src/ModSecurity/nginx/modsecurity --with-http_ssl_module
    make
    make install

    sed -i "s/#user  nobody;/user www-data www-data;/" /usr/local/nginx/conf/nginx.conf


    #### user default của ubutun là www-data nhá các bác :v 
    - Lúc này xong thì các file conf của nginx sẽ chuyển sang thử  mục /usr/local /nginx

        nginx path prefix: "/usr/local/nginx"
        nginx binary file: "/usr/local/nginx/sbin/nginx"
        nginx modules path: "/usr/local/nginx/modules"
        nginx configuration prefix: "/usr/local/nginx/conf"
        nginx configuration file: "/usr/local/nginx/conf/nginx.conf"
        nginx pid file: "/usr/local/nginx/logs/nginx.pid"
        nginx error log file: "/usr/local/nginx/logs/error.log"
        nginx http access log file: "/usr/local/nginx/logs/access.log"
        nginx http client request body temporary files: "client_body_temp"
        nginx http proxy temporary files: "proxy_temp"
        nginx http fastcgi temporary files: "fastcgi_temp"
        nginx http uwsgi temporary files: "uwsgi_temp"
        nginx http scgi temporary files: "scgi_temp"

    - you can test the installation with:

        /usr/local/nginx/sbin/nginx -t


    For your convenience, you can setup a systemd unit file for Nginx:


        cat <<EOF>> /lib/systemd/system/nginx.service
        [Service]
        Type=forking
        ExecStartPre=/usr/local/nginx/sbin/nginx -t -c /usr/local/nginx/conf/nginx.conf
        ExecStart=/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
        ExecReload=/usr/local/nginx/sbin/nginx -s reload
        KillStop=/usr/local/nginx/sbin/nginx -s stop

        KillMode=process
        Restart=on-failure
        RestartSec=42s

        PrivateTmp=true
        LimitNOFILE=200000

        [Install]
        WantedBy=multi-user.target
        EOF


    - Moving forward, you can start/stop/restart Nginx as follows:

        systemctl start nginx.service
        systemctl stop nginx.service
        systemctl restart nginx.service


    bugs : can fix here https://serverfault.com/questions/913055/removing-nginx-compilation

### Buoc 7 : Configure ModSecurity and Nginx

    - vi /usr/local/nginx/conf/nginx.conf :

        location / {
            ModSecurityEnabled on;
            ModSecurityConfig modsec_includes.conf;
            #proxy_pass http://localhost:8011;
            #proxy_read_timeout 180s;
            # nếu bỏ 2 cmt trên thì nó sẽ trở thành 1 reverse proxy
            root   html;
            index  index.html index.htm;
        }


    - vi /usr/local/nginx/conf/modsec_includes.conf : 

        include modsecurity.conf
        include owasp-modsecurity-crs/crs-setup.conf  // apply selective rules only
        include owasp-modsecurity-crs/rules/*.conf    // apply all of the OWASP ModSecurity Core Rules in the owasp-modsecurity-crs/rules/ directory


    - Import ModSecurity configuration files:

        cp /usr/src/ModSecurity/modsecurity.conf-recommended /usr/local/nginx/conf/modsecurity.conf
        cp /usr/src/ModSecurity/unicode.mapping /usr/local/nginx/conf/


    - Modify the /usr/local/nginx/conf/modsecurity.conf file:

        sed -i "s/SecRuleEngine DetectionOnly/SecRuleEngine On/" /usr/local/nginx/conf/modsecurity.conf


### Buoc 8 : Add OWASP ModSecurity CRS (Core Rule Set) files:


        cd /usr/local/nginx/conf
        git clone https://github.com/SpiderLabs/owasp-modsecurity-crs.git
        cd owasp-modsecurity-crs
        mv crs-setup.conf.example crs-setup.conf
        cd rules
        mv REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf.example REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf
        mv RESPONSE-999-EXCLUSION-RULES-AFTER-CRS.conf.example RESPONSE-999-EXCLUSION-RULES-AFTER-CRS.conf


### Buoc 9 : opne firewall

        ufw allow OpenSSH
        ufw allow 80
        ufw default deny
        ufw enable 


## Test modsecurity: 

    http://localhost/?param="><script>alert(1);</script>"

    check error logs :``` cat /usr/local/nginx/logs/error.log ``` 

    Config rules modsecurity : 
    /usr/local/nginx/conf/modsecurity.conf and /usr/local/nginx/conf/owasp-modsecurity-crs/crs-setup.conf
