## Add LDAPS Support to OpenLDAP

These instructions show you how to reconfigure your OpenLDAP server to support LDAPS, aka LDAP over SSL. LDAPS is required by MongoDB Atlas (and recommended, though optional, for MongoDB Enterprise Server).

Before we can enable LDAPS on our LDAP Server however we need an X.509 certificate so let's get to work...

### Install GNU TLS binaries & SSL Cert packages

Install the various packages on the guest as follows:

```
sudo apt install -y gnutls-bin ssl-cert
```

### Generate CA & self-signed Certificate for LDAPS server

We're going to use our own CA and self-signed certs for this demo.

#### Create Certificate Authority

```
sudo sh -c "certtool --generate-privkey > /etc/ssl/private/cakey.pem"

sudo sh -c "cat > /etc/ssl/ca.info" <<EOF
cn = MongoDB
ca
cert_signing_key
EOF

sudo certtool --generate-self-signed --load-privkey /etc/ssl/private/cakey.pem --template /etc/ssl/ca.info --outfile /etc/ssl/certs/cacert.pem
```

#### Create Server Certificate

```
sudo certtool --generate-privkey --bits 1024 --outfile /etc/ssl/private/mongodb_slapd_key.pem

sudo sh -c "echo cn = `hostname -f` > /etc/ssl/mongodb.info"
sudo sh -c "cat >> /etc/ssl/mongodb.info" <<EOF
organization = MongoDB
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

**Note**: This `ldapmodify` command appears to fail the first time and I'm not sure why. If you get an error (e.g. **"ldap_modify: Other (e.g., implementation specific) error (80)"**), try restarting the service and re-issuing the command as follows:

```
sudo systemctl restart slapd.service
sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f certinfo.ldif
```

Upon successful completion of this command the last line of the output should read:

```
modifying entry "cn=config"
```

### Add 'ldaps' as a valid endpoint

MongoDB only support LDAPS so the `SLAPD_SERVICES` string in the `slapd` config file (`/etc/default/slapd`) needs to include `ldaps:///`, e.g. this line:

```
SLAPD_SERVICES="ldap:/// ldapi:///"
```

should look like this:

```
SLAPD_SERVICES="ldap:/// ldaps:/// ldapi:///"
```

Let's do that:

```
sudo cp /etc/default/slapd /etc/default/slapd.orig
sudo sh -c "sed 's|\(^SLAPD_SERVICES.*\)\(ldapi:.*\)|\1ldaps:/// \2|' /etc/default/slapd.orig > /etc/default/slapd"
```

Check the updated contents of `/etc/default/slapd` to make sure everything's okay.

### Restart Service (again)

```
sudo systemctl restart slapd.service
```

### Validate

#### Check LDAP server status

We can check the LDAP server status by running:

```
sudo systemctl status slapd.service
```

This should output details including the command line used to start the server, similar to the following:

```
   CGroup: /system.slice/slapd.service
           └─3150 /usr/sbin/slapd -h ldap:/// ldaps:/// ldapi:/// -g openldap -u openldap -F /etc/ldap/slapd.d
```

Note the presence of `ldap:///`, `ldaps:///` and `ldapi:///`

#### Check using OpenSSL commands

We can check that we have some basic connectivity by using an OpenSSL command line tool:

```
openssl s_client -connect `hostname`:636 -showcerts -state -CAfile /etc/ssl/certs/mongodb_slapd_cert.pem
```

This should generate a lot of text and you'll need to `<CTRL>-C` back to the prompt. Near the top along with 'SSL_connect' messages you should see this line:

```
depth=1 CN = MongoDB
```

If you don't see this output go back to the 'Update LDAP Config' section above, restart the server and try re-applying the modification as per the note.

#### Check using `ldapsearch`

To test with `ldapsearch` you'll first need to run this for the TLS handshake to work:

```
echo "TLS_REQCERT allow" > ~/.ldaprc
```

**Note**: You'll need to run this command from any client machine where you intend to run `ldapsearch`.

And now a quick test with `ldapsearch`, noting the use of the `ldaps` URI scheme:

```
ldapsearch -H ldaps:/// -x -b "dc=ldap,dc=mongodb,dc=local" -LLL dn
```

All going well you should see all of the DN's listed:

```
dn: dc=ldap,dc=mongodb,dc=local
dn: cn=admin,dc=ldap,dc=mongodb,dc=local
...
```

#### If something goes wrong

If something goes wrong with any of the validation tests it is likely to be related to the registration of the certs, so try the following steps before attempting the validation step again:

```
sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f certinfo.ldif
sudo systemctl restart slapd.service
```
