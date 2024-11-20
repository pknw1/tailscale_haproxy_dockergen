
## auto-updating HAPRoxy reverse proxy (with tailscale)

- start full stack or cut down to core proxy service
- configured to operate using tailscale but can be run without it
- auto-populates HAProxy backends for defined containers
- configure with a single wildcard SSL cert

This configuration, based on jwilder/nginx-proxy creates a self contained HAProxy reverse proxy
Once started, the stack monitors for containers with the environment variable TAILSCALE_HOST set and adds them as a backend in HAProxy

![alt text](https://github.com/pknw1/tailscale_haproxy_dockergen/blob/main/tailscale.png?raw=true)

### Quickstart
```
git clone https://github.com/pknw1/haproxy-autogen
cd haproxy-autogen
cp env-example .env
cp  config/dockergen/haproxy-example.tmpl config/dockergen/haproxy.tmpl
cp  config/dockergen/www-example.tmpl config/dockergen/www.tmpl

1. update settings in env
2. update haproxy template to set your domain (global search and replace yourdomain.co.uk)
3. update haproxy template with the correct wildcard certificate PEM path for the secure frontend
4. customise or remove unwanted services from docker compose file
5. docker compose up -d && docker compose logs -f to start

update your DNS A record IP to the Tailscale WAN IP Address
```


<details>
  <summary>Detailed setup steps</summary>
  
1. Clone the repository
```
git clone https://github.com/pknw1/haproxy-autogen
cd haproxy-autogen
```

2. Modify the example config

- [ ] Copy env-example to .env
        ```
        cp env-example .env
        ```
- [ ] Update values in env file
    - [ ] Obtain API Key from [TailScale](https://login.tailscale.com/admin/settings/keys)
    - [ ] Set the HAPROXY_IP from your tailscale_network range 
    - [ ] Set the tailscale network name
    - [ ] Set the domain assigned to your Tailscale Network (tailscale.yourdomain.com)
    
        ```
        TS_AUTHKEY= ** create a key at https://login.tailscale.com/admin/settings/keys **
        HAPROXY_IP=172.32.0.200
        TAILSCALE_NETWORK=tailscale_haproxy
        TAILSCALE_DOMAIN=tailscale.yourdomain.co.uk
        ```
- [ ] Create config/dockergen/haproxy.tmpl from the example
    - [ ] global search and replace ".yourdomain.co.uk" as required
        ```
        sed -i 's/.yourdomain.co.uk/youdetails/g' config/dockergen/haproxy.tmpl
        ```
    - [ ] update the path to your wildcard certificate for *.tailscale.yourdomain.co.uk
        ```
        vi config/dockergen/haproxy.tmpl
        
        frontend secure 
          bind *:443 ssl crt /etc/ssl/private/tailscale-yourdomain.co.uk.pem 
          tcp-request inspect-delay 15s
          errorfiles httperrors
          mode http
          option forwardfor

        ```
    
- [ ] Create config/dockergen/www.tmpl from the example
          ```
          cp config/dockergen/www-example.tmpl config/dockergen/www.tmpl
          ```

- [ ] modify the config/dockergen/docker-gen.cfg
    - [ ] update paths to templates and outputs if changed from defaults
    
3. starts with docker compose up -d
```
docker compose up -d
```


The compose up command will start the stack as follows

 - create the network
 - start the tailscale VPN [container](https://tailscale.com/kb/1282/docker) and register
 - start [Dozzle](https://dozzle.dev/)
 - start [Glances](https://github.com/nicolargo/glances)
 - Start HAProxy Web UI
 - Start NGINX
 - Start WhoAmI
 - start the [dockergen](https://github.com/nginx-proxy/docker-gen) container 
 - start the HAPRoxy container
 
 Any container started with environment variable TAILSCALE_HOST set will dynamically add or 
 remove from the HA Proxy config and then restart the HA Proxy Server
 
</details>

 
 
<details>
  <summary>Setting up DNS to your Tailscale IP</summary>
  
 You can obtain your Tailscale IP from the tailscale container
    ```
    docker logs tailscale_vpn | grep peerapi
    2024/11/19 23:56:32 peerapi: serving on http://100.76.158.49:35352
    ```
 
    You should ensure that you have DNS entries for your domain
    ```
    - tailscale.yourdomain.co.uk.	3600	IN	A	100.xxx.xxx.xxx
    - *.tailscale.yourdomain.co.uk. 3600	IN	CNAME	tailscale.yourdomain.co.uk.
    ```

</details>



<details>
  <summary>Obtaining a wildcard certificate from LetsEncrypt</summary>
  
  I recommend setting up API Access for your DNS provider and scripting an SSL renew script
  

  ### tailscale-cert.sh
  ```bash

    #!/bin/bash


    function renew() {
    sudo docker run -it --rm --name certbot \
        -v "/etc/letsencrypt:/etc/letsencrypt" \
        -v "/var/lib/letsencrypt:/var/lib/letsencrypt" \
        -v "/root/ovh.conf:/ovh.conf" \
        certbot/dns-ovh certonly --dns-ovh --dns-ovh-credentials /ovh.conf \
        --agree-tos -m your@email.co.uk \
        -d *.tailscale.yourdomain.co.uk -d tailscale.yourdomain.co.uk
    }

    function merge() {
        if [ -f /etc/ssl/private/tailscale-yourdomain.co.uk.pem ]; then sudo rm /etc/ssl/private/tailscale-yourdomain.co.uk.pem; fi
        sudo find /etc/letsencrypt/live -type l -iname '*pem' -mmin -3 -exec cat "{}" >> /etc/ssl/private/tailscale-yourdomain.co.uk.pem \;
    }

    renew
    merge

  ```
</details>


<details>
<summary>Example haproxy.cfg</summary>

```
global
  log stdout len 65335 local0
  log /dev/log    local0
  log /dev/log    local1 notice
  chroot /var/lib/haproxy
  user haproxy
  group haproxy
  daemon
  #maxconn 4096
  
defaults
  log global
  mode http
  option dontlognull
  option httplog
  timeout connect 50000
  timeout client 50000
  timeout server 50000
  
http-errors httperrors
  errorfile 503 /config/error-pages/503.http
  errorfile 400 /config/error-pages/404.http
  
listen stats
  bind *:9000
  mode http
  log-format "%ci %ST %HM %HP%HQ"
  http-request set-log-level silent
  stats enable
  stats hide-version
  stats scope .
  stats realm Haproxy\ Statistics
  stats uri /
  
frontend http
  bind *:80
  http-request set-var(proc.sub) str("tailscale"),lower if { hdr_end(Host) -i .tailscale.yourdomain.co.uk }
  http-request set-var(proc.application) req.hdr(host),lower,regsub(\.tailscale\.pknw1\.co\.uk$,) if { hdr_end(Host) -i .tailscale.yourdomain.co.uk }
  redirect scheme https if !{ ssl_fc }
  default_backend failback
  
  
frontend secure 
  bind *:443 ssl crt /etc/ssl/private/tailscale-yourdomain.co.uk.pem 
  tcp-request inspect-delay 15s
  errorfiles httperrors
  mode http
  option forwardfor
  http-request set-var(proc.sub) str("insecure"),lower if { hdr_end(Host) -i .yourdomain.co.uk }
  http-request set-var(proc.application) req.hdr(host),lower,regsub(\.pknw1\.co\.uk$,) if { hdr_end(Host) -i .yourdomain.co.uk }
  
  http-request set-var(proc.sub) str("tailscale"),lower if { hdr_end(Host) -i .tailscale.yourdomain.co.uk }
  http-request set-var(proc.application) req.hdr(host),lower,regsub(\.tailscale\.pknw1\.co\.uk$,) if { hdr_end(Host) -i .tailscale.yourdomain.co.uk }
  
  http-request set-var(proc.origin) req.hdr(Origin)
  http-request set-var(proc.referrer) req.hdr(Referer)
  http-request set-var(proc.host) req.hdr(Host)
  http-request set-var(req.scheme) str(https) if { ssl_fc }
  http-request set-var(req.questionmark) str(?) if { query -m found }
  http-request set-var(req.path) path
  http-request set-var(txn.req_hdrs) req.hdrs
  
  http-request set-header X-Forwarded-Host   %[req.hdr(Host)]
  http-request set-header X-Forwarded-For %[src] if ! { req.hdr(X-Forwarded-For) -m found }
  http-request set-header X-Forwarded-Method %[method]
  http-request set-header X-Forwarded-Proto  %[var(req.scheme)]
  http-request set-header X-Forwarded-URI    %[path]%[var(req.questionmark)]%[query]
  http-request set-header X-Forwarded-Proto https if { ssl_fc }
  http-request set-header X-Forwarded-Proto http if !{ ssl_fc }
  
  http-response add-header sec-fetch-site cross-site
  http-response set-header Strict-Transport-Security "max-age=15552000; includeSubDomains"
  http-response set-header X-Frame-Options allowall
  http-response set-header X-Xss-Protection "0" #"1; mode=block"
  
  acl	secure_domain 	hdr_end(Host) -i .secure.yourdomain.co.uk
  acl	admin_domain	hdr_end(Host) -i .admin.yourdomain.co.uk
  acl   top_domain	hdr_end(Host) -i .yourdomain.co.uk
  
  use_backend %[req.hdr(Host)] 
  default_backend failback

# dozzle.tailscale.yourdomain.co.uk
backend dozzle.tailscale.yourdomain.co.uk
option http-server-close
  http-response add-header Set-Cookie "APP=%[var(proc.application)]; max-age=2620800; domain=.yourdomain.co.uk; path=/; samesite=lax; httponly;"
  http-response add-header Set-Cookie "SUB=%[var(proc.sub)]; max-age=2620800; domain=.yourdomain.co.uk; path=/; samesite=lax; httponly;"
  server dozzle_tailscale_pknw1_co_uk0 172.32.0.6:8080

# glances.tailscale.yourdomain.co.uk
backend glances.tailscale.yourdomain.co.uk
option http-server-close
  http-response add-header Set-Cookie "APP=%[var(proc.application)]; max-age=2620800; domain=.yourdomain.co.uk; path=/; samesite=lax; httponly;"
  http-response add-header Set-Cookie "SUB=%[var(proc.sub)]; max-age=2620800; domain=.yourdomain.co.uk; path=/; samesite=lax; httponly;"
  server  glances_tailscale_pknw1_co_uk0 172.32.0.9:61208

# haproxy-ui.tailscale.yourdomain.co.uk
backend haproxy-ui.tailscale.yourdomain.co.uk
option http-server-close
  http-response add-header Set-Cookie "APP=%[var(proc.application)]; max-age=2620800; domain=.yourdomain.co.uk; path=/; samesite=lax; httponly;"
  http-response add-header Set-Cookie "SUB=%[var(proc.sub)]; max-age=2620800; domain=.yourdomain.co.uk; path=/; samesite=lax; httponly;"
  server haproxy-ui_tailscale_pknw1_co_uk0 172.32.0.7:5000

# radarr.tailscale.yourdomain.co.uk
backend radarr.tailscale.yourdomain.co.uk
option http-server-close
  http-response add-header Set-Cookie "APP=%[var(proc.application)]; max-age=2620800; domain=.yourdomain.co.uk; path=/; samesite=lax; httponly;"
  http-response add-header Set-Cookie "SUB=%[var(proc.sub)]; max-age=2620800; domain=.yourdomain.co.uk; path=/; samesite=lax; httponly;"
  server radarr_tailscale_pknw1_co_uk0 172.32.0.5:7878

# whoami.tailscale.yourdomain.co.uk
backend whoami.tailscale.yourdomain.co.uk
option http-server-close
  http-response add-header Set-Cookie "APP=%[var(proc.application)]; max-age=2620800; domain=.yourdomain.co.uk; path=/; samesite=lax; httponly;"
  http-response add-header Set-Cookie "SUB=%[var(proc.sub)]; max-age=2620800; domain=.yourdomain.co.uk; path=/; samesite=lax; httponly;"
  server whoami_tailscale_pknw1_co_uk0 172.32.0.8:8000

# www.tailscale.yourdomain.co.uk
backend www.tailscale.yourdomain.co.uk
option http-server-close
  http-response add-header Set-Cookie "APP=%[var(proc.application)]; max-age=2620800; domain=.yourdomain.co.uk; path=/; samesite=lax; httponly;"
  http-response add-header Set-Cookie "SUB=%[var(proc.sub)]; max-age=2620800; domain=.yourdomain.co.uk; path=/; samesite=lax; httponly;"
  server www_tailscale_pknw1_co_uk0 172.32.0.3:80


backend failback
```

</details>
