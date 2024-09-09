---
title: "Certificates, revisited"
date: "2025-12-13"
---

In the never-ending quest for [env parity](../posts/2025-06-13-extreme-env-parity.md), I ran into another interesting issue.  
`mkcert` does not deal with client certificates, it only creates server certs.  
What if we want to do mutual TLS and need them?  
Well, buckle up, we'll take `openssl` for a ride.  
We'll need the three files shown below in a folder named `ca` and the scripts that follow.

```ini
# ca/ca.cnf
[ policy_match ]
countryName = match
stateOrProvinceName = match
organizationName = match
organizationalUnitName = optional
commonName = supplied
emailAddress = optional

[ req ]
prompt = no
distinguished_name = dn
default_md = sha256
default_bits = 4096
x509_extensions = v3_ca

[ dn ]
countryName = GR
organizationName = MyCompany
localityName = Athens
commonName = mycompany-ca

[ v3_ca ]
subjectKeyIdentifier=hash
basicConstraints = critical,CA:true
authorityKeyIdentifier=keyid:always,issuer:always
keyUsage = critical,keyCertSign,cRLSign

# ca/client.cnf
[req]
prompt = no
distinguished_name = dn
default_md = sha256
default_bits = 4096
req_extensions = v3_req

[ dn ]
countryName = GR
organizationName = MyCompnay
localityName = Athens
commonName = mycompany-ca
commonName=*.random.k8s

[ v3_ca ]
subjectKeyIdentifier=hash
basicConstraints = critical,CA:true
authorityKeyIdentifier=keyid:always,issuer:always
keyUsage = critical,keyCertSign,cRLSign

[ v3_req ]
subjectKeyIdentifier = hash
basicConstraints = CA:FALSE
nsComment = "OpenSSL Generated Certificate"
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = *.random.k8s
DNS.2 = random.k8s
DNS.3 = localhost
DNS.4 = 127.0.0.1
DNS.5 = myservice
DNS.6 = myotherservice
DNS.7 = *.infra
DNS.8 = *.app
DNS.9 = *.somedomain.iwantotouse

# ca/server.cnf
[req]
prompt = no
distinguished_name = dn
default_md = sha256
default_bits = 4096
req_extensions = v3_req

[ dn ]
countryName = GR
organizationName = mycompany
localityName = Athens
commonName = mycompany-ca
commonName=*.random.k8s

[ v3_ca ]
subjectKeyIdentifier=hash
basicConstraints = critical,CA:true
authorityKeyIdentifier=keyid:always,issuer:always
keyUsage = critical,keyCertSign,cRLSign

[ v3_req ]
subjectKeyIdentifier = hash
basicConstraints = CA:FALSE
nsComment = "OpenSSL Generated Certificate"
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = *.random.k8s
DNS.2 = random.k8s
DNS.3 = localhost
DNS.4 = 127.0.0.1
DNS.5 = myservice
DNS.6 = myotherservice
DNS.7 = *.infra
DNS.8 = *.app
DNS.9 = *.somedomain.iwantotouse
```

