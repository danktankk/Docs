## Source:  https://www.apalrd.net/posts/2024/network_mtls/

## Securely Expose Your Homelab Services with Mutual TLS

### Published: 2024-11-22

---
### If you set up SmallStep, then skip to:  [Sign a User Cert using Smallstep]
#### If you didnt set up smallStep, first, shame on you!
#### Second, you can just follow this excellent guide instead and skip the smallstep section lol
#### There wasnt a traefik or cloudflare section, so I had to kind of invert that along the way.
#### Enjoy!
---

**Tags**: #networking #security

Today I’m diving into Mutual TLS to securely expose my homelab services! TLS is already ubiquitous in the modern era, providing strong symmetric encryption, perfect forward secrecy, and a public chain of trust to authenticate the server. But, it also has a lesser-known ability to authenticate the client. By creating our own certificate authority to issue certs to clients, we can securely authenticate them to the server, preventing other users from even hitting our web app and probing it for vulnerabilities.

This is a simpler solution than using a VPN to ’expose’ your services, as long as the app is already relying on TLS (which includes more protocols than just HTTPS). There’s less user friction in installing a `.p12` cert than setting up a VPN client, which could be important if you are sharing your services with friends and family.

---

## Video

[![Mutual TLS Guide - Secure Your Homelab Services](https://img.youtube.com/vi/YhuWay9XJyw/0.jpg)](https://www.youtube.com/watch?v=YhuWay9XJyw)

---

## Create a Simple Root CA

This section will set up a small 2-layer (root + leaf) authority, instead of the usual 3-layer (root + intermediate + leaf), for testing only. You should probably use the Smallstep CA for anything scalable.

I’m also using an ECDSA key chain, more to see how the process compares to RSA. The root will use `secp384r1` due to its arguably overkill security level, roughly equivalent to ~7000 bit RSA.

```bash
# Generate private key using secp384r1:
openssl ecparam -name secp384r1 -genkey -out root.key

#Sign the root certificate
#Pathlen:0 means there can be only one more cert below this CA (no more CAs)
#Make sure you update the subj name with your own names
#C=US is also the country, it's optional
#O= is the organization, also optional
#CN= is the Common Name and it's required
#I also set validity to 4 years, make sure you watch for expiration (manually)
openssl req -new -key root.key -x509 -nodes -days 1461 -out root.pem -subj "/C=US/O=apalrd.net/CN=test" -addext "basicConstraints=critical,CA:TRUE,pathlen:0"

# View the certificate (optional)
openssl x509 -in root.pem -text -noout
```

---

## Sign a User Cert using the Root CA

Now we can use OpenSSL to generate and sign a user cert using the root key. I chose `secp256` for this since it has a smaller ‘blast radius’ than the root and still provides roughly equivalent security to RSA 3072.

```bash
# Generate secp256 key for this client
openssl ecparam -name prime256v1 -genkey -out adventure@apalrd.net.key

#Generate a CSR (certificate signing request) for my new key
#again, C and O are optional, CN is the Common Name of the cert
openssl req -new -key adventure@apalrd.net.key -out adventure@apalrd.net.csr -subj "/C=US/O=apalrd.net/CN=adventure@apalrd.net" -addext "extendedKeyUsage = clientAuth"

#Sign the CSR using the root
#Sign it allowing for server and client auth as the key usage
openssl x509 -req -in adventure@apalrd.net.csr -CA root.pem -CAkey root.key -CAcreateserial -out adventure@apalrd.net.crt -days 365 -sha256 -copy_extensions=copyall

#Now you can view it (for fun)
openssl x509 -in adventure@apalrd.net.crt -text -noout

#Now let's package it into a P12 archive so you can send it to your favorite client device
#You *must* enter a password here or some OSes will not accept the P12
#The password just encrypts the P12 file itself
openssl pkcs12 -export -out adventure@apalrd.net.p12 -in adventure@apalrd.net.crt -inkey adventure@apalrd.net.key
```

> ** Important note**: I know it’s tempting to try to use Bernsein’s curves (ed25519 especially), but they aren’t approved by the CA + Browser Forum for use in public CAs, and aren’t implemented in any major web browser. So, I’ve chosen scep256 (which OpenSSL confusingly calls prime256) and scep384 for my CA. Of course, RSA is always an option as well.

---

## Sign a User Cert using Smallstep

If you’ve already set up a Smallstep CA from my previous TLS videos, you can use that instead of OpenSSL:

```bash
#Sign cert (run as root on the CA)
#laptop.crt/laptop.key are the key files
#I signed this one for a long ass time
step ca certificate adventure@apalrd.net laptop.crt laptop.key --not-after=2160h

#Bundle into p12 and include intermediate cert we are using
#The P12 file can be imported into any OS
step certificate p12 laptop.p12 laptop.crt laptop.key --ca /etc/step/certs/intermediate_ca.crt
```

---
## Set up Traefik for Client Auth

```config.yml
## 
# I use one file for my Routers, Services, and Middleware.
# Unless I apply middleware to all domains, they go in this file.
# An example of something I use for everything would be crowdsec middleware.
# That would be added in the traefik.yml file as that is a place you can set middleware globally.
# This generally is not what I would want, but there are exceptions as noted.
#
# There is not much to the setup here, fortunately.
# Add the following in config.yml.
# I just put mine at the top since yaml doesnt really care where things go as long as its formatted properly
# You could probably also add it somewhere in traefik.yml after `entrypoints` but this works so I am leaving it here 
##

tls:
  options:
    acmeClient:
      clientAuth:
        caFiles:
          - "/certs/cert1.crt"
          - "/certs/cert2.crt"
          - "/certs/cert3.crt"
        clientAuthType: RequireAndVerifyClientCert

##
# This section above will terminate LOCAL mTLS --per router set for it-- 
# Here is an example router config to work with the TLS section above
##

http:
  routers:
    testapp:
      entryPoints:
        - https
      middlewares:
        - middleware1
        - middleware2
      rule: Host(`foo.domain.com`)
      service: testapp
      tls:
        options: acmeClient
##
# Notice `options: acmeClient` in `https.router.testapp.tls.options` matches the heading at `tls.options.acmeClient`
#
# Next is the service block.  The service does not change and it will terminate using your mTLS cert
# that you have in your traefik configuration directory.  (I.E ~/docker/traefik/certs/cert1.crt)
# I also added the cert1.key here in this directory as well.  The cert1.key is not added in the tls block above
# Next the services portion, again, it stays the same
# I added just to be thorough
##

  services:
    testapp:
      loadBalancer:
        passHostHeader: true
        servers:
          - url: http://192.168.0.2:1234
```
## Set up Traefik for Client Auth & Cloudflare

This was not as easy and required some help from `Xionous` - actually in several areas.
A super knowledgeable individual and equally as helpful.  Many thanks Xionous!

This set up is only a little different and involves some configuration on Cloudflare's dashboard for your domain.
Read this carefully as there are different certs that are responsible for different things.
It is also important to note this will need a separate domain from the first one we used.
Go to your domains dashboard and then go to `SSL/TLS>Client Certificates`
1. add your sub-domains so that mTLS is enforced or add a wild-card as I have done.
![image](https://github.com/user-attachments/assets/716dc703-f362-4d0d-b14c-e455f998223f)
In my case, *.domain.com & domain.com will cover all subdomains made in the future as well.

2. Create a client cert.  
Copy the `.crt` and `.key` file contents to a file on your machine.
Put them in the same certs directory as your local certs made with Step-CA
Name them appropriately (i.e. `cf.crt` / `cf.key`) so they are clearly different than your local cert names.
Go back and save your client cert on Cloudflare now. 

3. Then go to `Security>WAF>custom rules` and look for the mTLS template.
![image](https://github.com/user-attachments/assets/bc86ec68-25c3-4f6b-9791-526a6b2d9f38)
Open that and then set it up to block any traffic that doesnt have the proper client-side cert.
It should be noted that this is VERY effective.  Not all browsers will work, especially on phones.
I have found most chrome based, not all, work once the certs are added.

Set up your firewall rule.  Here is an example that work for all subdomains setup in DNS rules.
![image](https://github.com/user-attachments/assets/a65ffa3b-431c-4c30-a1b9-098564061c36)

I just added one DNS CNAME after my WAN IP.  `domain.com`
This way ALL sub-domains are covered with this one rule.

That should be it for Cloudflare.

Note:  You can also go further and add an `IDP` in `Cloudflare Access` and sets up apps as well.
This will make it so that only those individuals you authorize are able to log into the application 
or service.  I wont be covering that here though.  Maybe later.

5. Back to Traefik:
```config.yml
##
# Adding Cloudflare in `config.yml is much easier now, however the way I set it up, it uses a second domain as mentioned.
# I will differentiate it by calling it domain2.com.  Remember that domain.com is for local mTLS connections and the setup
# for that is above.  Here is the additional setup for config.yml (also including the LOCAL mTLS setup with an alternate domain name)
##

http:
  routers:
    testapp:  ## local mTLS router
      entryPoints:
        - https
      middlewares:
        - middleware1
        - middleware2
      rule: Host(`foo.domain.com`)  ## first domain
      service: testapp
      tls:
        options: acmeClient
    testapp-cf:  ## Cloudflare external mTLS router
      entryPoints:
        - https
      middlewares:
        - middleware1
        - middleware2
      rule: Host(`foo.domain2.com`)  ## second domain
      service: testapp-cf
      tls: {}

  services:
    testapp:  ## local mTLS service
      loadBalancer:
        passHostHeader: true
        servers:
          - url: http://192.168.0.2:1234
    testapp-cf:  ## ## Cloudflare external service
      loadBalancer:
        passHostHeader: true
        servers:
          - url: http://192.168.0.2:1234

```
This provides the best of both worlds as you can have a very secure internal network.  
Anyone that doesnt have the proper client-cert will not be able to login to your applications.
They wont even be able to see the application login page!

Externally, for API's that need to be exposed, you now have Cloudflare client side mTLS set up!
The same security applies here as well, however I would warn that you probably should put all your
eggs in one basket and treat this security measure as a layer of protection, not a complete solution 
as that does not exist.

Continuing on now are other configs you could use
```
---
## Setup Caddy for Client Auth

Here’s the Caddyfile I used (Debian packs a start page with Caddy):

```caddyfile
{
    email mail@example.net
}

test1.apalrd.net {
    root * /usr/share/caddy
    file_server

    tls {
        client_auth {
            mode require_and_verify
            trust_pool file {
                pem_file /etc/caddy/root.pem
            }
        }
    }
}
```

Alternatively, use a snippet for easier reuse:

```caddyfile
{
    email mail@example.net
}

(tls_client) {
    tls {
        client_auth {
            mode require_and_verify
            trust_pool file {
                pem_file /etc/caddy/root.pem
            }
        }
    }
}

test1.apalrd.net {
    root * /usr/share/caddy
    file_server
    import tls_client
}
```

---

## Setup Nginx for Client Auth

Here’s a full nginx configuration (`/etc/nginx/nginx.conf`) with client auth. The default Debian config is quite .. verbose .. so I’ve cut out a lot of things which I didn’t need, but you might have needed them. Anyway, it’s framework. Nginx on Debian also subscribes to the whole `sites-available` and `sites-enabled` thing, which I find unnecessary.

```nginx
user www-data;
worker_processes auto;
worker_cpu_affinity auto;
pid /run/nginx.pid;
error_log /var/log/nginx/error.log;
include /etc/nginx/modules-enabled/*.conf;

events {
    worker_connections 768;
}

http {
    sendfile on;
    tcp_nopush on;
    types_hash_max_size 2048;
    server_tokens off;

    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;

    access_log /var/log/nginx/access.log;

    gzip on;

    server {
        listen 80 default_server;
        server_name _;
        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl default_server;
        listen [::]:443 ssl default_server;

        ssl_certificate     /etc/letsencrypt/live/test-nginx.apalrd.net/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/test-nginx.apalrd.net/privkey.pem;

        ssl_verify_client optional;
        ssl_client_certificate /etc/nginx/root.pem;
        ssl_verify_depth 2;

        if ($ssl_client_verify != "SUCCESS") { return 444; }

        root /var/www/html;
        index index.html index.htm;

        server_name _;

        location / {
            try_files $uri $uri/ =404;
        }
    }
}
```
