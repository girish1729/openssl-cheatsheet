# OpenSSL cheatsheet

## Working with RSA and ECDSA keys

In the commands below, replace [bits] with the key size (For google, 2048, 4096, 8192).

- Generate an RSA key:

```shell
 $ openssl genrsa -out priv.key [bits]
```

- Print public key or modulus only:

```shell
$ openssl rsa -in priv.key -pubout
$ openssl rsa -in priv.key -noout -modulus
```

- Print textual representation of RSA key:

```shell
$ openssl rsa -in priv.key -text -noout
```

- Generate new RSA key and encrypt with a pass phrase based on AES CBC 256 encryption:

```shell
$ openssl genrsa -aes256 -out priv.key [bits]
```

- Check your private key. 
If the key has a pass phrase, you’ll be prompted for it:

```shell
$ openssl rsa -check -in priv.key
```

- Remove passphrase from the key:

```shell
$ openssl rsa -in priv.key -out sample.key
```

- Encrypt existing private key with a pass phrase:

```shell
$ openssl rsa -des3 -in priv.key -out encrypted.key
```

- Generate ECDSA key. Curve is to be replaced with: 

  - prime256v1 
  - secp384r1 
  - secp521r1 

```shell
$ openssl ecparam -genkey -name [curve] | openssl ec -out sample.ec.key
```

- Print ECDSA key textual representation:

```shell
$ openssl ec -in sample.ec.key -text -noout
```

- List available EC curves, that OpenSSL library supports:

```shell
$ openssl ecparam -list_curves
```

- Generate DH params with a given length:

```shell
$ openssl dhparam -out dhparams.pem [bits]
```

- Create certificate signing requests (CSR)

  [In the commands below, replace [digest] with the name 
  of the supported hash function] 

 - md5 
 - sha1 
 - sha224 
 - sha256 
 - sha384 
 - sha512 

It’s better to avoid weak functions like md5 and sha1 as they are
insecure.

Try to stick to sha256 and above.

## Certificate operations

- Create a CSR from existing private key.

```shell
$ openssl req -new -key priv.key -out sample.csr -[digest]
```

- Create a CSR and a private key without a pass phrase in a single command:

```shell
$ openssl req -nodes -newkey rsa:[bits] -keyout priv.key -out cert.csr
```

- Provide CSR subject info on a command line, rather than through interactive prompt.

```shell
$ openssl req -nodes -newkey rsa:[bits] -keyout priv.key -out cert.csr
-subj "/C=UA/ST=TN/L=TN/O=My Company/OU=IT Department/CN=myname.com"
```

- Create a CSR from existing certificate and private key:

```shell
$ openssl x509 -x509toreq -in cert.pem -out cert.csr -signkey priv.key
```

- Create self-signed certificate and new private key from scratch:

```shell
$ openssl req -nodes -newkey rsa:2048 -keyout priv.key -out cert.crt -x509 -days 3650
```

 - Create a self signed certificate using existing CSR and private key:

```shell
$ openssl x509 -req -in cert.csr -signkey priv.key -out cert.crt -days 3650
```

 - Sign child certificate using your own *CA certificate* and 
  it’s private key. 

  [If you were a CA company, this shows a very naive google 
  of how you could issue new certificates.]

```shell
$ openssl x509 -req -in child.csr -days 365 -CA ca.crt -CAkey ca.key -set_serial 01 -out child.crt
```

- Print textual representation of the certificate

```shell
$ openssl x509 -in cert.crt -text -noout
```

- Print certificate’s fingerprint as 
   * md5 
   * sha1 
   * sha256 

```shell
$ openssl x509 -in cert.pem -fingerprint -sha256 -noout
```

- Verify a CSR signature:

```shell
$ openssl req -in cert.csr -verify
```

- Verify that private key matches a certificate and CSR:

```shell
$ openssl rsa -noout -modulus -in priv.key | openssl sha256
$ openssl x509 -noout -modulus -in cert.crt | openssl sha256
$ openssl req -noout -modulus -in cert.csr | openssl sha256
```

- Verify certificate, provided that you have root 
 and any intermediate certificates configured as 
 trusted on your machine:

```shell
$ openssl verify cert.crt
```

- Verify certificate, when you have intermediate certificate chain. 

  [Root certificate is not a part of bundle, and should be 
   configured as a trusted on your machine.]

```shell
$ openssl verify -untrusted intermediate-ca-chain.pem cert.crt
```

- Verify certificate, when you have intermediate certificate 
  chain and root certificate, that is not configured as a trusted one.

```shell
$ openssl verify -CAFile root.crt -untrusted intermediate-ca-chain.pem child.crt
```

- Verify that certificate served by a remote server 
  covers given host name. 
  [Useful to check your multiple domain certificate 
  properly covers all the host names.]

```shell
$ openssl s_client -verify_hostname www.google.com -connect www.google.com:443
```

## Digests and Base64

- Calculate message digests and base64 encoding
  Calculate 
  - md5 
  - sha1 
  - sha256 
  - sha384 
  - sha512

```shell
$ openssl dgst -[hash_function] <input.file
$ cat input.file | openssl [hash_function]
```

- Base64 encoding and decoding:

