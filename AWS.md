## Deploy AWS Instance

Follow these steps to create a small AWS instance where we can install our OpenLDAP service.

### Create VM:

Launch an instance in AWS (all settings default unless otherwise noted):

1. AMI:
  * `Ubuntu Server 18.04 LTS (HVM), SSD Volume Type`
* Instance Type:
  * `t2.micro`
* Instance Details:
  * Network: _Select a VPC which is peered with your Atlas cluster. Create if necesssary._
  * Auto-assign Public IP: _Enable if required or associate an Elastic IP later._
* Storage:
  * _Defaults._
* Tags:
  * _Add as appropriate._
* Security Groups:
  * _Create or select a Security Group with access to port 636 from your VPC peered Atlas CIDR block._
  * _Enable access to port 22 for SSH access._
  * _Optionally enable access to port 636 and 389 from your laptop/local machine for testing purposes._
* Review Instance Launch:
  * _Verify your settings are correct, click Launch and specify your key pair as normal._

**Note**: Using a VPC peered connection is not required but if this is not in place you will likely need to expose `0.0.0.0` so the Atlas cluster can connect to your LDAP server over the external/public internet address. And if using self-signed certs you will also likely need to rename the AWS instance based on its external FQDN to ensure the certificates for the LDAPS service can be exchanged and validated.

### Connect to your Instance

`ssh -i <pem-file> ubuntu@<host>.compute.amazonaws.com`
