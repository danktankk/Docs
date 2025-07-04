

## Secure Docker Remote API over mTLS (mutual TLS)
### For the docker host:
```
step ca certificate host \
  docker-api.pem \
  docker-api-key.pem \
  --san host \
  --san 192.0.2.10
```
### For the uptime monitor
```
step ca certificate "pi4-3_192.0.2.10" \
  cert.pem \
  key.pem \
  --san 192.0.2.10
```
### remove password from key files:
(uk) cp key.pem key-with-pass.pem && openssl ec -in key-with-pass.pem -out key.pem && rm key-with-pass.pem
(docker) cp docker-api-key.pem key-with-pass.pem && openssl ec -in key-with-pass.pem -out docker-api-key.pem && rm key-with-pass.pem

### test from the cert dir on uptime kuma (optional and more for if it doesnt work at first)
```
curl --cert data/docker-tls/192.0.2.10/cert.pem \
     --key  data/docker-tls/192.0.2.10/key.pem \
     --cacert data/docker-tls/192.0.2.10/ca.pem \
     https://192.0.2.10:2376/version
```

### pull the certs from step-ca (all in their own dirs)

```
docker host keys ():
scp -i ~/.ssh/workstation root@192.0.2.10:/etc/step/certs/dockerRemoteAPI/docker-hosts/host/docker-api.pem ~/certs/step-ca/dockerRemoteAPI/docker-hosts/host/
docker-api.pem
scp -i ~/.ssh/workstation root@192.0.2.10:/etc/step/certs/dockerRemoteAPI/docker-hosts/host/docker-api-key.pem ~/certs/step-ca/dockerRemoteAPI/docker-hosts/host/
docker-api-key.pem

uptime-kuma keys for this particular host:
scp -i ~/.ssh/workstation root@192.0.2.10:/etc/step/certs/dockerRemoteAPI/uptime-kuma/192.0.2.10/ca-bundle.pem ~/certs/step-ca/dockerRemoteAPI/uptime-kuma/192.0.2.10/
scp -i ~/.ssh/workstation root@192.0.2.10:/etc/step/certs/dockerRemoteAPI/uptime-kuma/192.0.2.10/cert.pem ~/certs/step-ca/dockerRemoteAPI/uptime-kuma/192.0.2.10/
scp -i ~/.ssh/workstation root@192.0.2.10:/etc/step/certs/dockerRemoteAPI/uptime-kuma/192.0.2.10/key.pem ~/certs/step-ca/dockerRemoteAPI/uptime-kuma/192.0.2.10/
```

### push host keys to the appropriate docker host
```
scp -i ~/.ssh/workstation ~/certs/step-ca/dockerRemoteAPI/docker-hosts/host/ca-bundle.pem user@192.0.2.10:~/
scp -i ~/.ssh/workstation ~/certs/step-ca/dockerRemoteAPI/docker-hosts/host/docker-api.pem user@192.0.2.10:~/
scp -i ~/.ssh/workstation ~/certs/step-ca/dockerRemoteAPI/docker-hosts/host/docer-api-key.pem user@192.0.2.10:~/

then to uptime-kuma dir @ /docker-tls/<192.0.2.10>/<here>:
scp -i ~/.ssh/workstation ~/certs/step-ca/dockerRemoteAPI/uptime-kuma/192.0.2.10/key.pem user@192.0.2.10:~/
scp -i ~/.ssh/workstation ~/certs/step-ca/dockerRemoteAPI/uptime-kuma/192.0.2.10/cert.pem user@192.0.2.10:~/
scp -i ~/.ssh/workstation ~/certs/step-ca/dockerRemoteAPI/uptime-kuma/192.0.2.10/ca-bundle.pem user@192.0.2.10:~/ca.pem

be sure you rename ca-bundle.pem to ca.pem
uptime-kuma wont work unless you have the correct durectories and names
```

- change perms on the certs to 644 for everything but the key which is 600
- docker systemctl daemon-reload on docker host 
- docker systectl restart docker on docker host
- add volume to docker-compose:
      - /home/dankk/docker/uptime-kuma/docker-tls/192.0.2.10:/app/data/docker-tls/192.0.2.10:ro
- dfl on uptime kuma

if error on docker restart - docker service needs to be edited to remove that -H part
