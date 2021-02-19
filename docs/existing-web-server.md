# How to integrate into an existing Web server setup

Since the non-dockerized version of BigBlueButton has [many requirements](https://docs.bigbluebutton.org/2.2/install.html#minimum-server-requirements), such as a specific Ubuntu version (16.04) as well as ports 80/443 not being in use by other applications, and considering that [a "clean" server dedicated for BigBlueButton is recommended](https://docs.bigbluebutton.org/2.2/install.html#before-you-install), you may enjoy the benefits of this dockerized version in order to run BigBlueButton on a server that is not completely dedicated to this software, on which a Web server may be already in use.

You could dedicate a virtual host to BigBlueButton, allowing external access to it through a reverse proxy.

> **Note.** The automatic HTTPS Proxy is not needed if you are going to run BigBlueButton behind a reverse proxy; in that case, you should be able to enable SSL for the virtual host you are going to dedicate to BigBlueButton, using your Web server features. Please notice that it will not be possible to install and use the integrated TURN server, since it requires the automatic HTTPS Proxy to be installed; therefore, if a TURN server is required, you should install and configure it by yourself. You can set BigBlueButton to use a TURN server by uncommenting and adjusting `TURN_SERVER` and `TURN_SECRET` in the `.env` file, which is created after completion of the setup script.

## Installation
1. Install BigBlueButton Docker [as explained above](#install). While running the setup script, please choose `n` when you're asked the following question: `Should an automatic HTTPS Proxy be included? (y/n)`.
2. Now all the required Docker containers should be running. BigBlueButton listens to port 8080. Create a virtual host by which BigBlueButton will be publicly accessible (in this case, let's assume the following server name for the virtual host: `bbb.example.com`). Enable SSL for the new _https_ virtual host. Make sure that the SSL certificate you will be using is signed by a CA (Certificate Authority). You could generate an SSL certificate for free using Let's Encrypt. It is suggested to add some directives to the _http_ virtual host `bbb.example.com` to redirect all requests to the _https_ one.

At this point, choose one of the following sections according to which Web server you're running ([Apache](#integration-with-apache)).

Eventually, BigBlueButton should be publicly accessible on `https://bbb.example.com/`. If you chose to install Greenlight, then the previous URL should allow you to open its home page. The APIs will be accessible through `https://bbb.example.com/bigbluebutton/`.

## Integration with nginx

1. Do not enable automatic HTTPS proxy during initial installation

    Answer `No` when `./script/setup` asks whether `Should an automatic HTTPS Proxy be included?`.
    Alternatively edit `.env` and set `ENABLE_HTTPS_PROXY` to false.

2. Make sure that the following Nginx modules are in use: `http_ssl`.

3. Obtain SSL certificate from <https://letsencrypt.org> in this example

    ```
    certbot certonly -d bigbluebutton.example.com -d bbb.example.com --email "sysop@example.com"
    ```
    Check for `cert.pem`, `chain.pem`, `fullchain.pem` and `privkey.pem` (by default) in /etc/letsencrypt/live/bigbluebutton.example.com directory.

4. include site-enabled in `nginx.conf`

    Add/enable line `include /etc/nginx/sites-enabled/*;` in your `http` section in `/etc/nginx/conf/nginx.conf`

5. Create nginx `bbb.example.com` virtual host:

    - set proxy to docker `127.0.0.1:8080`
    - explicitly redirect (301) all request towards `http://bbb.example.com` to `https://bbb.example.com`.

    Create/edit `/etc/nginx/site-available/bbb_example_com-ssl.conf`

   (Configuration snippet from `CentOS Stream release 8` / `nginx/1.14.1`)

	```
	upstream bigbluebutton-backend {
	   server 127.0.0.1:8080;
	   keepalive 32;
	}
	
	## Redirects all HTTP traffic to the HTTPS host
	server {
	  listen 0.0.0.0:80;
	  server_name    bigbluebutton.example.com bbb.example.com;
	
	  # Don't show the nginx version number, a security best practice
	  server_tokens off;   # Don't show the nginx version number, a security best practice
	
	  return 301 https://$http_host$request_uri;
	}
	
	server {
	  listen 0.0.0.0:443 ssl http2;
	  server_name    bigbluebutton.example.com bbb.example.com;
	
	  # Don't show the nginx version number, a security best practice
	  server_tokens off;   ## Don't show the nginx version number, a security best practice
	
	  # SSL certs.
	  ssl_certificate     /etc/letsencrypt/live/bigbluebutton.example.com/fullchain.pem;
	  ssl_certificate_key /etc/letsencrypt/live/bigbluebutton.example.com/privkey.pem;
	
	  ssl_session_timeout 1d;
	  ssl_protocols TLSv1.2;
	  ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256';
	  ssl_prefer_server_ciphers on;
	
	  # HSTS (ngx_http_headers_module is required) (15768000 seconds = 6 months)
	  add_header Strict-Transport-Security max-age=15768000;
	
	  # OCSP Stapling ---
	  # fetch OCSP records from URL in ssl_certificate and cache them
	  ssl_stapling on;
	  ssl_stapling_verify on;
	
	  # Individual nginx logs for this bigbluebutton vhost
	  access_log  /var/log/nginx/bigbluebutton/bigbluebutton_access.log;
	  error_log   /var/log/nginx/bigbluebutton/bigbluebutton_error.log;
	
	  location / {
       # WebSocket support
	    proxy_http_version 1.1;
	    proxy_set_header Upgrade $http_upgrade;
	    proxy_set_header Connection "Upgrade";
	    proxy_set_header Host $http_host;
	    proxy_set_header X-Real-IP $remote_addr;
	    proxy_set_header X-Forwarded-Ssl on;
	    proxy_set_header X-Forwarded-Host $host:$server_port;
	    proxy_set_header X-Forwarded-Server $host;
	    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	    proxy_set_header X-Forwarded-Proto $scheme;
	    proxy_set_header X-Frame-Options SAMEORIGIN;
	    proxy_buffers 256 16k;
	    proxy_buffer_size 16k;
	    proxy_read_timeout 600s;
	    client_max_body_size 150M;
	    proxy_pass http://bigbluebutton-backend;
	  }
	}
	```

6. Make sure logging directory exists and is writable by nginx

    ```shell
    mkdir /var/log/nginx/bigbluebutton && chown nginx:root /var/log/nginx/bigbluebutton
    ```

7. Enable nginx site

    ```shell
    ln -s /etc/nginx/conf/site-available/bbb_example_com-ssl.conf /etc/nginx/conf/site-enabled/
    ```

8. Nginx config files syntax check

    ```shell
    nginx -t
    ```


Apply changes:

1. Systemd

	```shell
	systemctl reload nginx && systemctl status -l nginx
	```

2. init System V
	
	```shell
	service reload nginx
	```

## Integration with Apache
1. Make sure that the following Apache modules are in use: `proxy`, `rewrite`, `proxy_http`, `proxy_wstunnel`. On _apache2_, the following command activates these modules,  whenever they are not already enabled:
```
sudo a2enmod proxy rewrite proxy_http proxy_wstunnel
```
2. Add the following directives to the _https_ virtual host `bbb.example.com`:
```
ProxyPreserveHost On

RewriteEngine On
RewriteCond %{HTTP:UPGRADE} ^WebSocket$ [NC,OR]
RewriteCond %{HTTP:CONNECTION} ^Upgrade$ [NC]
RewriteRule .* ws://127.0.0.1:8080%{REQUEST_URI} [P,QSA,L]

<Location />
	Require all granted
	ProxyPass http://127.0.0.1:8080/
	ProxyPassReverse http://127.0.0.1:8080/
</Location>
```
3. Restart Apache:
```
service apache2 restart
```
