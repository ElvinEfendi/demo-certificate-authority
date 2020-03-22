# demo-certificate-authority

Based on certificate signing request information in `ca_csr.json` we initialize the root certificate authority (CA)
and store (using `cfssljson`) the generated certificate request (`ca.csr`), certificate (`ca.pem`) and private key (`ca-key.pem`) in the same directory:

```
cfssl gencert -initca ca_csr.json | cfssljson -bare ca
```

Before creating an intermediate CA, and other certificates we need signing configuration for cfssl
