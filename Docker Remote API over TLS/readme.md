## Secure Docker Remote API via mTLS (ECDSA + YubiKey-backed Smallstep CA)

This project provisions and deploys mutual TLS (mTLS) credentials to securely expose a Docker daemon over TCP (`tcp://<host>:2376`). It's intended for trusted internal services like Uptime Kuma or automation agents to remotely interact with Docker using strong certificate-based authentication. Instead of exposing the Docker socket or relying on SSH-based access, this setup uses native TLS with client authentication. Docker is configured to require both a valid server certificate and a client certificate signed by the same trusted Certificate Authority (CA), enforcing identity at the network layer.

Certificates are issued by a self-hosted [Smallstep CA](https://smallstep.com/docs/), backed by a **YubiKey-protected root key**. The root is air-gapped and only used for controlled promotions of intermediates. The online intermediate CA supports both **JWK provisioners** (for manual/scriptable issuance with password-file control) and **ACME endpoints** (for automated issuance and renewal). All certificates use **ECDSA keypairs (P-256)** for performance and compactness. Certificate issuance can be integrated with existing trust workflows, and certs can be short-lived for tighter security posture. The resulting identity system supports cryptographic non-repudiation with the added assurance of hardware-backed trust.

TLS is configured explicitly to use modern, hardened cipher suites with strong forward secrecy and authenticated encryption. The Docker daemon listens on TCP 2376 with full certificate validation (`tlsverify: true`), and the client (e.g. Uptime Kuma or a central controller) connects using its own issued cert/key. This model avoids all password-based access and removes the need to expose a shell or socket to external services. Certificates and configuration are automatically deployed to the target host, with `daemon.json` rebuilt and Docker restarted. This ensures both sides of the mTLS handshake are cryptographically validated and rotated securely.

### Cipher Suites Used

- `TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256`
- `TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256`

### Key Features

- Mutual TLS authentication using client and server certificates  
- ECDSA certificates issued by a Smallstep CA with YubiKey-backed root  
- Supports both ACME and JWK provisioners for flexible issuance  
- Secure Docker API access over TCP without exposing the Unix socket  
- Uses hardened TLS cipher suites and short-lived certs  
- Suitable for internal automation and monitoring tools over trusted networks  
- All keys and certs are locally managed, auditable, and under full operator control

This is a hardened, modern setup for secure Docker API access using cryptographic identity and hardware-rooted trust.
