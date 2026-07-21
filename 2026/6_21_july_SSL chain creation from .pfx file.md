A **.pfx (PKCS#12)** file contains the private key, server certificate, and often the intermediate/root certificates. You can extract the certificate chain from it using OpenSSL.

### Step 1: Check the contents of the PFX

```bash
openssl pkcs12 -info -in certificate.pfx
```

It will ask for the PFX password and display the certificates and private key.

---

## Option 1: Extract all certificates (recommended)

This extracts all certificates (server + intermediates + root) without the private key.

```bash
openssl pkcs12 -in certificate.pfx -nokeys -out certificates.pem
```

The output file will contain multiple certificates like:

```text
-----BEGIN CERTIFICATE-----
(Server Certificate)
-----END CERTIFICATE-----

-----BEGIN CERTIFICATE-----
(Intermediate CA)
-----END CERTIFICATE-----

-----BEGIN CERTIFICATE-----
(Root CA)
-----END CERTIFICATE-----
```

---

## Option 2: Extract only the CA chain

```bash
openssl pkcs12 -in certificate.pfx -cacerts -nokeys -out ca-chain.pem
```

This extracts only the intermediate and root certificates.

---

## Option 3: Extract the server certificate

```bash
openssl pkcs12 -in certificate.pfx -clcerts -nokeys -out server.crt
```

---

## Step 4: Create the full SSL chain

If you have:

* `server.crt`
* `ca-chain.pem`

Create the full chain:

```bash
cat server.crt ca-chain.pem > fullchain.pem
```

The order must always be:

1. Server certificate
2. Intermediate certificate(s)
3. Root certificate (optional for most web servers)

---

## Verify the chain

```bash
openssl verify -CAfile ca-chain.pem server.crt
```

Or inspect the certificates:

```bash
openssl x509 -in server.crt -text -noout
```

---

## Extract the private key (if needed)

```bash
openssl pkcs12 -in certificate.pfx -nocerts -out private.key
```

To remove the passphrase:

```bash
openssl rsa -in private.key -out private-unencrypted.key
```

---

### Typical files after extraction

```
certificate.pfx
├── server.crt
├── private.key
├── ca-chain.pem
└── fullchain.pem
```

If you are creating the chain for **Nginx**, **Apache**, **IIS**, **HAProxy**, or **Kubernetes Ingress**, let me know the target platform and I can provide the exact format and configuration required.
