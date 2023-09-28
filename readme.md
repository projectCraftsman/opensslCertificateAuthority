

```
CA="/home/leland/thing/sync/Projects/opensslCA/demo/CA"
mkdir -p ${CA}/root/certs ${CA}/root/crl ${CA}/root/newcerts ${CA}/root/private ${CA}/root/cnf
cd ${CA} 
chmod 700 ${CA}/root/private
touch ${CA}/root/index.txt
echo 1000 > ${CA}/root/serial
```

```
cat << EOF > ${CA}/root/cnf/openssl.cnf
# OpenSSL root CA configuration file.

[ ca ]
# \`man ca\`
default_ca = CA_default

[ CA_default ]
# Directory and file locations.
dir               = ${CA}/root/
certs             = \$dir/certs
crl_dir           = \$dir/crl
new_certs_dir     = \$certs
database          = \$dir/index.txt
serial            = \$dir/serial
RANDFILE          = \$dir/private/.rand

# The root key and root certificate.
private_key       = \$dir/private/ca.key.pem
certificate       = \$dir/certs/ca.cert.pem

# For certificate revocation lists.
crlnumber         = \$dir/crlnumber
crl               = \$dir/crl/ca.crl.pem
crl_extensions    = crl_ext
default_crl_days  = 30

# SHA-1 is deprecated, so use SHA-2 instead.
default_md        = sha256

name_opt          = ca_default
cert_opt          = ca_default
default_days      = 375
preserve          = no
policy            = policy_strict

[ policy_strict ]
# The root CA should only sign intermediate certificates that match.
# See the POLICY FORMAT section of \`man ca\`.
countryName             = match
stateOrProvinceName     = match
organizationName        = match
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

[ req ]
# Options for the \`req\` tool (\`man req\`).
default_bits        = 2048
distinguished_name  = req_distinguished_name
string_mask         = utf8only
prompt              = no

# SHA-1 is deprecated, so use SHA-2 instead.
default_md          = sha256

# Extension to add when the -x509 option is used.
x509_extensions     = v3_ca

[ req_distinguished_name ]
# See <https://en.wikipedia.org/wiki/Certificate_signing_request>.
countryName                     = Country Name (2 letter code)
stateOrProvinceName             = State or Province Name
localityName                    = Locality Name
0.organizationName              = Organization Name
organizationalUnitName          = Organizational Unit Name
commonName                      = Common Name
emailAddress                    = Email Address

countryName                     = US
stateOrProvinceName             = Kansas 
localityName                    = Lenexa
0.organizationName              = projectCraftsman 
organizationalUnitName          = projectCraftsman Root CA
commonName                      = projectCraftsman Root CA
emailAddress                    = leland@example.com

[ v3_ca ]
# Extensions for a typical CA (\`man x509v3_config\`).
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true
keyUsage = critical, digitalSignature, cRLSign, keyCertSign

[ v3_intermediate_ca ]
# Extensions for a typical intermediate CA (\`man x509v3_config\`).
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true, pathlen:0
keyUsage = critical, digitalSignature, cRLSign, keyCertSign

[ crl_ext ]
# Extension for CRLs (\`man x509v3_config\`).
authorityKeyIdentifier=keyid:always

[ ocsp ]
# Extension for OCSP signing certificates (\`man ocsp\`).
basicConstraints = CA:FALSE
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, digitalSignature
extendedKeyUsage = critical, OCSPSigning
EOF
```

```
cd ${CA}
openssl genrsa -aes256 -out root/private/ca.key.pem 4096
chmod 400 root/private/ca.key.pem
openssl req -config root/cnf/openssl.cnf -key root/private/ca.key.pem -new -x509 -days 36525 -sha256 -extensions v3_ca -out root/certs/ca.cert.pem
chmod 444 root/certs/ca.cert.pem
openssl x509 -noout -text -in root/certs/ca.cert.pem
```

```
mkdir -p ${CA}/intermediate/certs ${CA}/intermediate/crl ${CA}/intermediate/csr ${CA}/intermediate/newcerts ${CA}/intermediate/private ${CA}/intermediate/cnf
chmod 700 ${CA}/intermediate/private
touch ${CA}/intermediate/index.txt
echo 1000 > ${CA}/intermediate/serial
echo 1000 > ${CA}/intermediate/crlnumber
```

