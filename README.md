# demo-certificate-authority

Based on certificate signing request information in `ca_csr.json` we initialize the root certificate authority (CA)
and store (using `cfssljson`) the generated certificate request (`ca.csr`), certificate (`ca.pem`) and private key (`ca-key.pem`) in the same directory:

```
cfssl gencert -initca ca_csr.json | cfssljson -bare ca
```

Next we create an intermediate CA based on the root CA:

```
cfssl gencert -ca ca.pem -ca-key ca-key.pem -config=cfssl_config.json -profile=intermediate intermediate_ca_csr.json | cfssljson -bare intermediate_ca
```
