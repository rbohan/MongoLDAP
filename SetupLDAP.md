## Install & Configure LDAP

These instructions show you how to deploy a simple OpenLDAP service with 2 users and 2 groups configured.

### Update apt & install LDAP utils

**Note**: We'll need to specify a password during install but we'll be overwriting this in a moment so just fill in a temporary value for now.

```
sudo apt-get update -y
sudo apt install -y slapd ldap-utils ldapscripts
```

Let's reconfigure to our liking:

```
sudo dpkg-reconfigure slapd
```

* Omit OpenLDAP server configuration? `no`
* DNS domain name: `ldap.mongodb.local`
* Organization name: `mongodb`
* Administrator password: _choose a secure password here_
* Confirm password: _repeat the secure password here_
* Database backend to use: `MDB`
* Do you want the database to be removed when slapd is purged? `no`
* Move old database? `yes`

**Note**: Make sure you specify a password. Do not leave it blank as that will cause problems later.

This creates 2 Distinguished Names or DN's:

```
dn: dc=ldap,dc=mongodb,dc=local
dn: cn=admin,dc=ldap,dc=mongodb,dc=local
```

The first is the top level `organization` entity for our base DN `ldap.mongodb.local` while the second is an `organizationalRole` & `simpleSecurityObject` entity representing the the `admin` user.

