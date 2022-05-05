# Client TLS

Demo to demonstrate how to use OpenSSL to configure both regular TLS, and additionally client TLS.

There are 4 different parties involved (see below), each party is contained within its own directory
for the purpose of this example.

Parties:
* Public certificate authority (i.e. trusted root such as Amazon, Microsoft, etc)
* Organisation certificate authority (i.e. an organisation's own CA)
* A server (i.e. an application which terminates a TLS connection)
* A client (i.e. party making request to the server, who uses client cert to prove their identity)

## Setup

### Create public CA

This is the CA which signs the server certificate, this is to represent the role of a public cert authority.

The role of this CA is to sign regular TLS certs, so they can be trusted by public devices, for example in
people's browsers when viewing a web page.

```shell
# NOTE passphrase is 1234
openssl genrsa -des3 -out public-ca/public-ca.key 2048
openssl req -x509 -new -nodes -key public-ca/public-ca.key -sha256 -days 1825 -out public-ca/public-ca.crt
# Country Name (2 letter code) []:GB
# State or Province Name (full name) []:London
# Locality Name (eg, city) []:London
# Organization Name (eg, company) []:Global CA Inc.
# Organizational Unit Name (eg, section) []:CA Things
# Common Name (eg, fully qualified host name) []:Global CA Inc. CN
# Email Address []:public-ca@localhost
```

### Create org CA

This is the CA which signs the client certificate, this is to represent the role of an internal corporate cert authority.

The role of this CA is to sign client TLS certs, which can then be sent with signed http requests, so they are verified
by a server to ensure the request comes from a known party.

```shell
# NOTE passphrase is 1234
openssl genrsa -des3 -out org-ca/org-ca.key 2048
openssl req -x509 -new -nodes -key org-ca/org-ca.key -sha256 -days 1825 -out org-ca/org-ca.crt
# Country Name (2 letter code) []:GB
# State or Province Name (full name) []:London
# Locality Name (eg, city) []:London
# Organization Name (eg, company) []:ACME Corp
# Organizational Unit Name (eg, section) []:IT
# Common Name (eg, fully qualified host name) []:ACME Corp CN
# Email Address []:org-ca@localhost
```

## Configure regular TLS

### Create server's private key (if it doesn't exist already)

```shell
# NOTE passphrase is 1234
openssl genrsa -des3 -out server/server.key 2048
```

### Create a CSR for the server and send to CA to be signed

```shell
openssl req -key server/server.key -new -out server/server.csr
# Country Name (2 letter code) []:GB
# State or Province Name (full name) []:London
# Locality Name (eg, city) []:London
# Organization Name (eg, company) []:ACME Corp
# Organizational Unit Name (eg, section) []:Web team
# Common Name (eg, fully qualified host name) []:api.localhost
# Email Address []:server@localhost
mv server/server.csr public-ca/server.csr
```

### Public CA signs the CSR and creates a certificate to send back to the server

Create config file `public-ca/server.ext`
```
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
subjectAltName = @alt_names
[alt_names]
DNS.1 = api.localhost
```

Sign the CSR to create cert and send cert back to server
```shell
openssl x509 -req -CA public-ca/public-ca.crt -CAkey public-ca/public-ca.key -in public-ca/server.csr -out public-ca/server.crt -days 365 -sha256 -CAcreateserial -extfile public-ca/server.ext
mv public-ca/server.crt server/server.crt
rm public-ca/server.csr public-ca/server.ext
```

### Confirm working with nginx

```shell
docker-compose up
# in postman, request https://localhost:8443
```

## Configure client TLS

### Create client's private key (if it doesn't exist already)

```shell
# NOTE passphrase is 1234
openssl genrsa -des3 -out client/client.key 2048
```

### Create a CSR for the server and send to org CA to be signed

```shell
openssl req -key client/client.key -new -out client/client.csr
# Country Name (2 letter code) []:GB
# State or Province Name (full name) []:London
# Locality Name (eg, city) []:London
# Organization Name (eg, company) []:ACME Corp
# Organizational Unit Name (eg, section) []:Some team consuming the api
# Common Name (eg, fully qualified host name) []:Some team CN
# Email Address []:client@localhost
mv client/client.csr org-ca/client.csr
```

### Org CA signs the CSR and creates a certificate to send back to the client

Create config file `org-ca/client.ext`
```
basicConstraints=CA:FALSE
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid,issuer
keyUsage = critical, nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth
```

Sign the CSR to create cert and send cert back to server
```shell
openssl x509 -req -CA org-ca/org-ca.crt -CAkey org-ca/org-ca.key -in org-ca/client.csr -out org-ca/client.crt -days 365 -sha256 -CAcreateserial -extfile org-ca/client.ext
mv org-ca/client.crt client/client.crt
rm org-ca/client.csr org-ca/client.ext
```

### Distribute the org CA root certificate to the server to use for verifying inbound requests

```shell
cp org-ca/org-ca.crt server/org-ca.crt
```

### Confirm working with nginx

```shell
docker-compose up
# Use postman
#   configure client certificate
#     domain: api.localhost
#     port: 9443
#     CRT file: client/client.crt
#     KEY file: client/client.key
#   request GET https://localhost:8443
```
