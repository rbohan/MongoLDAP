# MongoDB and OpenLDAP

This guide shows you how to install and configure OpenLDAP on Ubuntu 18.04 in order to use for Authentication & Autorization with MongoDB.

## Setup

### Create VM:

```
vagrant init ubuntu/bionic64
```

Edit Vagrant file to expose port 389 as, e.g., 3389 or bridge network (we'll assume a bridged network accessible via `ldap.mongodb.local` - edit `/etc/hosts` to set this up if required)

### Start VM & connect:

```
vagrant up
vagrant ssh
```

## Install & Configure LDAP

### Update apt & install LDAP utils
```
sudo apt-get update
sudo apt install slapd ldap-utils ldapscripts
```

* Administrator password: `admin`
* Confirm password: `admin`

```
sudo dpkg-reconfigure slapd
```

* Omit OpenLDAP server configuration? `no`
* DNS domain name: `ldap.mongodb.local`
* Organization name: `mongodb` (????)
* Administrator password: `admin`
* Confirm password: `admin`
* Database backend to use: `MDB`
* Do you want the database to be removed when slapd is purged? `no`
* Move old database? `yes`

### Install memberof overlay
```
cat <<EOF > memberof_load_configure.ldif
dn: cn=module{1},cn=config
cn: module{1}
objectClass: olcModuleList
olcModuleLoad: memberof
olcModulePath: /usr/lib/ldap

dn: olcOverlay={0}memberof,olcDatabase={1}mdb,cn=config
objectClass: olcConfig
objectClass: olcMemberOf
objectClass: olcOverlayConfig
objectClass: top
olcOverlay: memberof
olcMemberOfDangling: ignore
olcMemberOfRefInt: TRUE
olcMemberOfGroupOC: groupOfNames
olcMemberOfMemberAD: member
olcMemberOfMemberOfAD: memberOf
EOF

sudo ldapadd -Q -Y EXTERNAL -H ldapi:/// -f memberof_load_configure.ldif
```

```
cat <<EOF > refint1.ldif
dn: cn=module{1},cn=config
add: olcmoduleload
olcmoduleload: refint
EOF

sudo ldapmodify -Q -Y EXTERNAL -H ldapi:/// -f refint1.ldif
```

```
cat <<EOF > refint2.ldif
dn: olcOverlay={1}refint,olcDatabase={1}mdb,cn=config
objectClass: olcConfig
objectClass: olcOverlayConfig
objectClass: olcRefintConfig
objectClass: top
olcOverlay: {1}refint
olcRefintAttribute: memberof member manager owner
EOF

sudo ldapadd -Q -Y EXTERNAL -H ldapi:/// -f refint2.ldif
```

### Create users & groups

Connect to the LDAP server using, e.g., ApacheDirectoryStudio. Under `dc=ldap,dc=mongodb,dc=local` create the following entries:

* add `organizationalUnit`: `ou=people`
  * add `inetOrgPerson`: `uid=ronan`
  * add `inetOrgPerson`: `uid=jim`
* add `organizationalUnit`: `ou=groups`
  * manually add the following groups or use the scripts below
  * add `groupOfNames`: `cn=admins` (add `uid=ronan` as a member)
  * add `groupOfNames`: `cn=biusers` (add `uid=jim` as a member)

```
cat <<EOF > add-admins.ldif
dn: cn=admins,ou=groups,dc=ldap,dc=mongodb,dc=local
objectClass: groupofnames
cn: admins
description: All users
# add the group members all of which are
# assumed to exist under people
member: uid=ronan,ou=people,dc=ldap,dc=mongodb,dc=local
EOF

ldapadd -x -D cn=admin,dc=ldap,dc=mongodb,dc=local -W -f add-admins.ldif
```

```
cat <<EOF > add-biusers.ldif
dn: cn=biusers,ou=groups,dc=ldap,dc=mongodb,dc=local
objectClass: groupofnames
cn: biusers
description: All users
# add the group members all of which are
# assumed to exist under people
member: uid=jim,ou=people,dc=ldap,dc=mongodb,dc=local
EOF

ldapadd -x -D cn=admin,dc=ldap,dc=mongodb,dc=local -W -f add-biusers.ldif
```

### Set LDAP passwords

```
ldappasswd -H ldap://ldap.mongodb.local:389 -S -W -D "cn=admin,dc=ldap,dc=mongodb,dc=local" -x "uid=ronan,ou=people,dc=ldap,dc=mongodb,dc=local"
ldappasswd -H ldap://ldap.mongodb.local:389 -S -W -D "cn=admin,dc=ldap,dc=mongodb,dc=local" -x "uid=jim,ou=people,dc=ldap,dc=mongodb,dc=local"
```

(We'll assume passwords are set to the same value as the username below)

### Test membership

```
ldapsearch -x -LLL -H ldap://ldap.mongodb.local:389 -b uid=ronan,ou=people,dc=ldap,dc=mongodb,dc=local dn memberof
ldapsearch -x -LLL -H ldap://ldap.mongodb.local:389 -b uid=jim,ou=people,dc=ldap,dc=mongodb,dc=local dn memberof
```

Based on the `member` field in the `admins` group above, the first command should return a `memberOf` attribute:

```
dn: uid=ronan,ou=people,dc=ldap,dc=mongodb,dc=local
memberOf: cn=admins,ou=groups,dc=ldap,dc=mongodb,dc=local
```

The second command should return a `memberOf` attribute:

```
dn: uid=jim,ou=people,dc=ldap,dc=mongodb,dc=local
memberOf: cn=biusers,ou=groups,dc=ldap,dc=mongodb,dc=local
```

## LDAP & MongoDB...

### Install MongoDB

Use the appropriate method (`apt-get`, `yum`, `m`, etc) to install MongoDB Enterprise Edition on your client machine.

### Create mongod.conf file pointing to our new LDAP server

Create the following mongod.conf file on your client machine:

```
cat <<EOF > mongod.conf
storage:
  dbPath: /data/

processManagement:
  fork: true

systemLog:
  destination: file
  path: /data/mongodb.log

net:
  port: 27017
  bindIp: 0.0.0.0

security:
#   authorization: "enabled"
   ldap:
      transportSecurity: none
      servers: "ldap.mongodb.local:389"
      bind:
         queryUser: "cn=admin,dc=ldap,dc=mongodb,dc=local"
         queryPassword: "admin"
      userToDNMapping:
         '[
            {
               match : "(.+)",
               ldapQuery: "dc=ldap,dc=mongodb,dc=local??sub?uid={0}"
            }
         ]'
      authz:
         queryTemplate: "{USER}?memberOf?base"
setParameter:
   authenticationMechanisms: "PLAIN"
EOF
```

### Test using `mongoldap`

```
mongoldap --config mongod.cfg --user ronan
mongoldap --config mongod.cfg --user jim
```

Ignore any FAIL's relating to binding with a plaintext password over a non-TLS connection (setting up an LDAPS server is left as an exercise for the reader...)

The first command returns the following role:

```
* cn=admins,ou=groups,dc=ldap,dc=mongodb,dc=local
```

The second command returns the following role:

```
* cn=biusers,ou=groups,dc=ldap,dc=mongodb,dc=local
```

### Start MongoDB & Connect

```
mongod -f mongod.conf
mongo
```

### Create admin & readonly roles

```
use admin
db.createRole({role: "cn=admins,ou=groups,dc=ldap,dc=mongodb,dc=local", privileges:[], roles: ["root"]})
db.createRole({role: "cn=biusers,ou=groups,dc=ldap,dc=mongodb,dc=local", privileges: [{ resource: { db: "reporting", collection: "webstats" }, actions: [ "find" ] }], roles: []})
```

Any member of the `cn=admins` group will be a MongoDB root user, while any member of the `cn=biusers` gropu will only have read access (`find`) to the `reporting.webstats` namespace.

### Test new roles

```
use $external

db.auth({mechanism: "PLAIN", user: "ronan", pwd:"ronan"})
db.runCommand({connectionStatus:1})

db.auth({mechanism: "PLAIN", user: "jim", pwd:"jim"})
db.runCommand({connectionStatus:1})
```

(replace passwords, `pwd` fields, with the ones you defined earlier)

The `db.auth()` command in both cases should succeed (returning `1`). Only the first `connectionStatus` will contain the `root@admin` role:

```
{
	"authInfo" : {
		...
		"authenticatedUserRoles" : [
			{
				"role" : "root",
				"db" : "admin"
			},
		...
	}
}
```

## TLS

### Install GNU TLS binaries & SSL Cert packages

```
sudo apt install gnutls-bin ssl-cert
```

### Generate CA & self-signed Certificate for LDAPS server

We're going to use our own CA and self-signed certs for this demo...

#### Create Certificate Authority

Note: Replace `cn` with a suitable name if required:

```
sudo sh -c "certtool --generate-privkey > /etc/ssl/private/cakey.pem"

sudo sh -c "cat <<EOF > /etc/ssl/ca.info"
cn = MongoDB
ca
cert_signing_key
EOF

sudo certtool --generate-self-signed --load-privkey /etc/ssl/private/cakey.pem --template /etc/ssl/ca.info --outfile /etc/ssl/certs/cacert.pem
```

#### Create certificate

**IMPORTANT**: Replace `cn` value with the FQDN for the LDAP server, especially if you want to connect from e.g. Atlas where you may have less control over DNS settings!

(You many also want to update the `organization` field too).

```
sudo certtool --generate-privkey --bits 1024 --outfile /etc/ssl/private/mongodb_slapd_key.pem

# MUST use public hostname of LDAP server for cn!!
sudo sh -c "cat <<EOF > /etc/ssl/mongodb.info"
organization = MongoDB
cn = ec2-ww-xx-yy-zz.abc.compute.amazonaws.com
tls_www_server
encryption_key
signing_key
expiration_days = 3650
EOF

sudo certtool --generate-certificate --load-privkey /etc/ssl/private/mongodb_slapd_key.pem --load-ca-certificate /etc/ssl/certs/cacert.pem --load-ca-privkey /etc/ssl/private/cakey.pem --template /etc/ssl/mongodb.info --outfile /etc/ssl/certs/mongodb_slapd_cert.pem
```

#### Update file ownership, permissions & group membership

```
sudo chgrp openldap /etc/ssl/private/mongodb_slapd_key.pem
sudo chmod 0640 /etc/ssl/private/mongodb_slapd_key.pem
sudo gpasswd -a openldap ssl-cert
```

### Update LDAP Config

```
cat <<EOF > certinfo.ldif
dn: cn=config
add: olcTLSCACertificateFile
olcTLSCACertificateFile: /etc/ssl/certs/cacert.pem
-
add: olcTLSCertificateFile
olcTLSCertificateFile: /etc/ssl/certs/mongodb_slapd_cert.pem
-
add: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/ssl/private/mongodb_slapd_key.pem
EOF

sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f certinfo.ldif
```

### Add 'ldaps' as a valid endpoint

`SLAPD_SERVICES` string in the `slapd` config file needs to include `ldaps:///`, e.g.

```
sudo vi /etc/default/slapd
```

Then replace this line:

```
SLAPD_SERVICES="ldap:/// ldapi:///"
```

with this line:

```
SLAPD_SERVICES="ldap:/// ldaps:/// ldapi:///"
```

### Restart Service

```
sudo systemctl restart slapd.service
```

### Validate

```
openssl s_client -connect ldap.mongodb.local:636 -showcerts -state -CAfile /etc/ssl/certs/mongodb_slapd_cert.pem
```

```
ldapsearch -H ldaps://ldap.mongodb.local:636 -x -b "dc=ldap,dc=mongodb,dc=local" -LLL
```

Note: You may need to add this to your `.ldaprc` file for the TLS handshake to work . (I need to do this on my Mac for example. If anyone knows why exactly I'd be very interested to hear!)

```
cat <<EOF > ~/.ldaprc
TLS_CACERT      /etc/ssl/certs/mongodb_slapd_cert.pem
EOF
```

## MongoDB Atlas

**Work In Progress**

Add `0.0.0.0/0` to the firewall rules on the LDAP server to allow MongoDB Atlas (and the MongoDB clusters) to connect.

### Enable LDAP Authentication & Authorization in Atlas

Navigate to the relevant MongoDB Atlas Project, then select the `Security` tab, followed by the `Enterprise Security` tab.

Enable the `LDAP Authentication` and `LDAP Authorization` options via the rocker.

Fill out the following entries (based on the configuration we created above):

**Configure Your LDAP Server**

`Server Hostname`: _FQDN of remote LDAPS server_

`Server Port`: `636`

`Bind Username`: `cn=admin,dc=ldap,dc=mongodb,dc=local`

`Bind Password`: _Bind Username's password_

`User To DN Mapping`: `[{ match : "(.+)", ldapQuery: "dc=ldap,dc=mongodb,dc=local??sub?(uid={0})" }]`

`CA Root Certificate`: _Contents of the CA file, e.g. `/etc/ssl/certs/cacert.pem`_

**Configure LDAP Authorization**

`Query Template`: `{USER}?memberOf?base`

## Acknowledgements

Based on: [https://technicalnotes.wordpress.com/2014/04/19/openldap-setup-with-memberof-overlay/]() (with some minor changes, e.g. changing `HDB` to `MDB` for the `olcDatabase`)