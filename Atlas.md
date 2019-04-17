## MongoDB Atlas

Here we provide instructions on how to configure MongoDB Atlas to use our LDAP server for user authentication and authorization.

**Note**: MongoDB Atlas communicates with any LDAP server using LDAPS (LDAP over SSL). If you haven't already done so please reconfigure your OpenLDAP server to support LDAPS (per [these instructions](SetupLDAPS.md)).

### Set up firewall rules

Make sure that the Security Group for the peered VPC connection is listed in the IP Whitelist in Atlas. See [the online docs](https://docs.atlas.mongodb.com/security-whitelist/) for details.

**Note**: If you are not using a VPC peered connection you will need to open up the Security Group for your OpenLDAP instance to allow incoming connections from anywhere (0.0.0.0/0)

### Enable LDAP Authentication & Authorization in Atlas

Navigate to the relevant MongoDB Atlas Project, then select the `Security` tab, followed by the `Enterprise Security` tab.

Enable the `LDAP Authentication` and `LDAP Authorization` options via the rocker. Remember that you'll need to deploy a dedicated cluster running MongoDB 3.4 or higher before you can enable LDAP auth.

Fill out the following entries (based on the configuration we created above):

#### Configure Your LDAP Server

![Authentication details](AtlasAuthN.png)

* `Server Hostname`: _FQDN of remote LDAPS server_
* `Server Port`: `636`
* `Bind Username`: `cn=admin,dc=ldap,dc=mongodb,dc=local`
* `Bind Password`: _Bind Username's password_
* `User To DN Mapping`: `[{ match : "(.+)", ldapQuery: "ou=users,dc=ldap,dc=mongodb,dc=local??sub?(uid={0})" }]`
* `CA Root Certificate`: _Contents of the CA file, e.g. `/etc/ssl/certs/cacert.pem`_

If using a peered VPC connection you should specify the FQDN using the internal hostname of the OpenLDAP server.

The 'User To DN Mapping' setting above defines a single rule which takes the username provided when the user connects to the Atlas cluster and performs a sub-tree search under the `ou=users` branch, searching for users where the `uid` attribute equals the username provided.

#### Configure LDAP Authorization

![Authorization details](AtlasAuthZ.png)

* `Query Template`: `{USER}?memberOf?base`

This sets up a query template which takes the matched user (from the 'User to DN Mapping' rules above) and performs a `memberOf` search on just that DN, i.e., a base search. This will return the group membership of that user.

#### Validate and Save

Once all the values are set click the 'Validate and Save' button at the bottom of the screen. All going well MongoDB Atlas should process the values and after a short pause (as it connects to the LDAP server) it should indicate a successful configuration change at the top of the page.

### Create LDAP Roles in Atlas

Now that we have Atlas configured to point to our LDAP Server we need to map LDAP groups to roles in MongoDB.

![Adding an Admin group](AtlasAddAdminGroup.png)

Switch to the `MongoDB Users` tab within the `Security` tab for your LDAP enabled project.

To add a new LDAP group, click the `Add New User or LDAP Group` button on the top right and select the `LDAP Group` method.

In the `LDAP Authorization` field enter the full DN of the respective LDAP group, e.g. `ou=admins,dc=ldap,dc=mongodb,dc=local`

Select the `User Privileges` as normal for any members of this LDAP group, e.g. `Atlas admin` or choose advanced options by clicking the `Show Advanced Options` and filling out the relevant details.

The result should look something like the following:

![LDAP groups in Atlas](AtlasLDAPGroups.png)

Here we have mapped members of the `ou=admins` LDAP group to the Atlas 'admin' role, while members of the `ou=users` LDAP group have been mapped to the Atlas 'read any database' role.

Verify your configuration by connecting to your Atlas cluster as a user who is a member of one of the defined LDAP groups, e.g. as the user `Jane` (who will get mapped to the DN `uid=jane,ou=users,dc=ldap,dc=mongodb,dc=local`). The groups this user is a member of (a result of running the `ldapQuery` template) define which roles that user has in the Atlas cluster based on the `LDAP Group` mappings defined above.