This `admin` user can be used as the [`bind`](https://ldap.com/the-ldap-bind-operation/) user to update the directory if required.

**Note**: LDAPv3 supports anonymous simple authentication so you will be able to _query_ this LDAP directory with or without a bind user (writes will require an authenticated, aka bind, user).

Validate the configuration with the following command from the Ubuntu shell:

```
ldapsearch -H ldap:/// -x -b "dc=ldap,dc=mongodb,dc=local" -LLL dn
```

This command connects to the LDAP server running on localhost (`-H ldap:///`), uses simple authentication (`-x`), queries our base DN (`-b "dc=ldap,dc=mongodb,dc=local"`), outputs in LDIF format without comments or version (`-LLL`) and returns just the DN attributes (`dn`). We are using anonymous authentication because we haven't supplied a 'bind DN' (there is no `-D` option). The output should match the 2 DN's listed above.

### Install memberof overlay

This provides the reverse lookup capabilities such that for a `member` listed in a group, when you query the `memberOf` attribute on that member's DN you get the DN of the group back - exactly what we need to map users to roles in MongoDB!

**Note**: Other LDAP servers, as as Microsoft's ActiveDirectory, implement this functionality natively.

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

### Populate the LDAP directory

Under `dc=ldap,dc=mongodb,dc=local` we will create the following entries where we will store our users and groups respectively:

* `organizationalUnit`: `ou=users`
* `organizationalUnit`: `ou=groups`

Under the `ou=users` entry we will create the following users:

  * `inetOrgPerson`: `uid=jane`
  * `inetOrgPerson`: `uid=john`

Under the `ou=groups` entry we will create the following groups:

  * `groupOfNames`: `cn=admins` (adding `uid=jane` as a member)
  * `groupOfNames`: `cn=users` (adding `uid=jane` and `uid=john` as members)

In other words, Jane will be a member of both the admins and users groups, while John will be a member of just the users group.

Note that we will not need to update Jane's or John's LDAP records. Instead we update the corresponding group entry by adding a `member` attribute containing the DN of each user in that group. This is why we need the `memberof` overlay to perform the reverse lookup - to see which groups a given user is in based on attributes stored in the group entity itself.

**Note**: For the commands below which update the LDAP tree structure you will need to specify the password for the `admin` user chosen earlier whenever you see the `Enter LDAP Password:` prompt.

#### Create the 'Users' hierarchy

##### Create parent node `ou=users`

```
cat <<EOF > add-users-ou.ldif
dn: ou=users,dc=ldap,dc=mongodb,dc=local
objectClass: organizationalunit
EOF

ldapadd -x -D cn=admin,dc=ldap,dc=mongodb,dc=local -W -f add-users-ou.ldif
```

##### Create leaf node `uid=jane`

```
cat <<EOF > add-jane.ldif
dn: uid=jane,ou=users,dc=ldap,dc=mongodb,dc=local
objectClass: inetorgperson
cn: jane
sn: doe
EOF

ldapadd -x -D cn=admin,dc=ldap,dc=mongodb,dc=local -W -f add-jane.ldif
```

##### Create leaf node `uid=john`

```
cat <<EOF > add-john.ldif
dn: uid=john,ou=users,dc=ldap,dc=mongodb,dc=local
objectClass: inetorgperson
cn: john
sn: doe
EOF

ldapadd -x -D cn=admin,dc=ldap,dc=mongodb,dc=local -W -f add-john.ldif
```

#### Create the 'Groups' hierarchy

##### Create parent node `ou=groups`

```
cat <<EOF > add-groups-ou.ldif
dn: ou=groups,dc=ldap,dc=mongodb,dc=local
objectClass: organizationalunit
EOF

ldapadd -x -D cn=admin,dc=ldap,dc=mongodb,dc=local -W -f add-groups-ou.ldif
```

##### Create leaf node `cn=admins`, adding Jane

```
cat <<EOF > add-admins.ldif
dn: cn=admins,ou=groups,dc=ldap,dc=mongodb,dc=local
objectClass: groupofnames
cn: admins
description: All users
# add the group members all of whom are
# assumed to exist under users
member: uid=jane,ou=users,dc=ldap,dc=mongodb,dc=local
EOF

ldapadd -x -D cn=admin,dc=ldap,dc=mongodb,dc=local -W -f add-admins.ldif
```

##### Create leaf node `cn=users`, adding Jane and John

```
cat <<EOF > add-users.ldif
dn: cn=users,ou=groups,dc=ldap,dc=mongodb,dc=local
objectClass: groupofnames
cn: users
description: All users
# add the group members all of whom are
# assumed to exist under users
member: uid=john,ou=users,dc=ldap,dc=mongodb,dc=local
member: uid=jane,ou=users,dc=ldap,dc=mongodb,dc=local
EOF

ldapadd -x -D cn=admin,dc=ldap,dc=mongodb,dc=local -W -f add-users.ldif
```

#### Set user passwords

Run the following commands to set the password for both users, following the prompts as required:

```
ldappasswd -H ldap:/// -S -W -D "cn=admin,dc=ldap,dc=mongodb,dc=local" -x "uid=jane,ou=users,dc=ldap,dc=mongodb,dc=local"
```

```
ldappasswd -H ldap:/// -S -W -D "cn=admin,dc=ldap,dc=mongodb,dc=local" -x "uid=john,ou=users,dc=ldap,dc=mongodb,dc=local"
```

#### Test group membership

Verify that Jane is in both the `cn=users` and the `cn=admins` groups by running the following command:

```
ldapsearch -x -LLL -H ldap:/// -b uid=jane,ou=users,dc=ldap,dc=mongodb,dc=local memberof
```

The output should be:

```
dn: uid=jane,ou=users,dc=ldap,dc=mongodb,dc=local
memberOf: cn=admins,ou=groups,dc=ldap,dc=mongodb,dc=local
memberOf: cn=users,ou=groups,dc=ldap,dc=mongodb,dc=local
```

Verify that John is in the `cn=users` group by running the following command:

```
ldapsearch -x -LLL -H ldap:/// -b uid=john,ou=users,dc=ldap,dc=mongodb,dc=local memberof
```

The output should be:

```
dn: uid=john,ou=users,dc=ldap,dc=mongodb,dc=local
memberOf: cn=users,ou=groups,dc=ldap,dc=mongodb,dc=local
```

### Congratulations!

Congratulations, at this point you have a fully functional LDAP server containing a valid user hierarchy!

You can now use tools such as [ApacheDirectoryStudio](https://directory.apache.org/studio/) to connect from your client machine (assuming your Security Groups allow access to port 389). Be sure to bind as the 'admin' user (`cn=admin,dc=ldap,dc=mongodb,dc=local`) using 'Simple Authentication' in order to be able to make modifications.
