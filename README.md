##setup SSL certificates and provide security for Spinnaker
###1. Generate key and self-signed certificate:

We will use openssl to generate a Certificate Authority (CA) key and a server certificate

####steps to create a certificate authority
*1. Create the CA key*
```
openssl genrsa -des3 -out ca.key 4096
```

*2. Self-sign the CA certificate*
```
openssl req -new -x509 -days 365 -key ca.key -out ca.crt
```

###2. Create the server certificate
Using the self-signed CA cert we created above to sign the server certificate.

*1. Create the server key*
```
openssl genrsa -des3 -out server.key 4096
```

*2. Generate a certificate signing request for the server. Specify localhost or Gate’s eventual fully-qualified domain name (FQDN) as the Common Name (CN).*

```
openssl req -new -key server.key -out server.csr
```

*3. Use the CA to sign the server’s request*
```
openssl x509 -req -days 365 -in server.csr -CA ca.crt \
	-CAkey ca.key -CAcreateserial -out server.crt
```

*4. Format server certificate into Java Keystore (JKS) importable form*

```
YOUR_KEY_PASSWORD=<enter pswd>

openssl pkcs12 -export -clcerts -in server.crt \
	-inkey server.key -out server.p12 -name spinnaker \
	-password pass:$YOUR_KEY_PASSWORD
```

*5. Create Java Keystore by importing CA certificate*

```
keytool -keystore keystore.jks -import -trustcacerts -alias ca -file ca.crt
```

*6. Import server certificate*
```
YOUR_KEY_PASSWORD=<enter pswd>
keytool -importkeystore -srckeystore server.p12 \
	-srcstoretype pkcs12 -srcalias spinnaker \
	-srcstorepass $YOUR_KEY_PASSWORD \
	-destkeystore keystore.jks -deststoretype jks \
	-destalias spinnaker -deststorepass $YOUR_KEY_PASSWORD \
	-destkeypass $YOUR_KEY_PASSWORD
```

we now have a Java Keystore with your certificate authority and server certificate ready to be used by Spinnaker!

###3. Configure SSL for Gate and Deck
With the above certificates and keys in hand, we can use Halyard to set up SSL for Gate and Deck

*For Gate*

```
KEYSTORE_PATH= </path/to/keystore.jks>

hal config security api ssl edit \
  --key-alias spinnaker \
  --keystore $KEYSTORE_PATH \
  --keystore-password \
  --keystore-type jks \
  --truststore $KEYSTORE_PATH \
  --truststore-password \
  --truststore-type jks

hal config security api ssl enable
```

*For Deck*


```
SERVER_CERT= </path/to/server.crt>
SERVER_KEY= </path/to/server.key>

hal config security ui ssl edit \
  --ssl-certificate-file $SERVER_CERT \
  --ssl-certificate-key-file $SERVER_KEY \
  --ssl-certificate-passphrase

hal config security ui ssl enable
hal deploy connect
```
