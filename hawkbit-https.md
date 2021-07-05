# Deploying HTTPS on Hawkbit

In this document, we will see how to set up HTTPS transactions on Hawkbit and how to configure it on your
targets. The final goal of this process is to make your OTA updates secure.

## Setting up the server

### Getting a certificate

Like every other HTTPS service, Hawkbit requires a signed certificate from a Certificate
Authority to comply with SSL protocols.

The following lines will allow you to get a self signed certificate **suitable for test environments only**.

```
sudo apt-get update && sudo apt-get upgrade
sudo apt-get install openssh
openssl req -x509 -newkey rsa:4096 -keyout privateKey.pem -out certificate.crt -days 3650 -nodes
openssl pkcs12 -export -out keyStore.p12 -inkey privateKey.pem -in certificate.crt
```
You will need a passphrase for `privateKey.pem`, which we will denote as #key-passphrase, as well as an export password for the `keyStore.p12`, which we will denote as #p12-password.

**Warning:** Password of `keyStore.p12` (ie, #p12-password) must be at least 6 characters long and contains letters AND numbers.

You will then obtain a certificate `certificate.crt` (the public key is included into this certificate), a private key `privateKey.pem` and a pkcs12 binary `keyStore.p12`. This last one is a special kind of file storing cryptographic objects, in our case your certificate as well as your private key.

**Warning:** This setup will not work for production environments. Please reach out to
us [on gitter][gitter] if you need to configure proper certificates.

**Coming soon:** Proper certificate generation with Let's Encrypt and certbot python tool.


### Modifying the docker-compose

Go to the cloud directory (`fullmetalupdate-cloud-demo`) and do :

`git checkout add-https`

This branch holds everything you need to set up Hawkbit to work with HTTPS. We will come back to this repo later.

### Converting the `p12` file into a `jks` file

`keytool`is a tool used to manage Java KeyStore (JKS) files.
JavaKeyStore objects are used by Hawkbit (note that Hawkbit client is written in Java) to manage certificates. We have to use it because Hawkbit does not seem to be able to deal with pkcs12 files.

Go in a the directoy where you generated `privateKey.pem`, `certificate.crt` and `keyStore.p12`.

You can now generate the jks file :

```
sudo apt-get install keytool
keytool -importkeystore -srckeystore keyStore.p12 -srcstoretype pkcs12 \
        -destkeystore keyStore.jks -deststoretype pkcs12 \
        -alias 1 -deststorepass #p12-password
```

First, enter the destination keystore password : **this must be the same as #p12-password**. Then, enter the source keystore password : #p12-password.

You finally get a JKS file as `keyStore.jks`, which you can use to configure Hawkbit.

### Building new docker image fullmetalupdate/build-yocto:v2.0

#### Configuring Hawkbit's options for SSL/TLS

Go to your FullMetalupdate local repo `dockerfiles` (or clone it on our Github) and configure `hawkbit/application.properties` file with the appropriate parameters.

**1st solution : Use config.sh script**

Move your JavaKeyStore file to the `hawkbit` directory.

`mv /home/user/path/to/keyStore.jks hawkbit/`

Run `config.sh` script and give #p12-password when it is required.

**2nd solution : By hand**

```
1) UI demo account
hawkbit.server.ui.demo.password=admin
hawkbit.server.ui.demo.user=admin
hawkbit.server.ui.demo.tenant=DEFAULT

2) User Security
security.user.name=admin
security.user.password=admin

3) SSL/TLS Configuration
server.port=8443
hawkbit.artifact.url.protocols.download-http.protocol=https
hawkbit.artifact.url.protocols.download-http.port=8443
security.require-ssl=true
server.use-forward-headers=true
server.ssl.key-store=/opt/hawkbit/keyStore.jks
server.ssl.key-password=#p12-password
server.ssl.key-store-password=#p12-password
```
Don't forget to replace #p12-password accordingly.

#### Setting up the build
You will need to generate a new docker image based on a Dockerfile specially written to set up HTTPS on Hawkbit. This step can take up to 1 hour depending on your computer.

```
git clone https://github.com/FullMetalUpdate/dockerfiles.git
git checkout add-https
docker build -t "fullmetalupdate/hawkbit:v2.0" hawkbit/Dockerfile
```

**Note:** When the container is running, you can access it by running `docker exec -it fullmetalupdate-cloud-demo_hawkbit_1 sh`. This will give you the hand on the container by opening a bourne shell inside the container.

At this point, the Hawkbit server is ready to be started.

Go to the cloud directory (`fullmetalupdate-cloud-demo`) and do :

`./StartServer`

Eventually if it is the first time you configure your target, you can lauch :

`./ConfigureServer`

Finally :

`firefox https://localhost:8443 &`

This last command will open the Hawkbit Web Client, working well in HTTPS! But your embedded target is not ready yet to support HTTPS OTA updates.

## Setting up the targets

### Configuration from the build system

Before proceeding further, **close any instanciation of the build-yocto docker** (any
`./Startbuild.sh bash *`).

 1. Go to your Yocto directory (`fullmetalupdate-yocto-demo`)

 2. Edit the `config.cfg` file and make the following changes :

     * change `hawkbit_url_port` to `8443`
     * change `hawkbit_ssl` to `true`

 3. Move your certificate (after renaiming it) into Yocto build container :

```
mv /path/to/certificate.crt /path/to/ca-certificates.crt
docker cp /path/to/ca-certificates.crt \
    fullmetalupdate/hawkbit:v2.0:/data/yocto/build/tmp/fullmetalupdate-os/work/x86_65-linux/curl-native/7.69.1-r0/              recipe-sysroot-native/etc/ssl/certs
```  

 4. Modify a few things to leverage SSL/TLS security (our current certificate is self signed, and whether your browser, curl, or the HTTP API do not like it and fail to compile or execute beacuse of that) :

First, add `-k` option to `curl` bash command in `curl_post()` Yocto function in `build/yocto/source/meta-fullmetalupdate/classes/fullmetalupdae.bbclass` file.

Secondly, go to FMU client repo (`fullmetalupdate/`, a repo you can clone on FullMetalUpdate Github) and modify the source code in `fullmetalupdate.py`: 
``` 
@@ -76,8 +76,8 @@ async def main():
     logging.basicConfig(level=LOG_LEVEL,
                         format='%(asctime)s %(levelname)-8s %(message)s',
                         datefmt='%Y-%m-%d %H:%M:%S')
-
-    async with aiohttp.ClientSession() as session:
+    connTCP = aiohttp.TCPConnector(verify_ssl=False)
+    async with aiohttp.ClientSession(connector=connTCP) as session:
         client = FullMetalUpdateDDIClient(session, HOST, SSL, TENANT_ID, TARGET_NAME,
                                           AUTH_TOKEN, ATTRIBUTES)
``` 

 5. Execute `./Startbuild.sh fullmetalupdate-os`. It should build the fullmetalupdate
    client again. Then you can flash the new image to your target, which will contain the
    new settings.

 6. After starting the target, you should see your device's new IP in the target's information panel on Hawkbit Web Client, which we will denote as #IP.

 7. To finish the deployment, you can use the `send-file` script furnished in FullMetalUpdate/dev-scripts to apply the modification done to FMU client source code.

`./send-file.sh fullmetalupdate.py #IP`

After this final step, you can start deploying your apps updates and OS updates as usual, but by benefiting from the security of HTTPS.

### Configuration on the already deployed target

> Note : these changes will be permanent only if you made the changes on the build system
> as described in the last section. Otherwise, the following changes, made to the target, will be lost on the
> next OS update.

If you have already deployed your target and have remote access to it, please follow these
steps :

 1. Remount `/usr` as read-write by executing: `mount -o remount,rw /usr`

 1. Edit the configuration file by executing `vi /usr/fullmetalupdate/rauc_hawkbit/config.cfg`

 1. Make the following changes :

     * change `hawkbit_url_port` to `8443`
     * change `hawkbit_ssl` to `true`

 1. go to FMU client repo (`fullmetalupdate/`, a repo you can clone on FullMetalUpdate Github) and modify the source code in `fullmetalupdate.py`: 
``` 
@@ -76,8 +76,8 @@ async def main():
     logging.basicConfig(level=LOG_LEVEL,
                         format='%(asctime)s %(levelname)-8s %(message)s',
                         datefmt='%Y-%m-%d %H:%M:%S')
-
-    async with aiohttp.ClientSession() as session:
+    connTCP = aiohttp.TCPConnector(verify_ssl=False)
+    async with aiohttp.ClientSession(connector=connTCP) as session:
         client = FullMetalUpdateDDIClient(session, HOST, SSL, TENANT_ID, TARGET_NAME,
                                           AUTH_TOKEN, ATTRIBUTES)
``` 

 1. After starting the target, you should see your device's new IP in the target's information panel, which we will denote as #IP.

 1. To finish the deployment, you can use the send-file script furnished in fullmetalupdate/dev-scripts to apply the modification done to FMU client source code.

`./send-file.sh fullmetalupdate.py #IP`

You can verify with `journalctl -f | grep fullmetalupdate.sh` that your FMU client is working properly again, in HTTPS.


[wiki_p12]: https://fr.wikipedia.org/wiki/PKCS12
[gitter]: https://gitter.im/fullmetalupdate/community#