```bash
function create_certs() {
  local CERTS_DIR="${CERTS_DIR:-"/opt/my_certs"}"
  local STORE_PASS="${STORE_PASS:-changeit}"

  # uninstall any existing CAs handled by mkcert
  mkcert -uninstall

  # then delete the CA files
  # and existing client/server certs
  rm -rf $(mkcert -CAROOT)/*
  rm -rf ${CERTS_DIR}/*

  # create new ca
  openssl req -new -nodes \
    -x509 \
    -days 3650 \
    -newkey rsa:2048 \
    -keyout "${CERTS_DIR}"/ca.key \
    -out "${CERTS_DIR}"/ca.crt \
    -config ./ca/ca.cnf

  # create ca pem from crt and key
  cat "${CERTS_DIR}"/ca.crt "${CERTS_DIR}"/ca.key > "${CERTS_DIR}"/ca.pem

  # create ca p12 from existing java trust store
  docker run --rm -it -v "${CERTS_DIR}":/certs  \
    eclipse-temurin:17-jdk -- keytool -importkeystore \
   -srckeystore/opt/java/openjdk/lib/security/cacerts \
   -srcstoretype PKCS12 \
   -srcstorepass changeit \
   -destkeystore /certs/ca.p12  \
   -deststoretype PKCS12 \
   -deststorepass "${STORE_PASS}"

  # change files to have proper user permissions
  sudo chown $(id -u):$(id -g) "${CERTS_DIR}"/ca.p12

  # add our new ca to newly created store
  docker run --rm -it -v ${CERTS_DIR}:/certs  \
    eclipse-temurin:17-jdk \
           -- keytool -importcert -alias my_ca_cert \
                      -file /certs/ca.pem \
                      -keystore /certs/ca.p12 \
                      -deststorepass "${STORE_PASS}" \
                      -noprompt

  # create key and csr for server EKU
  openssl req -new \
      -newkey rsa:2048 \
      -keyout "${CERTS_DIR}"/server.key \
      -out "${CERTS_DIR}"/server.csr \
      -config ./ca/server.cnf \
      -nodes

  # sign the server csr and create cert
  openssl x509 -req \
      -days 3650 \
      -in "${CERTS_DIR}"/server.csr \
      -CA "${CERTS_DIR}"/ca.crt \
      -CAkey "${CERTS_DIR}"/ca.key \
      -CAcreateserial \
      -out "${CERTS_DIR}"/server.crt \
      -extfile ./ca/server.cnf \
      -extensions v3_req

  # create server pem from crt and key
  cat "${CERTS_DIR}"/server.crt "${CERTS_DIR}"/server.key > "${CERTS_DIR}"/server.pem

  # create server p12 keystore
  openssl pkcs12 -export \
      -in "${CERTS_DIR}"/server.crt \
      -inkey "${CERTS_DIR}"/server.key \
      -chain \
      -CAfile "${CERTS_DIR}"/ca.pem \
      -name server \
      -out "${CERTS_DIR}"/server.p12 \
      -password pass:"${STORE_PASS}"

  # change files to have proper user permissions
  sudo chown $(id -u):$(id -g) "${CERTS_DIR}"/server.p12

  # import server p12 to store if needed
  docker run --rm -it -v ${CERTS_DIR}:/certs  \
    eclipse-temurin:17-jdk -- keytool \
         -importcert -alias server_cert \
         -file /certs/server.pem \
         -keystore /certs/ca.p12 \
         -deststorepass "${STORE_PASS}" \
         -noprompt

  # create key and csr for client EKU
  openssl req -new \
      -newkey rsa:2048 \
      -keyout "${CERTS_DIR}"/client.key \
      -out "${CERTS_DIR}"/client.csr \
      -config ./ca/client.cnf \
      -nodes

  # sign the client csr and create cert
  openssl x509 -req \
      -days 3650 \
      -in "${CERTS_DIR}"/client.csr \
      -CA "${CERTS_DIR}"/ca.crt \
      -CAkey "${CERTS_DIR}"/ca.key \
      -CAcreateserial \
      -out "${CERTS_DIR}"/client.crt \
      -extfile ./ca/client.cnf \
      -extensions v3_req

  # create client pem from crt and key
  cat "${CERTS_DIR}"/client.crt "${CERTS_DIR}"/client.key > "${CERTS_DIR}"/client.pem

  # create client p12 keystore
  openssl pkcs12 -export \
      -in "${CERTS_DIR}"/client.crt \
      -inkey "${CERTS_DIR}"/client.key \
      -chain \
      -CAfile "${CERTS_DIR}"/ca.pem \
      -name client \
      -out "${CERTS_DIR}"/client.p12 \
      -password pass:"${STORE_PASS}"

  # change files created to have proper user permissions
  sudo chown $(id -u):$(id -g) "${CERTS_DIR}"/client.p12

  # import client p12 to store if needed
  docker run --rm -it -v ${PWD}:/certs \
         eclipse-temurin:17-jdk \
           -- keytool -importcert -alias client_cert \
           -file /certs/client.pem \
           -keystore /certs/ca.p12 \
           -deststorepass "${STORE_PASS}" \
           -noprompt

  # list certs in store to verify everything imported
  docker run --rm -it -v ${PWD}:/certs  \
         eclipse-temurin:17-jdk \
           -- keytool -list -v -keystore /certs/ca.p12 \
           -storepass "${STORE_PASS}"

  # cleanup files we don't need
  rm -rf "${CERTS_DIR}"/client.csr
  rm -rf "${CERTS_DIR}"/server.csr

  # copy ca files to mkcert folder
  cp "${CERTS_DIR}"/ca.key $(mkcert -CAROOT)/rootCA-key.pem
  cp "${CERTS_DIR}"/ca.crt $(mkcert -CAROOT)/rootCA.pem
  cp "${CERTS_DIR}"/ca.srl $(mkcert -CAROOT)/rootCA.srl

  # trust the ca for your system
  mkcert -install

  # concatenate server cert + ca.cert to use for ingress
  cat ${CERTS_DIR}/server.crt ${CERTS_DIR}/ca.crt > ${CERTS_DIR}/full.pem

}

function create_tls_secrets() {
  kubectl create namespace traefik
  kubectl create namespace infra
  kubectl create namespace app

  # create ingress certificate
  kubectl -n traefik create secret tls my-root-ca-tls \
          --key ${CERTS_DIR}/server.key  \
          --cert ${CERTS_DIR}/server.crt \
          --dry-run=client -o yaml \
          | kubectl apply -f -

  # create secret for plain certificates in infra + app namespaces
  kubectl -n infra create secret generic certificates \
        --from-file=ca.crt=${CERTS_DIR}/ca.crt \
        --from-file=server.crt=${CERTS_DIR}/server.crt \
        --from-file=server.key=${CERTS_DIR}/server.key \
        --from-file=client.crt=${CERTS_DIR}/client.crt \
        --from-file=client.key=${CERTS_DIR}/client.key \
        --dry-run=client -o yaml \
        | kubectl apply -f -

  kubectl -n app create secret generic certificates \
        --from-file=ca.crt=${CERTS_DIR}/ca.crt \
        --from-file=server.crt=${CERTS_DIR}/server.crt \
        --from-file=server.key=${CERTS_DIR}/server.key \
        --from-file=client.crt=${CERTS_DIR}/client.crt \
        --from-file=client.key=${CERTS_DIR}/client.key \
        --dry-run=client -o yaml \
        | kubectl apply -f -

  # create secret for keystores in infra + app namespaces
  kubectl -n infra create secret generic keystores \
        --from-file=ca.p12=${CERTS_DIR}/ca.p12\
        --from-file=server.p12=${CERTS_DIR}/server.p12 \
        --from-file=client.p12=${CERTS_DIR}/client.p12 \
        --from-literal=password="${STORE_PASS}" \
        --dry-run=client -o yaml \
        | kubectl apply -f -

  kubectl -n app create secret generic keystores \
        --from-file=ca.p12=${CERTS_DIR}/ca.p12\
        --from-file=server.p12=${CERTS_DIR}/server.p12 \
        --from-file=client.p12=${CERTS_DIR}/client.p12 \
        --from-literal=password="${STORE_PASS}" \
        --dry-run=client -o yaml \
        | kubectl apply -f -
}
```

Now we've got 2 secrets per namespace.  
One contains certificates, the other keystores.  
We can mount them to our apps as needed to be used for mutual TLS.

> cert-manager would have been nother useful tool here. Food for another adventure.
