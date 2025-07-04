Secure Docker Remote API over mTLS (Mutual TLS)
This guide outlines how to expose the Docker Remote API securely using mutual TLS (mTLS). Certificates are issued by a hardened Smallstep CA infrastructure using:

YubiKey-backed EC intermediate key (slot 9c)

Dual provisioners: JWT for interactive and ACME for automated issuance

TLS cipher suites like:

TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256

TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256

Certificate Issuance via step-ca
For Docker Host (node-alpha, IP 10.10.10.10)
bash
Copy
Edit
step ca certificate node-alpha \
  docker-api.pem \
  docker-api-key.pem \
  --san node-alpha \
  --san 10.10.10.10
For Monitoring Client (e.g., Uptime-Kuma on 10.10.10.20)
bash
Copy
Edit
step ca certificate "node-alpha_10.10.10.20" \
  cert.pem \
  key.pem \
  --san 10.10.10.20
Remove Key Passwords
bash
Copy
Edit
# On client
cp key.pem key-with-pass.pem && openssl ec -in key-with-pass.pem -out key.pem && rm key-with-pass.pem

# On Docker host
cp docker-api-key.pem key-with-pass.pem && openssl ec -in key-with-pass.pem -out docker-api-key.pem && rm key-with-pass.pem
(Optional) Test Docker Remote API Access
bash
Copy
Edit
curl --cert data/docker-tls/10.10.10.10/cert.pem \
     --key  data/docker-tls/10.10.10.10/key.pem \
     --cacert data/docker-tls/10.10.10.10/ca.pem \
     https://10.10.10.10:2376/version
Pull Certificates from Step-CA
For Docker Host
bash
Copy
Edit
scp -i ~/.ssh/id_ed25519 root@10.10.10.1:/etc/step/certs/dockerRemoteAPI/docker-hosts/node-alpha/docker-api.pem ~/certs/step-ca/dockerRemoteAPI/docker-hosts/node-alpha/
scp -i ~/.ssh/id_ed25519 root@10.10.10.1:/etc/step/certs/dockerRemoteAPI/docker-hosts/node-alpha/docker-api-key.pem ~/certs/step-ca/dockerRemoteAPI/docker-hosts/node-alpha/
For Monitoring Client
bash
Copy
Edit
scp -i ~/.ssh/id_ed25519 root@10.10.10.1:/etc/step/certs/dockerRemoteAPI/uptime-kuma/10.10.10.20/cert.pem ~/certs/step-ca/dockerRemoteAPI/uptime-kuma/10.10.10.20/
scp -i ~/.ssh/id_ed25519 root@10.10.10.1:/etc/step/certs/dockerRemoteAPI/uptime-kuma/10.10.10.20/key.pem ~/certs/step-ca/dockerRemoteAPI/uptime-kuma/10.10.10.20/
scp -i ~/.ssh/id_ed25519 root@10.10.10.1:/etc/step/certs/dockerRemoteAPI/uptime-kuma/10.10.10.20/ca-bundle.pem ~/certs/step-ca/dockerRemoteAPI/uptime-kuma/10.10.10.20/
Push Certificates to Hosts
To Docker Host (node-beta, IP 10.10.10.10)
bash
Copy
Edit
scp -i ~/.ssh/id_ed25519 ~/certs/step-ca/dockerRemoteAPI/docker-hosts/node-beta/ca-bundle.pem user@10.10.10.10:~/
scp -i ~/.ssh/id_ed25519 ~/certs/step-ca/dockerRemoteAPI/docker-hosts/node-beta/docker-api.pem user@10.10.10.10:~/
scp -i ~/.ssh/id_ed25519 ~/certs/step-ca/dockerRemoteAPI/docker-hosts/node-beta/docker-api-key.pem user@10.10.10.10:~/
To Uptime-Kuma Host (10.10.10.30)
bash
Copy
Edit
scp -i ~/.ssh/id_ed25519 ~/certs/step-ca/dockerRemoteAPI/uptime-kuma/10.10.10.20/cert.pem user@10.10.10.30:~/
scp -i ~/.ssh/id_ed25519 ~/certs/step-ca/dockerRemoteAPI/uptime-kuma/10.10.10.20/key.pem user@10.10.10.30:~/
scp -i ~/.ssh/id_ed25519 ~/certs/step-ca/dockerRemoteAPI/uptime-kuma/10.10.10.20/ca-bundle.pem user@10.10.10.30:~/ca.pem

# Rename if necessary
mv ca-bundle.pem ca.pem
Final Touches
bash
Copy
Edit
chmod 644 *.pem
chmod 600 *key.pem
bash
Copy
Edit
sudo systemctl daemon-reexec
sudo systemctl restart docker
And don’t forget to:

Mount the cert directory in your docker-compose.yml:

yaml
Copy
Edit
volumes:
  - /home/user/docker/uptime-kuma/docker-tls/10.10.10.10:/app/data/docker-tls/10.10.10.10:ro
Define the remote Docker socket in Uptime-Kuma UI (https://10.10.10.10:2376).

If Docker fails to restart, remove any -H override from docker.service.

