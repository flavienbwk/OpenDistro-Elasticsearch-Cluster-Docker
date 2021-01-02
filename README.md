# OpenDistro cluster - Docker

A fully functional OpenDistro cluster configuration (of 3 ElasticSearch nodes) with TLS enabled and explained. Run with Docker.

Note : It is a bit more of a pain to configure TLS on OpenDistro than the [original Elastic stack](https://github.com/flavienbwk/Secure-Docker-ELK-Cluster) but with some efforts it works !

## Installation

First, you will need to raise your host's ulimits for ElasticSearch to handle high I/O :

```bash
sudo sysctl -w vm.max_map_count=500000
```

Now, we will [generate the certificates](https://opendistro.github.io/for-elasticsearch-docs/docs/security/configuration/generate-certificates/) for your cluster :

```bash
# Copy paste from this project root directory

mkdir -p certs/{ca,kibana,es01,es02,es03}
export OPENDISTRO_DN="/C=FR/ST=IDF/L=PARIS/O=EXAMPLE"   # Edit here and in elasticsearch.yml

# Root CA
openssl genrsa -out certs/ca/ca.key 2048
openssl req -new -x509 -sha256 -days 1095 -subj "$OPENDISTRO_DN/CN=CA" -key certs/ca/ca.key -out certs/ca/ca.pem

# Admin
openssl genrsa -out certs/ca/admin-temp.key 2048
openssl pkcs8 -inform PEM -outform PEM -in certs/ca/admin-temp.key -topk8 -nocrypt -v1 PBE-SHA1-3DES -out certs/ca/admin.key
openssl req -new -subj "$OPENDISTRO_DN/CN=ADMIN" -key certs/ca/admin.key -out certs/ca/admin.csr
openssl x509 -req -in certs/ca/admin.csr -CA certs/ca/ca.pem -CAkey certs/ca/ca.key -CAcreateserial -sha256 -out certs/ca/admin.pem

# Node 1
openssl genrsa -out certs/es01/es01-temp.key 2048
openssl pkcs8 -inform PEM -outform PEM -in certs/es01/es01-temp.key -topk8 -nocrypt -v1 PBE-SHA1-3DES -out certs/es01/es01.key
openssl req -new -subj "$OPENDISTRO_DN/CN=es01" -key certs/es01/es01.key -out certs/es01/es01.csr
openssl x509 -req -extfile <(printf "subjectAltName=DNS:localhost,IP:127.0.0.1,DNS:es01") -in certs/es01/es01.csr -CA certs/ca/ca.pem -CAkey certs/ca/ca.key -CAcreateserial -sha256 -out certs/es01/es01.pem

# Node 2
openssl genrsa -out certs/es02/es02-temp.key 2048
openssl pkcs8 -inform PEM -outform PEM -in certs/es02/es02-temp.key -topk8 -nocrypt -v1 PBE-SHA1-3DES -out certs/es02/es02.key
openssl req -new -subj "$OPENDISTRO_DN/CN=es02" -key certs/es02/es02.key -out certs/es02/es02.csr
openssl x509 -req -extfile <(printf "subjectAltName=DNS:localhost,IP:127.0.0.1,DNS:es02") -in certs/es02/es02.csr -CA certs/ca/ca.pem -CAkey certs/ca/ca.key -CAcreateserial -sha256 -out certs/es02/es02.pem

# Node 3
openssl genrsa -out certs/es03/es03-temp.key 2048
openssl pkcs8 -inform PEM -outform PEM -in certs/es03/es03-temp.key -topk8 -nocrypt -v1 PBE-SHA1-3DES -out certs/es03/es03.key
openssl req -new -subj "$OPENDISTRO_DN/CN=es03" -key certs/es03/es03.key -out certs/es03/es03.csr
openssl x509 -req -extfile <(printf "subjectAltName=DNS:localhost,IP:127.0.0.1,DNS:es03") -in certs/es03/es03.csr -CA certs/ca/ca.pem -CAkey certs/ca/ca.key -CAcreateserial -sha256 -out certs/es03/es03.pem

# Kibana
openssl genrsa -out certs/kibana/kibana-temp.key 2048
openssl pkcs8 -inform PEM -outform PEM -in certs/kibana/kibana-temp.key -topk8 -nocrypt -v1 PBE-SHA1-3DES -out certs/kibana/kibana.key
openssl req -new -subj "$OPENDISTRO_DN/CN=kibana" -key certs/kibana/kibana.key -out certs/kibana/kibana.csr
openssl x509 -req -in certs/kibana/kibana.csr -CA certs/ca/ca.pem -CAkey certs/ca/ca.key -CAcreateserial -sha256 -out certs/kibana/kibana.pem

# Cleanup
rm certs/ca/admin-temp.key certs/ca/admin.csr
rm certs/es01/es01-temp.key certs/es01/es01.csr
rm certs/es02/es02-temp.key certs/es02/es02.csr
rm certs/es03/es03-temp.key certs/es03/es03.csr
rm certs/kibana/kibana-temp.key certs/kibana/kibana.csr
unset OPENDISTRO_DN
```

Start the cluster :

```bash
docker-compose up -d
```

Finally, run `securityadmin` to initialize the security plugin :

```bash
docker-compose exec es01 bash -c "chmod +x plugins/opendistro_security/tools/securityadmin.sh && bash plugins/opendistro_security/tools/securityadmin.sh -cd plugins/opendistro_security/securityconfig -icl -nhnv -cacert config/certificates/ca/ca.pem -cert config/certificates/ca/admin.pem -key config/certificates/ca/admin.key -h localhost"
```

> Find all the configuration files under the container's `plugins/opendistro_security/securityconfig` directory. You might want to [mount them as volumes](https://opendistro.github.io/for-elasticsearch-docs/docs/install/docker-security/).

Access Kibana through [https://localhost:5601](https://localhost:5601)

> Default username is `kibanaserver` and password is `kibanaserver`

## Why OpenDistro

- Fully open source (including plugins)
- Fully under Apache 2.0 license
- Advanced security modules are free
- Allows you to [perform SQL queries against ElasticSearch](https://opendistro.github.io/for-elasticsearch-docs/docs/sql/)
- Maintained by AWS and used for its cloud services
