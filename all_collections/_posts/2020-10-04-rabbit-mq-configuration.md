---
title: "How To Set Up RabbitMQ"
date: 2020-10-03 17:49:16
categories: ["rabbitmq"]
---

# Introduction

Surprisingly, there don't seem to be any decent guides on how to configure RabbitMQ properly (especially with TLS working). And it's more fiddly than one would expect. The [documentation](https://www.rabbitmq.com/install-debian.html) doesn't exactly do much to help either. So for anyone in this position, hopefully this will help. This guide will be for Ubuntu 18.04.

# Manual Installation

## Install Erlang

First install some necessary prerequisite packages:

```
sudo apt-get install curl gnupg build-essential apt-transport-https -y
```

Add repository signing key:

```bash
curl -fsSL https://github.com/rabbitmq/signing-keys/releases/download/2.0/rabbitmq-release-signing-key.asc | sudo apt-key add -
```

Update the list of apt sources:

```bash
sudo echo "deb https://dl.bintray.com/rabbitmq-erlang/debian bionic erlang-22.x" | sudo tee /etc/apt/sources.list.d/bintray.erlang.list
sudo apt-get update -y
```

Install all things erlang:

```bash
sudo apt-get install -y erlang-base \
                        erlang-asn1 erlang-crypto erlang-eldap erlang-ftp erlang-inets \
                        erlang-mnesia erlang-os-mon erlang-parsetools erlang-public-key \
                        erlang-runtime-tools erlang-snmp erlang-ssl \
                        erlang-syntax-tools erlang-tftp erlang-tools erlang-xmerl
```

## Install RabbitMQ

```bash
sudo tee /etc/apt/sources.list.d/bintray.rabbitmq.list <<EOF
deb https://dl.bintray.com/rabbitmq-erlang/debian bionic erlang-22.x
deb https://dl.bintray.com/rabbitmq/debian bionic main
EOF

sudo apt-get update -y

sudo apt-get install rabbitmq-server -y --fix-missing
```

Congratulations! You have now installed RabbitMQ. Carry on to add TLS, enable the management plugin and modify users.

## Advanced configuration

### Configure TLS with self-signed certificate

The rabbitmq documentation shows you how to use `tls-gen` to create the required keys/certificates. I would advise against that, instead we will be following the instructions for [manual certificate generation](https://www.rabbitmq.com/ssl.html#manual-certificate-generation) but modified to include the newer `subjectAltName` in addition to the deprecated `commonName`. This is often needed by other libraries to use the TLS certificate as a RabbitMQ client.

```bash
mkdir testca
cd testca
mkdir certs private
chmod 700 private
echo 01 > serial
touch index.txt
```

Still in `testca`, create a file called `openssl.cnf` with the below replacing `<ENTER EXTERNAL IP ADDRESS HERE>` with the IP address or domain name of your server.

{% highlight conf %}
[ ca ]
default_ca = testca

[ testca ]
dir = .
certificate = $dir/ca_certificate.pem
database = $dir/index.txt
new_certs_dir = $dir/certs
private_key = $dir/private/ca_private_key.pem
serial = $dir/serial

default_crl_days = 7
default_days = 365
default_md = sha256

policy = testca_policy
x509_extensions = certificate_extensions

[ testca_policy ]
commonName = supplied
stateOrProvinceName = optional
countryName = optional
emailAddress = optional
organizationName = optional
organizationalUnitName = optional
domainComponent = optional

[ certificate_extensions ]
basicConstraints = CA:false

[ req ]
default_bits = 2048
default_keyfile = ./private/ca_private_key.pem
default_md = sha256
prompt = yes
distinguished_name = root_ca_distinguished_name
x509_extensions = root_ca_extensions
req_extensions = req_ext

[ root_ca_distinguished_name ]
commonName = <ENTER EXTERNAL IP ADDRESS HERE>

[ root_ca_extensions ]
basicConstraints = CA:true
keyUsage = keyCertSign, cRLSign
subjectAltName = @alt_names

[ client_ca_extensions ]
basicConstraints = CA:false
keyUsage = digitalSignature,keyEncipherment
extendedKeyUsage = 1.3.6.1.5.5.7.3.2

[ server_ca_extensions ]
basicConstraints = CA:false
keyUsage = digitalSignature,keyEncipherment
extendedKeyUsage = 1.3.6.1.5.5.7.3.1
subjectAltName = @alt_names

[req_ext]
subjectAltName = @alt_names

[alt_names]
IP.1 = <ENTER EXTERNAL IP ADDRESS HERE>
{% endhighlight %}

Create certificate:

```bash
openssl req -x509 -config openssl.cnf -newkey rsa:2048 -days 365 -out ca_certificate.pem -outform PEM -subj /CN=35.246.50.229/ -nodes
```

To check the certificate has been created correctly, run the command below and you should see something like this:

Instead of `35.246.50.229` you should see whatever you entered as your IP address in `openssl.cnf` above.

```console
$ openssl x509 -noout -subject -text -in ca_certificate.pem
subject=CN = 35.246.50.229
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            39:84:fa:b9:3e:c8:7e:96:a1:89:6e:65:ee:6c:e8:83:b3:a0:d4:47
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN = 35.246.50.229
        Validity
            Not Before: Sep 17 14:48:51 2020 GMT
            Not After : Sep 17 14:48:51 2021 GMT
        Subject: CN = 35.246.50.229
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                RSA Public-Key: (2048 bit)
                Modulus:
                    00:cc:bc:c7:75:0e:0c:fd:cb:a1:42:bb:0d:f8:d8:
                    ff:96:dd:f7:1b:f2:5a:40:d3:f5:82:b9:63:e4:53:
                    c1:da:63:2a:a5:2d:1c:38:d7:9f:48:4d:9d:2a:c5:
                    93:fc:6f:33:02:4a:55:57:5f:d0:0d:22:8c:44:cd:
                    c1:15:b0:aa:8a:43:1c:61:06:75:86:c5:fc:0d:dc:
                    57:b4:c3:8d:be:84:87:38:d7:ff:69:6f:1a:4e:67:
                    81:be:9f:32:57:b9:66:9b:fc:1b:f1:3d:7f:3a:5e:
                    7f:d4:a8:ea:83:33:e9:e5:79:04:09:27:33:26:ff:
                    d6:7f:cd:f9:b9:b2:b1:a3:b2:ba:62:09:ce:08:3b:
                    9b:59:9c:ba:56:69:72:c3:11:f4:03:05:f3:70:56:
                    90:4b:79:03:1f:e8:4e:01:ab:f5:46:b1:ce:e7:49:
                    bb:f8:d8:6f:3d:1c:bf:e1:61:5a:d5:e8:4f:85:78:
                    cb:56:43:ec:59:8f:1e:04:51:c4:24:19:15:8a:03:
                    99:55:f6:17:a7:69:17:4e:b9:5b:ff:33:08:88:e9:
                    a4:e7:46:2d:c8:d4:01:91:ad:46:da:38:9d:6f:65:
                    9e:2c:f0:fe:16:c1:01:9e:46:4d:8c:f9:6c:52:3d:
                    7f:f2:bb:c3:2a:99:e4:df:9e:ee:d6:49:34:17:2f:
                    a1:83
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Basic Constraints:
                CA:TRUE
            X509v3 Key Usage:
                Certificate Sign, CRL Sign
            X509v3 Subject Alternative Name:
                IP Address:35.246.50.229
    Signature Algorithm: sha256WithRSAEncryption
         75:47:43:2f:11:9a:40:ad:fe:a8:f0:40:c1:93:be:86:3e:e9:
         a3:74:21:c5:58:49:33:d1:02:a9:a4:cb:bd:84:a6:80:ea:06:
         0b:b2:42:7a:e1:d4:53:8b:e1:29:bb:1c:2f:7b:e2:ad:5c:d8:
         d9:2a:31:78:37:a4:f4:72:4a:00:7a:dd:b8:7a:35:46:81:a3:
         78:68:c5:de:db:ed:6a:a7:b8:ca:5a:4a:69:25:b0:5b:2b:18:
         2b:57:ab:76:8d:51:a6:8f:92:23:f6:78:d1:c2:73:c2:a2:e7:
         7c:25:37:a0:84:21:16:12:0e:17:13:6d:a9:3f:5c:50:60:df:
         b6:34:67:98:db:20:6e:48:91:f6:6a:3d:eb:9f:47:b4:b1:2f:
         dc:f7:5f:01:a6:4e:17:22:99:6e:3d:fd:2a:52:36:6a:d4:1c:
         47:40:04:34:84:67:d1:80:7f:eb:95:7f:c3:de:a9:8a:6c:bc:
         18:0b:e4:b1:04:e2:53:70:d0:aa:3a:8e:e3:3b:27:c3:2a:89:
         c5:8b:ae:b5:dc:2f:51:53:2a:f2:16:7f:eb:6e:fc:cb:30:b9:
         f9:8e:ff:d8:29:d1:b1:4a:6e:05:8e:2d:f5:b6:3c:fd:24:18:
         bd:ca:9e:e3:28:87:5f:3e:a2:18:b4:35:c6:66:23:88:0c:fe:
         d8:3b:93:4c

```

Then create a private key and certificate signing request:

```bash
mkdir ../server/
cd ../server/
openssl genrsa -out private_key.pem 2048
openssl req -new -key private_key.pem -out req.pem -outform PEM -subj /CN=35.246.50.229/O=server/ -nodes
```

If the last command above doesn't work, click [here](#openssl-bug).

Finally sign the certificate:

```bash
cd ../testca/
openssl ca -config openssl.cnf -in ../server/req.pem -out ../server/server_certificate.pem -notext -batch -extensions server_ca_extensions
chmod 644 ../server/private_key.pem
```

### Add RabbitMQ config file

Now that we have created the required keys/certificates, we need to create a config file and point it towards our keys/certificates:

Run `sudo nano /etc/rabbitmq/rabbitmq.conf` and fill it with the following:

```conf
# Comment the line below to enable unencrypted communication over port 5672
listeners.tcp = none
listeners.ssl.default = 5671

ssl_options.cacertfile = /path/to/testca/ca_certificate.pem
ssl_options.certfile   = /path/to/server/server_certificate.pem
ssl_options.keyfile    = /path/to/server/private_key.pem
ssl_options.verify     = verify_peer
ssl_options.fail_if_no_peer_cert = false


# Uncomment the line below to enable unencrypted management plugin on port 15672
# management.tcp.port       = 15672
management.ssl.port       = 15671
management.ssl.cacertfile = /path/to/testca/ca_certificate.pem
management.ssl.certfile   = /path/to/server/server_certificate.pem
management.ssl.keyfile    = /path/to/server/private_key.pem
```

Obviously take care to replace the paths with the actual paths.

### Set up Management console

```bash
sudo rabbitmq-plugins enable rabbitmq_management
```

### Restart the RabbitMQ Server

This is done for the changes we have made to take effect.

```bash
sudo service rabbitmq-server stop
sudo service rabbitmq-server start
```

### Verify TLS Configuration

Running the command below, you should see in the output that we are now listening on ports 5671 and 15671:

```console
$ sudo rabbitmq-diagnostics listeners
Asking node rabbit@instance-1 to report its protocol listeners ...
Interface: [::], port: 15671, protocol: https, purpose: HTTP API over TLS (HTTPS)
Interface: [::], port: 25672, protocol: clustering, purpose: inter-node and CLI tool communication
Interface: [::], port: 5671, protocol: amqp/ssl, purpose: AMQP 0-9-1 and AMQP 1.0 over TLS
```

### Modify users

The default `guest` user can only be run on `localhost`. Therefore we need to remove this user and create another one:

```bash
sudo rabbitmqctl add_user admin admin
sudo rabbitmqctl set_user_tags admin administrator
sudo rabbitmqctl set_permissions -p / admin ".*" ".*" ".*"
sudo rabbitmqctl delete_user guest
```

Tada! All done.

# Docker Installation

An alternative way to the manual installation is to let `docker` take care of everything for you! First go ahead and install `docker` by following the instructions [here](https://docs.docker.com/engine/install/ubuntu/).

Then ensure you have followed the instructions to create the relevant certificates/keys [here](#configure-tls-with-self-signed-certificate).

Pull the `rabbitmq` docker image:

```bash
docker pull rabbitmq
```

Finally, create the container from the image:

```bash
sudo docker run -d --hostname my-rabbit \
  --name test \
  -p 15671:15671 \
  -p 5671:5671 \
  -e RABBITMQ_DEFAULT_USER=admin \
  -e RABBITMQ_DEFAULT_PASS=admin \
  -e RABBITMQ_SSL_CERTFILE=/server/server_certificate.pem \
  -e RABBITMQ_SSL_KEYFILE=/server/private_key.pem \
  -e RABBITMQ_SSL_CACERTFILE=/testca/ca_certificate.pem \
  -e RABBITMQ_SSL_FAIL_IF_NO_PEER_CERT=false \
  -e RABBITMQ_SSL_VERIFY=verify_peer \
  -v /path/to/server/:/server \
  -v /path/to/testca/:/testca \
  rabbitmq:3-management
```

Take care to replace the `/path/to/` with the actual paths and that should do it!

Verify the container has been created properly by running:

```bash
sudo docker ps
```

You should see the name of your container in the output.

Any other environment variables can be specified by using the `-e` flag as needed.

For more information, take a look at the image description on [docker hub](https://registry.hub.docker.com/_/rabbitmq/)

# Gotchas

Ensure:

- your config file ending is `.conf` and not `.config`. If you use the latter ending, RabbitMQ will expect the contents in Classic Erlang format (which you definitely don't want)
- you use full paths in the config file, you can't use environment variables like `$HOME`
- your private key (and all other keys) have full read access using `chmod 644`
- if you are using a VM, that it has allowed protocols `amqp`, `amqps`, `http`, `https` and opened ports 15671, 15672, 5671 and 5672

Finally to actually use TLS on the client side, you may need to download the certificate we created:
`scp -i <SSH KEY> <USERNAME@IPADDRESS>:/path/to/testca/ca_certificate.pem .`

## OpenSSL Bug

Depending on your `openssl` version, you may get an error like:

```console
Can't load /home/ali/.rnd into RNG
140345410372032:error:2406F079:random number generator:RAND_load_file:Cannot open file:../crypto/rand/randfile.c:88:Filename=/home/ali/.rnd
```

This is a bug and can be solved by following [https://github.com/openssl/openssl/issues/7754](https://github.com/openssl/openssl/issues/7754). Simply go into `/etc/ssl/openssl.cnf` with sudo and remove the two lines which start with `RANDFILE`. Then run the openssl command again.

<!-- sudo rabbitmqctl stop_app
sudo rabbitmqctl reset
sudo rabbitmqctl start_app -->
