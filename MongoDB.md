## LDAP & MongoDB Enterprise Server

The following instructions detail how to manually update the MongoDB configuration file to enable LDAP/LDAPS support.

### Install MongoDB

Use the appropriate method (`apt-get`, `yum`, `m`, etc) to install MongoDB Enterprise Edition on your client machine.

### Create mongod.cfg file pointing to our new LDAP server

Create the following mongod.cfg file on your host machine:

```
cat <<EOF > mongod.cfg
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
      servers: "<LDAP server>:389"
      bind:
         queryUser: "cn=admin,dc=ldap,dc=mongodb,dc=local"
         queryPassword: <password>
      userToDNMapping:
         '[
            {
               match : "(.+)",
               ldapQuery: "dc=ldap,dc=mongodb,dc=local??sub?(uid={0})"
            }
         ]'
      authz:
         queryTemplate: "{USER}?memberOf?base"
setParameter:
   authenticationMechanisms: "PLAIN"
EOF
```

This defines a fairly standard confuration but with LDAP enabled. Focusing on the `security` section:

* We are not enabling authorization at this point (can be enabled later if required).
* We have actively disabled transport encryption meaning that all network communication occurs in plaintext (the default is to use transport encryption, aka TLS). Do not use this setting in a production environment!
* We have specified the FQDN of our LDAP server (including port). If deployed in the same or a peered VPC you should use the FQDN of the internal hostname. If you wish to use LDAPS ensure you have performed the optional setup steps and replace the port with 636.
* We are binding with the `admin` user (using the full DN).
* We have specified a single `userToDNMapping` rule which takes any input and converts it into a sub-tree LDAP query, looking for users with that string as their `uid`.
* We've specified a base LDAP query as the authorization template, taking the DN we matched via the `userToDNMapping` query and retrieving the `memberOf` attribute.
* We are using the `PLAIN` authentication mechanism, which is how we tell the system we're using an LDAP server for authentication purposes.

### Test using `mongoldap`

`mongoldap` is a command line tool supplied with MongoDB Enterprise which can validate the mongod configuration file without requiring you to start/restart a `mongod` process.

Test our configuration file using the two users we have defined. Note that we use a simple value for each user which will be mapped to a DN based on the `userToDNMapping` rules.

```
mongoldap --config mongod.cfg --user jane
mongoldap --config mongod.cfg --user john
```

Ignore any FAIL's relating to binding with a plaintext password over a non-TLS connection.

**Note**: If you have performed the optional steps to configure LDAPS you can change the `servers` entry to include the LDAPS port (636) instead of the LDAP port (389). This will remove the FAIL messages related to the plaintext password highlighted above.

The first command (for `uid=jane`) returns the following roles:

```
* cn=admins,ou=groups,dc=ldap,dc=mongodb,dc=local
* cn=users,ou=groups,dc=ldap,dc=mongodb,dc=local
```

The second command (for `uid=john`) returns the following role:

```
* cn=users,ou=groups,dc=ldap,dc=mongodb,dc=local
```

### Start MongoDB & Connect

Start the `mongod` process (make sure you are using the Enterprise version!):

```
mongod -f mongod.cfg
```

Check the log file (`/data/mongodb.log`) for a successful start. In particular look for the `initandlisten` line specifying the `options`, making sure they match the options in the configuration file. Also check for a subsequent `initandlisten` line containing the following text:

```
Server configured with LDAP Authorization. Spawned $external user cache invalidator.
```

Once the server is running you can connect via the mongo shell (we don't have authorization enabled at this point so we can just connect):

```
mongo
```

### Create admin & readonly roles

In order to allow users defined in our LDAP server to connect to the database we have to create `roles` in the database with associated privileges. We do *not* have to create any users directly. Instead the roles defined in the database map to the groups in our LDAP server and users which are members of these groups will be granted the privileges defined by the associated MongoDB role.

We will create 2 roles:

1. For the `cn=admins` group, giving those users full root permission.
2. For the `cn=users` group, giving those users read-only permissions.

```
use admin
db.createRole({role: "cn=admins,ou=groups,dc=ldap,dc=mongodb,dc=local", privileges:[], roles: ["root"]})
db.createRole({role: "cn=users,ou=groups,dc=ldap,dc=mongodb,dc=local", privileges: [], roles: ["readAnyDatabase"]})
```

In this example any member of the `cn=admins` group will be a MongoDB root user, while any member of the `cn=users` group will only be granted read access.

### Test new roles

```
use $external

db.auth({mechanism: "PLAIN", user: "jane", pwd:<jane's password>})
db.runCommand({connectionStatus:1})

db.auth({mechanism: "PLAIN", user: "john", pwd:"<john's password>"})
db.runCommand({connectionStatus:1})
```

(replace passwords, i.e., the values for the `pwd` fields, with the ones you defined earlier).

The `db.auth()` command in both cases should succeed (returning `1`).

User `jane` is the only one that contains the `root@admin` role:

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

Both users should include the `readAnyDatabase@admin` role:

```
{
	"authInfo" : {
		...
		"authenticatedUserRoles" : [
			{
				"role" : "readAnyDatabase",
				"db" : "admin"
			},
		...
	}
}
```
