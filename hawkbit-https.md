# Deploying HTTPS on Hawkbit

In this document, we will see how to deploy https on Hawkbit and how to configure it on your
targets with FullMetalUpdate.

## Setting up the server

### Getting a certificate

Like every other HTTPS service, Hawkbit requires a signed certificate from a Certificate
Authority to comply with SSL protocols. We will not through how to get one, as there
plenty of resources already available on them.  Hawkbit also requires the private key
linked to the certificate.

**Warning:** this setup will not work with self-signed certificates. Please reach out to
us [on gitter][gitter] if you need to configure self-signed certificates.

### Modifying the docker-compose

Go to the cloud directory (`fullmetalupdate-cloud-demo`) and edit `docker-composer.yml`.
Go the `hawkbit:`, and change all the `8080` to `8443` in the `expose` and `ports`
subsections.

Exit, and restart the server using `./StopServer.sh` and `./StartServer.sh`. The server is
not yet configured for https.

### Getting the server's CONTAINER ID

Before setting HTTPS up, you need to take note of the Hawkbit's server container id. You
must have run `./StartServer.sh` from the `fullmetalupdate-cloud-demo` directory.

Run this command:

```
docker ps -a
```

And take note of the `fullmetalupdate/hawkbit`'s CONTAINER ID. We will denote it as
`<hawkbit-id>` in this document.

### Setting up the certificates for Hawkbit

Hawkbit requires a single certificate file generated from the `pem` and the `key` files,
assuming that the `pem` file is the certificate, and the `key` file is the private key
linked to it.

First, create a [`p12` file][wiki_p12] from the last two file:

```
openssl pkcs12 -export -out cert.p12 -in CA.pem -inkey CA.key
```

`CA.pem` and `CA.key` are the certificate and the private key. Enter the private key's
passphrase, which we will denote as `<key-passphrase>` in this document, and type in the
export password, which we will denote as `<p12-password>`. **Warning**: this password must
be at least 6 character long, and contain letters and numbers.

This will create a `cert.p12` file. Copy this file to the currently running container :

```
docker cp cert.p12 <hawkbit-id>:/opt/hawkbit/
```

### Converting the `p12` file into a `jks` file

The Hawkbit container embeds `keytool`, the tool used to manage the Java KeyStore (JKS).
This is what is used by Hawkbit to manage certificates.

Run a shell in the container by executing:

```
docker exec -it <hawkbit-id> /bin/sh
```

Execute `ls`: you should see the `cert.p12` file generated above. Execute the following
command to convert the `p12` file into a `jks` file:

```
keytool -importkeystore -srckeystore cert.p12 -srcstoretype pkcs12 \
        -destkeystore cert.jks -deststoretype pkcs12 \
        -alias 1
```

First, enter the destination keystore password : **this must be the same as
`<p12-password>`**.

Then, enter the source keystore password : `<p12-password>`.

You can now exit the container by executing `exit`.

### Setting up Hawkbit's options for SSL/TLS

Create an `application.properties` file which contains these parameters :

```
# UI demo account
hawkbit.server.ui.demo.password=admin
hawkbit.server.ui.demo.user=admin
hawkbit.server.ui.demo.tenant=DEFAULT

# User Security
security.user.name=admin
security.user.password=admin

# SSL/TLS Configuration
server.port=8443
hawkbit.artifact.url.protocols.download-http.protocol=https
hawkbit.artifact.url.protocols.download-http.port=8443
security.require-ssl=true
server.use-forward-headers=true
server.ssl.key-store=/opt/hawkbit/cert.jks
server.ssl.key-password=<p12-password>
server.ssl.key-store-password=<p12-password>
```

Replace `<p12-password>` accordingly.

Copy this file the currently running container :

```
docker cp application.properties <hawkbit-id>:/opt/hawkbit/
```

This will replace the existing `application.properties`. If you already configured it
please `sh` into the container and append the previous content instead.

The server is all set up! One last step: stop the server (`./StopServer`) and start it
again (`./StartServer`). You can now connect to `https://<your-hostname>.local:8443`.

## Setting up the targets

### Configuration from the build system

Before proceeding further, **close any instanciation of the build-yocto docker** (any
`./Startbuild.sh bash *`).

 1. Go to your Yocto directory (`fullmetalupdate-yocto-demo`)

 1. Edit the `config.cfg.sample` and make the following changes :

     * change `hawkbit_url_port` to `8443`
     * change `hawkbit_ssl` to `true`

 1. Exit, and execute `bash ConfigureBuild.sh`

 1. Execute `./Startbuild.sh fullmetalupdate-os`. It should build the fullmetalupdate
    client again. Then you can flash the new image to your target, which will contain the
    new settings.

 1. After starting the target, you should see your device's IP in the target's
    information panel

### Configuration on the already deployed target

> Note : these changes will be permanent only if you made the changes on the build system
> as described in the last section. Otherwise, the following changes will be lost on the
> next OS update.

If you have already deployed your target and have remote access to it, please follow these
steps :

 1. Remount `/usr` as read-write by executing: `mount -o remount,rw /usr`

 1. Edit the configuration file by executing `vi
    /usr/fullmetalupdate/rauc_hawkbit/config.cfg`

 1. Make the following changes :

     * change `hawkbit_url_port` to `8443`
     * change `hawkbit_ssl` to `true`

 1. Restart the FullMetalUpdate client by executing `systemctl restart fullmetalupdate`

 1. You should see your device's IP in the target's information panel


[wiki_p12]: https://fr.wikipedia.org/wiki/PKCS12
[gitter]: https://gitter.im/fullmetalupdate/community#
