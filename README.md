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

Then we create another certificate based for signing OCSP responses. Notice that we are not using root CA anymore
and isntead use intermediate CA to issue this certificate:

```
cfssl gencert -ca intermediate_ca.pem -ca-key intermediate_ca-key.pem -config=cfssl_config.json -profile=ocsp ocsp_csr.json | cfssljson -bare ocsp
```

In addition to intermediate CA certificates we also need a database to store signed certificates and their OCSP responses.

```
go get -u bitbucket.org/liamstask/goose/cmd/goose
```

```
sudo docker run -p 5432:5432 --name=postgres-cfssl -e POSTGRES_PASSWORD=cfssl -d postgres:latest
```

```
psql -c 'create database certdb_development;' -U postgres -h localhost -p 5432
```

Apply following patch in `/Users/elvinefendi/go/src/github.com/cloudflare/cfssl`:

```
cfssl (master)$ git diff
diff --git a/certdb/pg/dbconf.yml b/certdb/pg/dbconf.yml
index 7880fbd..2371d5e 100644
--- a/certdb/pg/dbconf.yml
+++ b/certdb/pg/dbconf.yml
@@ -1,6 +1,6 @@
 development:
   driver: postgres
-  open: dbname=certdb_development sslmode=disable
+  open: user=postgres password=cfssl dbname=certdb_development sslmode=disable

 test:
   driver: postgres
```

Then we create necessary tables:

```
goose -env development -path $GOPATH/src/github.com/cloudflare/cfssl/certdb/pg up
```

And finally we can start the CA server:

```
cfssl serve -db-config=db-pg.json -ca-key=intermediate_ca-key.pem -ca=intermediate_ca.pem -config=cfssl_config.json -responder=ocsp.pem -responder-key=ocsp-key.pem
```

Note that setting `responder` and `responder-key` does not ensure that OCSP responsed are generated for the issued certificates - so not sure why they are even there.

Now we can use this demo CA to issue i.e sever certificate:

```
cfssl gencert -remote=localhost:8888 -profile=server leaf_csr.json | cfssljson -bare leaf
```

We then generate OCSP responses:

```
cfssl ocsprefresh -ca intermediate_ca.pem -responder=ocsp.pem -responder-key=ocsp-key.pem -db-config=db-pg.json
```

And start OCSP responder server:

```
cfssl ocspserve -port=8080 -db-config=db-pg.json
```

Now we can test the status of our leaf certificate:

```
openssl ocsp -issuer intermediate_ca.pem -no_nonce -cert leaf.pem -CAfile bundle_ca.pem -text -url http://localhost:8080
```

Here `bundle_ca.pem` is the bundle of root and intermediate certificates.

Now we can test revocoation. Frst revoke the leaf certificate using

```
cfssl revoke -db-config=db-pg.json -serial="33290512342717340869779931031481355153102343591" -aki="223068220b3ba60b470bae62af8053b0b40736e8" -reason=superseded
```

In the above command setial can be obtained using:

```
cfssl certinfo -cert leaf.pem | jq .serial_number
```

and aki can be obtained using:

```
cfssl certinfo -cert leaf.pem | jq .authority_key_id | tr -d : | tr '[:upper:]' '[:lower:]'
```

Appendix

```
psql -U postgres -h localhost -p 5432 certdb_development
```
