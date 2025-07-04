Certificate Issuance with step-ca
For the Docker Host:
bash
Copy
Edit
step ca certificate node-alpha \
  docker-api.pem \
  docker-api-key.pem \
  --san node-alpha \
  --san 10.10.10.10
For the Monitoring Client (e.g. Uptime-Kuma):
bash
Copy
Edit
step ca certificate "node-alpha_10.10.10.20" \
  cert.pem \
  key.pem \
  --san 10.10.10.20
Remove password from private keys:
bash
Copy
Edit
# On client
cp key.pem key-with-pass.pem && openssl ec -in key-with-pass.pem -out key.pem && rm key-with-pass.pem

# On Docker host
cp docker-api-key.pem key-with-pass.pem && openssl ec -in key-with-pass.pem -out docker-api-key.pem && rm key-with-pass.pem
🧪 Optional: Test Docker Remote API Access
bash
Copy
Edit
curl --cert data/docker-tls/10.10.10.10/cert.pem \
     --key  data/docker-tls/10.10.10.10/key.pem \
     --cacert data/docker-tls/10.10.10.10/ca.pem \
     https://10.10.10.10:2376/version
📥 Pull Certificates from Step-CA
For Docker Host:
bash
Copy
Edit
scp -i ~/.ssh/id_ed25519 root@10.10.10.1:/etc/step/certs/dockerRemoteAPI/docker-hosts/node-alpha/docker-api.pem ~/certs/step-ca/dockerRemoteAPI/docker-hosts/node-alpha/
scp -i ~/.ssh/id_ed25519 root@10.10.10.1:/etc/step/certs/dockerRemoteAPI/docker-hosts/node-alpha/docker-api-key.pem ~/certs/step-ca/dockerRemoteAPI/docker-hosts/node-alpha/
For Monitoring Client:
bash
Copy
Edit
scp -i ~/.ssh/id_ed25519 root@10.10.10.1:/etc/step/certs/dockerRemoteAPI/uptime-kuma/10.10.10.20/{cert.pem,key.pem,ca-bundle.pem} ~/certs/step-ca/dockerRemoteAPI/uptime-kuma/10.10.10.20/
📤 Push Certificates to Hosts
Send certs to Docker host:
bash
Copy
Edit
scp -i ~/.ssh/id_ed25519 ~/certs/step-ca/dockerRemoteAPI/docker-hosts/node-beta/ca-bundle.pem user@10.10.10.10:~/
scp -i ~/.ssh/id_ed25519 ~/certs/step-ca/dockerRemoteAPI/docker-hosts/node-beta/docker-api.pem user@10.10.10.10:~/
scp -i ~/.ssh/id_ed25519 ~/certs/step-ca/dockerRemoteAPI/docker-hosts/node-beta/docker-api-key.pem user@10.10.10.10:~/
Send certs to Uptime-Kuma host:
bash
Copy
Edit
scp -i ~/.ssh/id_ed25519 ~/certs/step-ca/dockerRemoteAPI/uptime-kuma/10.10.10.20/{cert.pem,key.pem,ca-bundle.pem} user@10.10.10.30:~/
mv ca-bundle.pem ca.pem  # Rename if needed
🛠 Final Setup
Set permissions:
chmod 644 *.pem && chmod 600 *key.pem

On Docker host:
sudo systemctl daemon-reexec && sudo systemctl restart docker

Add volume to docker-compose.yml (Uptime-Kuma):

yaml
Copy
Edit
    volumes:
      - /home/user/docker/uptime-kuma/docker-tls/10.10.10.10:/app/data/docker-tls/10.10.10.10:ro
Define Docker Remote API in Uptime-Kuma UI.

Note:
If Docker fails to restart due to -H conflict, check your Docker service file and remove conflicting -H flags.