```
cat << EOF > ${CA}/intermediate/cnf/openssl.cnf
# OpenSSL intermediate CA configuration file.

[ ca ]
# \`man ca\`
default_ca = CA_default

[ CA_default ]
# Directory and file locations.
dir               = ${CA}/intermediate
certs             = \$dir/certs
crl_dir           = \$dir/crl
new_certs_dir     = \$certs
database          = \$dir/index.txt
serial            = \$dir/serial
RANDFILE          = \$dir/private/.rand

# The root key and root certificate.
private_key       = \$dir/private/intermediate.key.pem
certificate       = \$dir/certs/intermediate.cert.pem

# For certificate revocation lists.
crlnumber         = \$dir/crlnumber
crl               = \$dir/crl/intermediate.crl.pem
crl_extensions    = crl_ext
default_crl_days  = 30

# SHA-1 is deprecated, so use SHA-2 instead.
default_md        = sha256

name_opt          = ca_default
cert_opt          = ca_default
default_days      = 375
preserve          = no
policy            = policy_loose

[ policy_loose ]
# Allow the intermediate CA to sign a more diverse range of certificates.
# See the POLICY FORMAT section of the \`ca\` man page.
countryName             = optional
stateOrProvinceName     = optional
localityName            = optional
organizationName        = optional
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

[ req ]
# Options for the \`req\` tool (\`man req\`).
default_bits        = 2048
distinguished_name  = req_distinguished_name
string_mask         = utf8only
prompt              = no

# SHA-1 is deprecated, so use SHA-2 instead.
default_md          = sha256

# Extension to add when the -x509 option is used.
#x509_extensions     = v3_intermediate_ca

[ req_distinguished_name ]
# See <https://en.wikipedia.org/wiki/Certificate_signing_request>.
countryName                     = Country Name (2 letter code)
stateOrProvinceName             = State or Province Name
localityName                    = Locality Name
0.organizationName              = Organization Name
organizationalUnitName          = Organizational Unit Name
commonName                      = Common Name
emailAddress                    = Email Address

countryName                     = US
stateOrProvinceName             = Kansas 
localityName                    = Lenexa
0.organizationName              = projectCraftsman 
organizationalUnitName          = projectCraftsman Intermediate CA
commonName                      = projectCraftsman Intermediate CA
emailAddress                    = leland@example.com

[ usr_cert ]
# Extensions for client certificates (\`man x509v3_config\`).
basicConstraints = CA:FALSE
nsCertType = client, email
nsComment = "OpenSSL Generated Client Certificate"
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth, emailProtection
subjectAltName=@alt_names

[ server_cert ]
# Extensions for server certificates (\`man x509v3_config\`).
basicConstraints = CA:FALSE
nsCertType = server
nsComment = "OpenSSL Generated Server Certificate"
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer:always
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName=@alt_names

[ crl_ext ]
# Extension for CRLs (\`man x509v3_config\`).
authorityKeyIdentifier=keyid:always

[ ocsp ]
# Extension for OCSP signing certificates (\`man ocsp\`).
basicConstraints = CA:FALSE
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, digitalSignature
extendedKeyUsage = critical, OCSPSigning
EOF
```

```
cd ${CA}
openssl genrsa -aes256 -out intermediate/private/intermediate.key.pem 4096
chmod 400 intermediate/private/intermediate.key.pem
openssl req -config intermediate/cnf/openssl.cnf -new -sha256 -key intermediate/private/intermediate.key.pem -out intermediate/csr/intermediate.csr.pem
```

```
openssl ca -config root/cnf/openssl.cnf -extensions v3_intermediate_ca -days 3650 -notext -md sha256 -in intermediate/csr/intermediate.csr.pem -out intermediate/certs/intermediate.cert.pem
chmod 444 intermediate/certs/intermediate.cert.pem
openssl x509 -noout -text -in intermediate/certs/intermediate.cert.pem
openssl verify -CAfile root/certs/ca.cert.pem intermediate/certs/intermediate.cert.pem
```

```
cat intermediate/certs/intermediate.cert.pem root/certs/ca.cert.pem > intermediate/certs/ca-chain.cert.pem
chmod 444 intermediate/certs/ca-chain.cert.pem
```

```
cat << EOF > ${CA}/intermediate/cnf/www.example.com.cnf
.include ${CA}/intermediate/cnf/openssl.cnf

[ req_distinguished_name ]
countryName                     = US
stateOrProvinceName             = Kansas 
localityName                    = Lenexa
0.organizationName              = projectCraftsman 
organizationalUnitName          = projectCraftsman www.example.com Cert 
commonName                      = www.example.com
emailAddress                    = leland@example.com

[ alt_names ]
DNS.0 = www.example.com
EOF
```

