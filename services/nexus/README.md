# Nexus

The implementation (for now) is in docker-compose, but deliberately is not shared in the public repository. See `/opt/nexus` on Morris and edit things with care. In future the implementation will be shared - if nothing else for redundancy reasons.

## Ingress
In order to balance the multiple roles of the Nexus instance and not require developers to remember port numbers etc, Nginx Ingress is provided. As the project moves forwards, this will likely need to relocate from being a specific Nginx ingress for Nexus, to a wider path based routing system. But YAGNI for now.

Nginx will not provide WAF, much better options exist.

### Routing
All traffic incoming on port 80 will be redirected (301'd) to SSL on 443, with the request URI and parameters unaffected.

The supported subdomains for this ingress are:
- nexus.edpn.io

Navigation to an unsupported subdomain (or direct IP) results the Nexus interface loading, but functionality is not guaranteed to work.

### SSL
For now, SSL is provided by a local self-signed certificate. As/when this becomes a problem, better options are available. Note to developers: install the self-signed certificate locally if you don't want your browser or build tools nagging you.

By default, ingress will only support TLSv1.3, due to the vulnerabilities already announced in TLSv1.2. This means Ops will not be able to inspect network traffic beyond counters without themselves committing a MITM. Should this be problematic for the group, TLSv1.2 has no formal deprecation date (yet), so we can compromise.

This also means that traffic from Nginx to Nexus is not encrypted, and in future, should be mTLS.

To create a self-signed certificate on Morris (if trying on a Windows machine in gitbash, remember `export MSYS_NO_PATHCONV=1` first):
1. Create the root CA.
```shell
openssl req -x509 \
            -sha256 -days 356 \
            -nodes \
            -newkey rsa:2048 \
            -subj "/O=EDPN/OU=EDPN Ops/CN=nexus.edpn.io" \
            -keyout edpn_rootCA.key -out edpn_rootCA.crt 
```
2. Create the Private key.
```shell
openssl genrsa -out server.key 2048
```
3. Create the signing request configuration.
```shell
cat > csr.conf <<EOF
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[ dn ]
O = EDPN
OU = EDPN Ops
CN = nexus.edpn.io

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = nexus.edpn.io

EOF
```
4. Create the signing request.
```shell
openssl req -new -key edpn_rootCA.key -out nexus.edpn.io.csr -config csr.conf
```
5. Create the external request configuration.
```shell
cat > cert.conf <<EOF

authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = nexus.edpn.io

EOF
```
6. Finally, generate the certificate itself.
```shell
openssl x509 -req \
    -in nexus.edpn.io.csr \
    -CA edpn_rootCA.crt -CAkey edpn_rootCA.key \
    -CAcreateserial -out nexus.edpn.io.crt \
    -days 365 \
    -sha256 -extfile cert.conf
```
7. Save the private key and public certificate to the directory that Nginx will read them from (see `nexus.conf`), then restart the Nexus container and tidy up any intermediary files.

### Logging
Logs are written to stdout and can be redirected to an aggregation system when we have one. Disk usages is expected to be minimal, especially early in the project.

## Nexus

### Data retention
Important for upgrades as well as disk space, `docker inspect nexus` to view the location of data (see the mount point for `nexus-data`). If upgrading Nexus at Sonotype suggestion due to a notification, remember to check if the versions are compatible and if not, how to migrate.

Nexus supports the concept of cleanup policies, but as yet, none exist. As/when data storage becomes an issue, this can be implemented.

### Logging
Logs are written to stdout and can be redirected to an aggregation system when we have one. Nexus Admins (Ops) can also see logs within the Nexus UI.

### Authentication
The admin password has been set securely (and shared appropriately). Ops members are welcome to be added to the admin group. For now, accounts are local until we need them not to be.

Developers are welcome to have upload capability. The current instance supports anonymous pull (for both Docker and Maven/Gradle). @Frontend, let us know your requirements, and if Nexus supports it, Ops will add it. Discuss with OfficialyInsane if you wish.

### Access policy (to be redefined by group agreement).
1. Anonymous can download from any repository.
2. Authenticated users can change their passwords.
2. Project Developers have Authenticated user permissions, and can upload to any repository (this may be removed/restricted in future).
3. Build applications (GitHub Workflows) are equal to Project Developers (agian, may be changed in future).
4. Group Leads have Developer permissions, and can administer any repository.
5. Operations have Group Lead permissions, and can administer the Nexus instance.

### Push policy
Artifact overwrites are allowed. Be careful when pushing.

### Misc Developer notes
1. Remember, self signed-certificate. Tell Docker to trust insecure certificates (or install the certificate locally) if you want to login.
2. `docker login https://nexus.edpn.io` - give it your Nexus credentials (speak to Ops if you need credentials).
3. `docker tag <ref> nexus.edpn.io/<image_path>` - the subdomain can change in future per request. 
4. Prefer GA Workflows over manual pushing, but don't use the build server to build for local tests.

### Maven Developer notes
Suggested settings.xml:
```xml
<settings>
    <servers>
        <server>
            <id>edpn-releases</id>
            <username>test-developer</username>
            <password>{xWrjYZfXWtYHsQIcWy9GgapgKrVloZCP7guhoVwMpwg=}</password>
        </server>
        <server>
            <id>edpn-snapshots</id>
            <username>test-developer</username>
            <password>{xWrjYZfXWtYHsQIcWy9GgapgKrVloZCP7guhoVwMpwg=}</password>
        </server>
    </servers>
    <repositories>
        <repository>
            <id>edpn-mirror</id>
            <name>EDPN Maven Central Mirror</name>
            <url>https://nexus.edpn.io/repository/maven-public/</url>
        </repository>
        <repository>
            <id>edpn-releases</id>
            <name>EDPN FOSS Repository</name>
            <url>https://nexus.edpn.io/repository/maven-releases/</url>
        </repository>
        <repository>
            <id>edpn-snapshots</id>
            <name>EDPN FOSS Snapshots</name>
            <url>https://nexus.edpn.io/repository/maven-snapshots/</url>
        </repository>
    </repositories>
    <mirrors>
        <mirror>
            <id>edpn-public</id>
            <name>Other Mirror Repository</name>
            <url>https://nexus.edpn.io/repository/maven-public/</url>
            <mirrorOf>*</mirrorOf>
        </mirror>
    </mirrors>
</settings>
```

Due to the insecure certificate, don't forget `-Dmaven.wagon.http.ssl.insecure=true` when building/pushing.

Use encrypted local passwords. Consider the Sonatype deploy plugin (`nexus-staging-maven-plugin`).