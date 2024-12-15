# Securely Expose Your Homelab Services with Mutual TLS

## Published: 2024-11-22

**Tags**: #networking #security

Today I’m diving into Mutual TLS to securely expose my homelab services! TLS is already ubiquitous in the modern era, providing strong symmetric encryption, perfect forward secrecy, and a public chain of trust to authenticate the server. But, it also has a lesser-known ability to authenticate the client. By creating our own certificate authority to issue certs to clients, we can securely authenticate them to the server, preventing other users from even hitting our web app and probing it for vulnerabilities.

This is a simpler solution than using a VPN to ’expose’ your services, as long as the app is already relying on TLS (which includes more protocols than just HTTPS). There’s less user friction in installing a `.p12` cert than setting up a VPN client, which could be important if you are sharing your services with friends and family.

---

## Contents

1. [Video](#video)
2. [Create a Simple Root CA](#create-a-simple-root-ca)
3. [Sign a User Cert using the Root CA](#sign-a-user-cert-using-the-root-ca)
4. [Sign a User Cert using Smallstep](#sign-a-user-cert-using-smallstep)
5. [Setup Caddy for Client Auth](#setup-caddy-for-client-auth)
6. [Setup Nginx for Client Auth](#setup-nginx-for-client-auth)

---

## Video

[![Mutual TLS Guide - Secure Your Homelab Services](https://img.youtube.com/vi/qfF0xH5N2nM/0.jpg)](https://www.youtube.com/watch?v=qfF0xH5N2nM)

---

## Create a Simple Root CA

This section will set up a small 2-layer (root + leaf) authority, instead of the usual 3-layer (root + intermediate + leaf), for testing only. You should probably use the Smallstep CA for anything scalable.

I’m also using an ECDSA key chain, more to see how the process compares to RSA. The root will use `secp384r1` due to its arguably overkill security level, roughly equivalent to ~7000 bit RSA.

```bash
# Generate private key using secp384r1:
openssl ecparam -name secp384r1 -genkey -out root.key

# Sign the root certificate
openssl req -new -key root.key -x509 -nodes -days 1461 -out root.pem -subj "/C=US/O=apalrd.net/CN=test" -addext "basicConstraints=critical,CA:TRUE,pathlen:0"

# View the certificate (optional)
openssl x509 -in root.pem -text -noout
```

---

## Sign a User Cert using the Root CA

Now we can use OpenSSL to generate and sign a user cert using the root key. I chose `secp256r1` for this since it has a smaller ‘blast radius’ than the root and still provides roughly equivalent security to RSA 3072.

```bash
# Generate secp256r1 key for this client
openssl ecparam -name prime256v1 -genkey -out adventure@apalrd.net.key

# Generate a CSR (certificate signing request) for the new key
openssl req -new -key adventure@apalrd.net.key -out adventure@apalrd.net.csr -subj "/C=US/O=apalrd.net/CN=adventure@apalrd.net" -addext "extendedKeyUsage = clientAuth"

# Sign the CSR using the root
openssl x509 -req -in adventure@apalrd.net.csr -CA root.pem -CAkey root.key -CAcreateserial -out adventure@apalrd.net.crt -days 365 -sha256 -copy_extensions=copyall

# Package it into a P12 archive
openssl pkcs12 -export -out adventure@apalrd.net.p12 -in adventure@apalrd.net.crt -inkey adventure@apalrd.net.key
```

> **Note**: Some OSes require a password for the P12 archive.

---

## Sign a User Cert using Smallstep

If you’ve already set up a Smallstep CA from my previous TLS videos, you can use that instead of OpenSSL:

```bash
# Sign cert (run as root on the CA)
step ca certificate adventure@apalrd.net laptop.crt laptop.key --not-after=2160h

# Bundle into p12 and include intermediate cert
step certificate p12 laptop.p12 laptop.crt laptop.key --ca /etc/step/certs/intermediate_ca.crt
```

---

## Setup Caddy for Client Auth

Here’s the Caddyfile I used:

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

Here’s a full nginx configuration with client auth:

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
