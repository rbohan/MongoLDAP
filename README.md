# MongoDB and OpenLDAP

This guide shows you how to install and configure a secure OpenLDAP service on Ubuntu 18.04 running in AWS. The service supports LDAP and optionally LDAPS.

In addition we provide instructions on how to configure MongoDB Enterprise and/or MongoDB Atlas to use this LDAP service to manage user authentication and authorization.

### OpenLDAP

Follow [these steps](AWS.md) to create a suitable/small AWS instance where we can deploy our OpenLDAP service.

Follow [these steps](SetupLDAP.md) to deploy a basic OpenLDAP service (LDAP only).

This LDAP server is configured with two users and two groups for demonstration purposes.

Follow [these optional steps](SetupLDAPS.md) if you wish to reconfigure your LDAP server to support LDAPS. These steps are _required_ if you plan to use your LDAP server with MongoDB Atlas (option 2 below).

### Option 1 - MongoDB Enterprise

Follow [these steps](MongoDB.md) to install and configure a MongoDB Enterprise server to use our OpenLDAP server as an LDAP endpoint.

### Option 2 - MongoDB Atlas

Follow [these steps](Atlas.md) to configure MongoDB Atlas to use our OpenLDAP server as an LDAPS endpoint.

## Acknowledgements & References

### MongoDB Documentation References

* [LDAP Proxy Authentication](https://docs.mongodb.com/manual/core/security-ldap/)
* [LDAP Authorization](https://docs.mongodb.com/manual/core/security-ldap-external/)
* [Configuration File Options for LDAP](https://docs.mongodb.com/manual/reference/configuration-options/#security-ldap-options)
* [mongoldap](https://docs.mongodb.com/manual/reference/program/mongoldap/)

### MongoDB Atlas Documentation References

* [Set up User Authentication and Authorization with LDAP](https://docs.atlas.mongodb.com/security-ldaps/)

### MongoDB Blog posts

* [How to Configure LDAP Authentication for MongoDB](https://www.mongodb.com/blog/post/how-to-configure-LDAP-authentication-for-mongodb)

### External Guides

* [OpenLDAP setup](https://technicalnotes.wordpress.com/2014/04/19/openldap-setup-with-memberof-overlay/) (with some minor changes, e.g. changing `HDB` to `MDB` for the `olcDatabase`)
* [TLS setup](https://help.ubuntu.com/lts/serverguide/openldap-server.html) (see the 'TLS' section)
