

First things first, this document is based on [work by Jamie Nguyen.](https://jamielinux.com/docs/openssl-certificate-authority/introduction.html) I have added support for Subject Alternative Names on the Server and User certs and have made changes to the directory structure but the lineage should be obvious. 

A few quick thoughts... 

Why did I write this? I went on a side quest to understand Certificate Authorities and Public Key Infrastructure. For myself I wanted to be able to sign and trust certificates for services inside my network. For my clients that are not already managing their own CA and instead choosing to depend on self signed certs for individual applications I wanted to be able to offer something better and back it up with experience. 

Will I be using this to run my own Certificate Authority? Yes. For now... I will eventually get restless and implement [smallstep](https://smallstep.com/docs/step-ca/) or [openxpki](https://github.com/openxpki) with [acme2certifier](https://github.com/grindsa/acme2certifier).

Should you use this? You should read, follow, destroy what you created, rinse and and repeat until you understand how an why this works. You should understand the (undocumented) short cuts I took and decide for yourself. (Hint: I don't cover signing CSRs for certificate/key pairs I didn't create also CRL and OSCP)

Ideally a Certificate Authority is initially created with a Root and Intermediate key and certificate pairs. The Root key is used to sign your Intermediate certificate and the Intermediate key is used to sign Server and User certificates. Depending on your security requirements both the Root and Intermediate key and certificate pairs should be created in on a secure, encrypted, offline (as in [sneakernet](https://en.wikipedia.org/wiki/Sneakernet)) system. The Intermediate key and certificate pair (along with the openssl file structure) can then be copied to a secure, encrypted, online system where it can be used to sign Sever and User Certificates. 

In the event that Intermediate key is compromised or the certificate expires, the Root key can be used to sign a new Intermediate certificate (which should be derived fom a new Intermediate key if the original key was compromised). This document has some plumbing for Certificate Revocation List (CRL) and Online Certificate Status Protocol (OCSP) but at this time I am not going to cover its practical use. 

Now... let's get started.

Create the Root CA directory structure, and configuration for use by `openssl`. The ${CA} environment variable can be modified to meet your needs and is used throughout as a base for absolute paths in config files and some commands. 

Todo: standardize/document/assume all commands are run from ${CA}

```
CA=${HOME}/projectCraftsman/CertificateAuthority 
mkdir -p ${CA}/root/certs ${CA}/root/crl ${CA}/root/private ${CA}/root/cnf
chmod 700 ${CA}/root/private
touch ${CA}/root/index.txt
echo 1000 > ${CA}/root/serial
```

Create the Root CA openssl configuration file. Be sure to review and modify the Distinguished Name values to identify your organization.  

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
default_bits        = 4096 
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

Generate the Root key, generate the Root certificate, and set appropriate file permissions. You will be prompted for a passphrase when generating the Root key, this passphrase will be required to (self) sign the Root certificate and later to sign the Intermediate certificate. Choose and protect this passphrase wisely. Consider an appropriate `-days` value the longer the value (about 10 years in this example) the less likely you are to ever have to create a new Root key/certificate pair. However, without a complete CRL/OSCP configuration the more likely a compromised Root key will will be trusted long into the future. 

```
cd ${CA}
openssl genrsa -aes256 -out root/private/ca.key.pem 4096
chmod 400 root/private/ca.key.pem
openssl req -config root/cnf/openssl.cnf -key root/private/ca.key.pem -new -x509 -days 3650 -sha256 -extensions v3_ca -out root/certs/ca.cert.pem
chmod 444 root/certs/ca.cert.pem
```

Verify your newly created, self signed Root certificate. Confirm the Issuer, Validity, and Subject. 

```
openssl x509 -noout -text -in root/certs/ca.cert.pem
```

Create the Intermediate CA directory structure, and configuration for use by `openssl`. 

```
mkdir -p ${CA}/intermediate/certs ${CA}/intermediate/crl ${CA}/intermediate/csr  ${CA}/intermediate/private ${CA}/intermediate/cnf
chmod 700 ${CA}/intermediate/private
touch ${CA}/intermediate/index.txt
echo 1000 > ${CA}/intermediate/serial
echo 1000 > ${CA}/intermediate/crlnumber
```

Create the Intermediate openssl configuration file. Be sure to review and modify the Distinguished Name values to identify your organization.  

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

Generate the Intermediate key, generate the Intermediate Certificate Signing Request, and set appropriate file permissions. You will be prompted for a passphrase when generating the Intermediate key, this passphrase will be required later to sign Server and User certificates. Choose and protect this passphrase wisely. 

```
cd ${CA}
openssl genrsa -aes256 -out intermediate/private/intermediate.key.pem 4096
chmod 400 intermediate/private/intermediate.key.pem
openssl req -config intermediate/cnf/openssl.cnf -new -sha256 -key intermediate/private/intermediate.key.pem -out intermediate/csr/intermediate.csr.pem
```

Sign the Intermediate certificate using he Root key, and set appropriate file permissions. You will be prompted for the Root key passphrase. Consider an appropriate `-days` value, the longer the value (1 year in this example) the less often you will have to create a new Intermediate certificate. However, without a complete CRL/OSCP configuration the more likely a compromised Intermediate key will will be trusted long into the future. 

```
openssl ca -config root/cnf/openssl.cnf -extensions v3_intermediate_ca -days 3650 -notext -md sha256 -in intermediate/csr/intermediate.csr.pem -out intermediate/certs/intermediate.cert.pem
chmod 444 intermediate/certs/intermediate.cert.pem
```

Verify your newly created, Root signed, Intermediate certificate. Confirm the Issuer, Validity, and Subject. 

```
openssl x509 -noout -text -in intermediate/certs/intermediate.cert.pem
```

Verify the validity of your Intermediate certificate against the Root certificate.

```
openssl verify -CAfile root/certs/ca.cert.pem intermediate/certs/intermediate.cert.pem
```

Concatenate the Root and Intermediate certificates into a single certificate chain, and set appropriate file permissions. The certificate chain can be distributed publicly to enable server and clients to trust the Server and User certificates signed by your Intermediate key. 

```
cat intermediate/certs/intermediate.cert.pem root/certs/ca.cert.pem > intermediate/certs/ca-chain.cert.pem
chmod 444 intermediate/certs/ca-chain.cert.pem
```

If you created your Root and Intermediate key and certificate pairs on an offline system, copy the `intermediate` directory structure from your offline system to a secure, encrypted, online system. Be sure to retain file permissions an attributes.

Generate a Server key and set appropriate file permissions. Depending on your requirements consider including the `-aes256` option, to passphrase protect your Server key. 

```
openssl genrsa -out intermediate/private/www.example.com.key.pem 2048
chmod 400 intermediate/private/www.example.com.key.pem
```

Create an openssl configuration for your Sever certificate. Be sure to review and modify the Distinguished Name and Subject Alternative Name (atl_names) values to identify your organization and server. Note this file includes the Intermediate configuration overriding specific values. 

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

Create a Certificate Signing Request for your Server certificate using your Server key and Server configuration file.

```
openssl req -config intermediate/cnf/www.example.com.cnf -key intermediate/private/www.example.com.key.pem -new -sha256 -out intermediate/csr/www.example.com.csr.pem
```

Sign your Sever certificate and set appropriate file permissions. You will be prompted for the Intermediate key passphrase. Consider an appropriate `-days` value the longer the value (1 year in this example) the less often you will have to renew your Server certificate. However, without a complete CRL/OSCP configuration the more likely a compromised Server key will will be trusted long into the future. 

```
openssl ca -config intermediate/cnf/www.example.com.cnf -extensions server_cert -days 365 -notext -md sha256 -in intermediate/csr/www.example.com.csr.pem -out intermediate/certs/www.example.com.cert.pem
chmod 444 intermediate/certs/www.example.com.cert.pem
```

Verify your newly created, Root signed, Intermediate certificate. Confirm the Issuer, Validity, Subject and Subject Alternative Name.

```
openssl x509 -noout -text -in intermediate/certs/www.example.com.cert.pem
```

Verify the validity of your Server certificate against the Intermediate certificate.

```
openssl verify -CAfile intermediate/certs/ca-chain.cert.pem intermediate/certs/www.example.com.cert.pem
```

Optional: use `docker`, `traefik/whoami` and `curl` to prove your Server certificate works. 

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

Generate a Server key and set appropriate file permissions. Depending on your requirements consider including the `-aes256` option, to passphrase protect your User key.

```
openssl genrsa -out intermediate/private/leland.at.example.com.key.pem 2048
chmod 400 intermediate/private/leland.at.example.com.key.pem
```

Create an openssl configuration for your User certificate. Be sure to review and modify the Distinguished Name and Subject Alternative Name (atl_names) values to identify your organization and User. Note this file includes the Intermediate configuration overriding specific values. 

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

Create a Certificate Signing Request for your User certificate using your User key and User configuration file.

```
openssl req -config intermediate/cnf/leland.at.example.com.cnf -key intermediate/private/leland.at.example.com.key.pem -new -sha256 -out intermediate/csr/leland.at.example.com.csr.pem
```

Sign your Sever certificate and set appropriate file permissions. You will be prompted for the Intermediate key passphrase. Consider an appropriate `-days` value the longer the value (1 year in this example) the less often you will have to renew your User certificate. However, without a complete CRL/OSCP configuration the more likely a compromised User key will will be trusted long into the future. 

```
openssl ca -config intermediate/cnf/leland.at.example.com.cnf -extensions usr_cert -days 365 -notext -md sha256 -in intermediate/csr/leland.at.example.com.csr.pem -out intermediate/certs/leland.at.example.com.cert.pem
chmod 444 intermediate/certs/leland.at.example.com.cert.pem
```

Verify your newly created, Root signed, Intermediate certificate. Confirm the Issuer, Validity, Subject and Subject Alternative Name.

```
openssl x509 -noout -text -in intermediate/certs/leland.at.example.com.cert.pem
```

Verify the validity of your Server certificate against the Intermediate certificate.

```
openssl verify -CAfile intermediate/certs/ca-chain.cert.pem intermediate/certs/leland.at.example.com.cert.pem
```

Optional: use `docker`, `traefik/whoami` and `curl` to prove your User certificate works. 

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