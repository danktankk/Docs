# Secure Docker Remote API over mTLS

This setup uses Smallstep CA with a YubiKey-backed EC intermediate (slot 9c). Certificates are issued via JWT or ACME provisioners and secured using modern cipher suites:
```
- TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256
- TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
```
---

## Certificate Issuance via Smallstep

# For Docker Host (node-alpha, IP 192.0.2.20)
```
step ca certificate node-alpha \
  docker-api.pem \
  docker-api-key.pem \
  --san node-alpha \
  --san 192.0.2.20
```

# For Monitoring Client (Uptime-Kuma on 192.0.2.20)
```
step ca certificate "node-alpha_10.20.1.100" \
  cert.pem \
  key.pem \
  --san 192.0.2.20
```
---

## Remove Key Passwords
# On Uptime-Kuma
`cp key.pem key-with-pass.pem && openssl ec -in key-with-pass.pem -out key.pem && rm key-with-pass.pem`

# On Docker Host
`cp docker-api-key.pem key-with-pass.pem && openssl ec -in key-with-pass.pem -out docker-api-key.pem && rm key-with-pass.pem`

---

## Test Docker Remote API Access
```
curl --cert data/docker-tls/192.0.2.20/cert.pem \
     --key  data/docker-tls/192.0.2.20/key.pem \
     --cacert data/docker-tls/192.0.2.20/ca.pem \
     https://192.0.2.20:2376/version
```
---

## Pull Certificates from Step-CA

# Docker Host: node-alpha
`scp -i ~/.ssh/id_ed25519 root@192.0.2.20:/etc/step/certs/dockerRemoteAPI/docker-hosts/node-alpha/docker-api.pem ~/certs/step-ca/dockerRemoteAPI/docker-hosts/node-alpha/`
`scp -i ~/.ssh/id_ed25519 root@192.0.2.20:/etc/step/certs/dockerRemoteAPI/docker-hosts/node-alpha/docker-api-key.pem ~/certs/step-ca/dockerRemoteAPI/docker-hosts/node-alpha/`

# Uptime-Kuma Client
`scp -i ~/.ssh/id_ed25519 root@192.0.2.20:/etc/step/certs/dockerRemoteAPI/uptime-kuma/192.0.2.20/cert.pem ~/certs/step-ca/dockerRemoteAPI/uptime-kuma/192.0.2.20/`
`scp -i ~/.ssh/id_ed25519 root@192.0.2.20:/etc/step/certs/dockerRemoteAPI/uptime-kuma/192.0.2.20/key.pem ~/certs/step-ca/dockerRemoteAPI/uptime-kuma/192.0.2.20/`
`scp -i ~/.ssh/id_ed25519 root@192.0.2.20:/etc/step/certs/dockerRemoteAPI/uptime-kuma/192.0.2.20/ca-bundle.pem ~/certs/step-ca/dockerRemoteAPI/uptime-kuma/192.0.2.20/`

---

## Push Certificates to Hosts

# Docker Host: node-beta (192.0.2.20)
`scp -i ~/.ssh/id_ed25519 ~/certs/step-ca/dockerRemoteAPI/docker-hosts/node-beta/ca-bundle.pem user@192.0.2.20:~/`
`scp -i ~/.ssh/id_ed25519 ~/certs/step-ca/dockerRemoteAPI/docker-hosts/node-beta/docker-api.pem user@192.0.2.20:~/`
`scp -i ~/.ssh/id_ed25519 ~/certs/step-ca/dockerRemoteAPI/docker-hosts/node-beta/docker-api-key.pem user@192.0.2.20:~/`

# Uptime-Kuma Host (192.0.2.20)
`scp -i ~/.ssh/id_ed25519 ~/certs/step-ca/dockerRemoteAPI/uptime-kuma/192.0.2.20/cert.pem user@192.0.2.20:~/`
`scp -i ~/.ssh/id_ed25519 ~/certs/step-ca/dockerRemoteAPI/uptime-kuma/192.0.2.20/key.pem user@192.0.2.20:~/`
`scp -i ~/.ssh/id_ed25519 ~/certs/step-ca/dockerRemoteAPI/uptime-kuma/192.0.2.20/ca-bundle.pem user@192.0.2.20:~/ca.pem`

mv ca-bundle.pem ca.pem

---

## Final Setup
```
chmod 644 *.pem
chmod 600 *key.pem

sudo systemctl daemon-reexec
sudo systemctl restart docker
```
---

## Docker Compose Volume Mount

# Inside uptime-kuma service block:
```
volumes:
  - /home/user/docker/uptime-kuma/docker-tls/192.0.2.20:/app/data/docker-tls/192.0.2.20:ro
```
---

## Uptime-Kuma Config

Add or Update Docker Host:
https://192.0.2.20:2376 and test

If Docker fails to start, remove any `-H` lines from the systemd unit file and reload the service.
