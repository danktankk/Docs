<p align="center">
  <img src="https://github.com/danktankk/Docs/blob/main/Docker%20Remote%20API%20over%20TLS/Assets/dockerr.png" style="vertical-align: middle;"/>
</p>

## Secure Docker Remote API via mTLS (3-tier PKI with YubiKey + Smallstep)

This project securely exposes the Docker Remote API over TCP (`tcp://<host>:2376`) using mutual TLS (mTLS) and a hardened three-tier certificate hierarchy. It’s designed to allow trusted internal services (like Uptime Kuma) to connect to Docker remotely using client certificates, rather than exposing the Docker socket, relying on SSH, or allowing unauthenticated TCP access.

The Public Key Infrastructure (PKI) behind this setup uses an offline OpenSSL-generated root CA, which signs a YubiKey-protected intermediate CA certificate. The intermediate key is imported into the YubiKey’s PIV slot and used by a custom-built `step-ca` instance to sign client and server leaf certificates. Both **JWK** and **ACME** provisioners are supported, allowing secure certificate issuance for manual workflows, automation, or internal ACME clients.

All certificates are ECDSA-based (P-256), and mTLS communication is encrypted using strong cipher suites like `TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256`. The Docker daemon is configured to require both server and client certificates, and remote services must present valid certificates signed by your internal CA. This architecture provides strong identity guarantees, hardware-backed private key protection, and clean separation of trust anchors — ideal for secure homelab or production environments.

### Cipher Suites Used

- `TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256`
- `TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256`

### Key Features

- Mutual TLS authentication (both sides present valid certs)
- 3-tier PKI: Offline OpenSSL root > YubiKey intermediate > leaf certs
- Smallstep CA (custom-built with YubiKey support) handles issuance
- ACME + JWK provisioners supported for flexible cert management
- Docker API exposed securely over TCP with full bidirectional TLS verification
- Hardware‑backed intermediate CA on YubiKey for signing mTLS certs (root CA offline; air-gapped, 10-year)