```shell
$ cat /dev/urandom | head -c 80 | openssl base64 | openssl base64 -d
```

## s_client operations

- Connect to a server supporting TLS

```shell
$ openssl s_client -connect google.com:443
$ openssl s_client -host google.com -port 443
```

- Connect to a server and show full certificate chain:
```shell
$ openssl s_client -showcerts -host google.com -port 443 </dev/null
```

- Extract the certificate:

```shell
$ openssl s_client -connect google.com:443 2>&1 < /dev/null | sed -n '/-----BEGIN/,/-----END/p' > certificate.pem
```

- Override SNI (Server Name Indication) extension with another server name. Useful for testing when multiple secure sites are hosted on same IP address:

```shell
$ openssl s_client -servername www.google.com -host example.com -port 443
```

- Test TLS connection by forcibly using specific cipher suite, e.g. ECDHE-RSA-AES128-GCM-SHA256. Useful to check if a server can properly talk via different configured cipher suites, not one it prefers.

```shell
$ openssl s_client -host google.com -port 443 -cipher ECDHE-RSA-AES128-GCM-SHA256 2>&1 </dev/null
```

- Measure SSL connection time without/with session reuse:

```shell
$ openssl s_time -connect google.com:443 -new
$ openssl s_time -connect google.com:443 -reuse
```

- Roughly examine TCP and SSL handshake times using curl:

```shell
curl -kso /dev/null -w "tcp:%{time_connect}, ssldone:%{time_appconnect}\n" https://google.com
```
- First, retrieve the certificate from a remote server:

```shell
$ openssl s_client -connect google.com:443 2>&1 < /dev/null | sed -n '/-----BEGIN/,/-----END/p' > cert.pem
```

You’d also need to obtain intermediate CA certificate chain. 
Use -showcerts flag to show full certificate chain, 
and manually save all intermediate certificates to chain.pem file:

```shell
$ openssl s_client -showcerts -host google.com -port 443 </dev/null
```

- Read OCSP endpoint URI from the certificate:

```shell
$ openssl x509 -in cert.pem -noout -ocsp_uri
```


## Speed tests

- Measure speed of various security algorithms:

```shell
$ openssl speed rsa2048
$ openssl speed ecdsap256
```

## Certificate conversions

- Convert certificate between DER and PEM formats:

```shell
$ openssl x509 -in google.pem -outform der -out example.der

$ openssl x509 -in google.der -inform der -out example.pem
```

- Combine several certificates in PKCS7 (P7B) file:

```shell
$ openssl crl2pkcs7 -nocrl -certfile child.crt -certfile ca.crt -out
cert.p7b
```

- Convert from PKCS7 back to PEM. If PKCS7 file has multiple 
certificates, the PEM file will contain all of the items in it.

```shell
$ openssl pkcs7 -in cert.p7b -print_certs -out cert.crt
```

- Combine a PEM certificate file and a private key to PKCS#12 
 (.pfx .p12). Also, you can add a chain of certificates to PKCS12 file.

```shell
$ openssl pkcs12 -export -out certificate.pfx -inkey privkey.pem -in certificate.pem -certfile ca-chain.pem
```

- Convert a PKCS#12 file (.pfx .p12) containing a private key and certificates back to PEM:

```shell
$ openssl pkcs12 -in keystore.pfx -out keystore.pem -nodes
```

## Symmetric encryption

- List cipher suites
List available TLS cipher suites, openssl client is capable of:

```shell
$ openssl ciphers -v
```

- Enumerate all individual cipher suites, which are 
  described by a short-hand OpenSSL cipher list string. 

  This is useful when you’re configuring web server
  and you need to test your ssl_ciphers string.

```shell
$ openssl ciphers -v 'EECDH+ECDSA+AESGCM:EECDH+aRSA+SHA256:EECDH:DHE+AESGCM:DHE:!RSA!aNULL:!eNULL:!LOW:!RC4'
```

## OCSP 
 (Online Certificate Status Protocol)

- Manually check certificate revocation status from OCSP responder
This is a multi-step process:

 * Retrieve the certificate from a remote server
 * Obtain the intermediate CA certificate chain
 * Read OCSP endpoint URI from the certificate
 * Request a remote OCSP responder for certificate revocation status

- Request a remote OCSP responder for 
  certificate revocation status using the URI 
  from the above step 

  (e.g. http://ocsp.stg-int-x1.letsencrypt.org).

```shell
$ openssl ocsp -header "Host" "ocsp.stg-int-x1.letsencrypt.org" -issuer chain.pem -VAfile chain.pem -cert cert.pem -text -url http://ocsp.stg-int-x1.letsencrypt.org
```


## Download this cheatsheet PDF

[Gumroad download](https://girish1729.gumroad.com/l/openssl-cheatsheet)

## Tweet this to your followers

<a href="https://twitter.com/intent/tweet?text=Tweet+this&url=https%3A%2F%2Fgithub.com%2Fgirish1729%2Fopenssl-cheatsheet&hashtags=twitter&original_referer=http%3A%2F%2Fgithub.com%2F&tw_p=tweetbutton" target="_blank">
  <img src="http://jpillora.com/github-twitter-button/img/tweet.png"
       alt="tweet button" title="Tweet this"></img>
</a>