```
cd ${CA} 
openssl genrsa -out intermediate/private/www.example.com.key.pem 2048
chmod 400 intermediate/private/www.example.com.key.pem
openssl req -config intermediate/cnf/www.example.com.cnf -key intermediate/private/www.example.com.key.pem -new -sha256 -out intermediate/csr/www.example.com.csr.pem
openssl ca -config intermediate/cnf/www.example.com.cnf -extensions server_cert -days 375 -notext -md sha256 -in intermediate/csr/www.example.com.csr.pem -out intermediate/certs/www.example.com.cert.pem
chmod 444 intermediate/certs/www.example.com.cert.pem
openssl x509 -noout -text -in intermediate/certs/www.example.com.cert.pem
openssl verify -CAfile intermediate/certs/ca-chain.cert.pem intermediate/certs/www.example.com.cert.pem
```

```
mkdir -p /tmp/whoami/certs
cp ${CA}/intermediate/private/www.example.com.key.pem /tmp/whoami/certs/.
cp ${CA}/intermediate/certs/www.example.com.cert.pem /tmp/whoami/certs/.
sudo docker run -d -p 127.0.0.2:443:80 -v /tmp/whoami/certs:/certs --name iamfoo traefik/whoami --cert /certs/www.example.com.cert.pem --key /certs/www.example.com.key.pem
echo "127.0.0.2 www.example.com" | sudo tee -a /etc/hosts
curl --cacert ./intermediate/certs/ca-chain.cert.pem -i  https://www.example.com:443
sudo sed -i '/127.0.0.2 www.example.com/d' /etc/hosts
sudo docker rm -f iamfoo
rm -rf /tmp/whoami/certs
```

```
cat << EOF > ${CA}/intermediate/cnf/leland.at.example.com.cnf
.include ${CA}/intermediate/cnf/openssl.cnf

[ req_distinguished_name ]
countryName                     = US
stateOrProvinceName             = Kansas 
localityName                    = Lenexa
0.organizationName              = projectCraftsman 
organizationalUnitName          = projectCraftsman leland@example.com Cert 
commonName                      = leland@example.com
emailAddress                    = leland@example.com

[ alt_names ]
email = leland@example.com
EOF
```

```
cd ${CA} 
openssl genrsa -out intermediate/private/leland.at.example.com.key.pem 2048
chmod 400 intermediate/private/leland.at.example.com.key.pem
openssl req -config intermediate/cnf/leland.at.example.com.cnf -key intermediate/private/leland.at.example.com.key.pem -new -sha256 -out intermediate/csr/leland.at.example.com.csr.pem
openssl ca -config intermediate/cnf/leland.at.example.com.cnf -extensions usr_cert -days 375 -notext -md sha256 -in intermediate/csr/leland.at.example.com.csr.pem -out intermediate/certs/leland.at.example.com.cert.pem
chmod 444 intermediate/certs/leland.at.example.com.cert.pem
openssl x509 -noout -text -in intermediate/certs/leland.at.example.com.cert.pem
openssl verify -CAfile intermediate/certs/ca-chain.cert.pem intermediate/certs/leland.at.example.com.cert.pem
```

```
mkdir -p /tmp/whoami/certs
cp ${CA}/intermediate/private/www.example.com.key.pem /tmp/whoami/certs/.
cp ${CA}/intermediate/certs/www.example.com.cert.pem /tmp/whoami/certs/.
cp ${CA}/intermediate/certs/ca-chain.cert.pem /tmp/whoami/certs/.
sudo docker run -d -p 127.0.0.2:443:80 -v /tmp/whoami/certs:/certs --name iamfoo traefik/whoami --cert /certs/www.example.com.cert.pem --key /certs/www.example.com.key.pem --cacert /certs/ca-chain.cert.pem
echo "127.0.0.2 www.example.com" | sudo tee -a /etc/hosts
curl --cert ./intermediate/certs/leland.at.example.com.cert.pem --key ./intermediate/private/leland.at.example.com.key.pem --cacert ./intermediate/certs/ca-chain.cert.pem -i  https://www.example.com:443
sudo sed -i '/127.0.0.2 www.example.com/d' /etc/hosts
sudo docker rm -f iamfoo
rm -rf /tmp/whoami/certs
```