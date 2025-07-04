# Secure Docker Remote API over mTLS   <p align="right">
  <img src="https://github.com/danktankk/Docs/blob/main/Docker%20Remote%20API%20over%20TLS/Assets/dockerr2.png" height="50" style="vertical-align: middle;"/>
</p>

This setup uses Smallstep CA with a YubiKey-backed EC intermediate (slot 9c). Certificates are issued via JWT or ACME provisioners and secured using modern cipher suites:
```
- TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256
- TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
```
---

# Certificate Issuance via Smallstep

## TLS for Docker Remote API
### For the docker host:
```
step ca certificate host1 \
  docker-api.pem \
  docker-api-key.pem \
  --san host1 \
  --san 192.168.100.150
```
### For the uptime monitor
```
step ca certificate "host1_192.168.100.150" \
  cert.pem \
  key.pem \
  --san 192.168.100.150
```
### remove password from key files:
```
(uptime) cp key.pem key-with-pass.pem && openssl ec -in key-with-pass.pem -out key.pem && rm key-with-pass.pem
(docker) cp docker-api-key.pem key-with-pass.pem && openssl ec -in key-with-pass.pem -out docker-api-key.pem && rm key-with-pass.pem
```
### test from the cert dir on uptime kuma (optional)
```
curl --cert data/docker-tls/192.168.100.110/cert.pem \
     --key  data/docker-tls/192.168.100.110/key.pem \
     --cacert data/docker-tls/192.168.100.110/ca.pem \
     https://192.168.100.110:2376/version
```

### pull the certs from step-ca (all in their own dirs)
note: 

```
docker host keys:
scp -i ~/.ssh/arch root@192.168.100.122:/etc/step/certs/dockerRemoteAPI/docker-hosts/host1/docker-api.pem ~/certs/step-ca/dockerRemoteAPI/docker-hosts/host1/
docker-api.pem
scp -i ~/.ssh/arch root@192.168.100.122:/etc/step/certs/dockerRemoteAPI/docker-hosts/host1/docker-api-key.pem ~/certs/step-ca/dockerRemoteAPI/docker-hosts/host1/
docker-api-key.pem

uptime-kuma keys for this particular host:
scp -i ~/.ssh/arch root@192.168.100.122:/etc/step/certs/dockerRemoteAPI/uptime-kuma/192.168.100.150/ca-bundle.pem ~/certs/step-ca/dockerRemoteAPI/uptime-kuma/192.168.160.150/
scp -i ~/.ssh/arch root@192.168.100.122:/etc/step/certs/dockerRemoteAPI/uptime-kuma/192.168.100.150/cert.pem ~/certs/step-ca/dockerRemoteAPI/uptime-kuma/192.168.160.150/
scp -i ~/.ssh/arch root@192.168.100.122:/etc/step/certs/dockerRemoteAPI/uptime-kuma/192.168.100.150/key.pem ~/certs/step-ca/dockerRemoteAPI/uptime-kuma/192.168.160.150/
```

### push host keys to the appropriate docker host
```
scp -i ~/.ssh/arch ~/certs/step-ca/dockerRemoteAPI/docker-hosts/pi4-2/ca-bundle.pem dankk@192.168.100.150:~/
scp -i ~/.ssh/arch ~/certs/step-ca/dockerRemoteAPI/docker-hosts/pi4-2/docker-api.pem dankk@192.168.100.150:~/
scp -i ~/.ssh/arch ~/certs/step-ca/dockerRemoteAPI/docker-hosts/pi4-2/docer-api-key.pem dankk@192.168.100.150:~/
```
#### Add the following to: /etc/docker/daemon.json on docker host
```
{
  "hosts": ["unix:///var/run/docker.sock", "tcp://192.168.100.150:2376"],
  "tls": true,
  "tlsverify": true,
  "tlscacert": "/etc/docker/certs/ca-bundle.pem",
  "tlscert": "/etc/docker/certs/docker-api.pem",
  "tlskey": "/etc/docker/certs/docker-api-key.pem"
}
```

```
then to uptime-kuma dir @ /path/to/uptime-kuma/docker-tls/<ip for docker host>/<here>
scp -i ~/.ssh/arch ~/certs/step-ca/dockerRemoteAPI/uptime-kuma/192.168.100.150/key.pem dankk@192.168.100.110:~/docker/uptime-kuma/docker-tls/192.168.100.150/
scp -i ~/.ssh/arch ~/certs/step-ca/dockerRemoteAPI/uptime-kuma/192.168.100.150/cert.pem dankk@192.168.100.110:~/docker/uptime-kuma/docker-tls/192.168.100.150/
scp -i ~/.ssh/arch ~/certs/step-ca/dockerRemoteAPI/uptime-kuma/192.168.160.150/ca-bundle.pem dankk@192.168.100.110:~/uptime-kuma/docker-tls/192.168.100.150/
```
## be sure you rename ca-bundle.pem to ca.pem
## uptime-kuma wont use the cert unless you have the correct directories and names

- change perms on the certs to 644 for everything but the key which is 600
- docker systemctl daemon-reload on docker host 
- docker systectl restart docker on docker host
- add volume to docker-compose:
      - /home/$USER/docker/uptime-kuma/docker-tls/192.168.160.150:/app/data/docker-tls/192.168.160.150:ro
- restart the container:  docker compose up -d --force-recreate && docker compose logs -f

If Docker fails to restart, edit the `override.conf` file for the docker.service to include the following:
`sudo nano /etc/systemd/system/docker.service.d/override.conf`
```
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd --containerd=/run/containerd/containerd.sock
```
1. sudo systemctl daemon-reload
2. sudo systemctl restart docker

Should be working now
