
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


