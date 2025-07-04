<h1 align="center">Secure Docker Remote API over mTLS</h1>

<div align="center">
  <img src="https://github.com/danktankk/Docs/blob/main/Docker%20Remote%20API%20over%20TLS/Assets/dockerr2.png" width="250" vertical-align: top; align="right"/>
</div>

This setup uses Smallstep CA with a YubiKey-backed EC intermediate (slot 9c). Certificates are issued via JWT or ACME provisioners and secured using modern cipher suites:
```
- TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256
- TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
```
---

# Certificate Issuance via Smallstep
>[!NOTE]
>I’ve developed a script that automates this process, but it’s currently tailored to a very specific environment and unlikely to be directly compatible with most peoples infra. I’m working on revising it to accept runtime variables, so it can scale and adapt to a wider range of lab configurations.

## mTLS for Docker Remote API & uptime monitor
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
### remove password from key files (uptime doesn't have a way to add the password that I know of):
```
(uptime) cp key.pem key-with-pass.pem && openssl ec -in key-with-pass.pem -out key.pem && rm key-with-pass.pem
(docker) cp docker-api-key.pem key-with-pass.pem && openssl ec -in key-with-pass.pem -out docker-api-key.pem && rm key-with-pass.pem
```
### test from the cert dir on uptime kuma (optional)
```
curl --cert data/docker-tls/192.168.100.150/cert.pem \
     --key  data/docker-tls/192.168.100.150/key.pem \
     --cacert data/docker-tls/192.168.100.150/ca.pem \
     https://192.168.100.150:2376/version
```

### pull the certs from step-ca (all in their own dirs)
> [!NOTE] 
> This would be easiest to just `scp`|`rsync`|etc from your **step-ca** machine over directly to the docker host and uptime monitoring solution you have.  I have an intermediary stop in my flow due to how I track and backup certs. Skip this part if you want to and just send direct to the machines you are working on.

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

### push host keys to the appropriate docker host (may need to create the certs dir)
```
scp -i ~/.ssh/arch ~/certs/step-ca/dockerRemoteAPI/docker-hosts/svr2/ca-bundle.pem user@192.168.100.150:~/etc/docker/certs/
scp -i ~/.ssh/arch ~/certs/step-ca/dockerRemoteAPI/docker-hosts/svr2/docker-api.pem user@192.168.100.150:~/etc/docker/certs/
scp -i ~/.ssh/arch ~/certs/step-ca/dockerRemoteAPI/docker-hosts/svr2/docker-api-key.pem user@192.168.100.150:~/etc/docker/certs/
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
then to uptime-kuma dir @ /path/to/uptime-kuma/docker-tls/192.168.100.150/<here>
```
scp -i ~/.ssh/arch ~/certs/step-ca/dockerRemoteAPI/uptime-kuma/192.168.100.150/key.pem user@192.168.100.110:~/docker/uptime-kuma/docker-tls/192.168.100.150/
scp -i ~/.ssh/arch ~/certs/step-ca/dockerRemoteAPI/uptime-kuma/192.168.100.150/cert.pem user@192.168.100.110:~/docker/uptime-kuma/docker-tls/192.168.100.150/
scp -i ~/.ssh/arch ~/certs/step-ca/dockerRemoteAPI/uptime-kuma/192.168.160.150/ca-bundle.pem user@192.168.100.110:~/uptime-kuma/docker-tls/192.168.100.150/
```
>[!IMPORTANT]
> be sure you follow the naming conventions used by uptime-kuma for the certs on the monitor side of this.
uptime-kuma wont use the certs unless you have the correct directories and names
>the directory names also are named in a particular way due to how uptime-kuma
>handles certs for docker hosts.  see the very bottom of the page at https://github.com/louislam/uptime-kuma/wiki/How-to-Monitor-Docker-Containers

- change perms on the certs to 644 for everything but the key which is 600
- docker systemctl daemon-reload on docker host 
- docker systectl restart docker on docker host
>[!WARNING]
>dont forget to add a bind mount for the certs so uptime-kuma can find them!  
>Example:
>      - /home/$USER/docker/uptime-kuma/docker-tls:/app/data/docker-tls:ro
- restart the container:  `docker compose up -d --force-recreate && docker compose logs -f`

## Troubleshooting:
If Docker fails to restart, edit the `override.conf` file for the docker.service to include the following:
`sudo nano /etc/systemd/system/docker.service.d/override.conf`
```
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd --containerd=/run/containerd/containerd.sock
```
1. sudo systemctl daemon-reload
2. sudo systemctl restart docker

# Enjoy
