# LDAP CONFIGURATION ON SUSE LINUX

## CONFIGURING SYSTEM AS LDAP SERVER

### Installing LDAP Package

```shell
zypper -n install openldap2*
```

### Changing parameter's value in Openldap file

Make the Changes as follows.

```shell
vi /etc/sysconfig/openldap

OPENLDAP_START_LDAPI= “yes”
OPENLDAP_START_LDAPS= “yes”
OPENLDAP_CONFIG_BACKEND= “ldap”

```

### Taking Backup of slapd.conf file

```shell
mv /etc/openldap/slapd.conf /etc/openldap/slapd.conf.org
```

### Creating Empty slapd.conf file

```shell
touch /etc/openldap/slapd.conf
```
### Before Running slaptest command do the following

```shell
rm -rf /etc/openldap/slapd.d/*
```

### Running slaptest Command

```shell
slaptest -f /etc/openldap/slapd.conf -F /etc/openldap/slapd.d
```

### Changing values in olcdatabase={0}conf.ldif

```shell
vi /etc/openldap/slapd.d/cn=config/olcDatabase\={0}config.ldif
#Remove

# CRC32 xxxxxxxx

#Change

olcAccess: {0}to * by dn.exact=gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth manage by * break
```

### Changing Permissions and Ownerships

```shell
chown -R ldap. /etc/openldap/slapd.d
chmod -R 700 /etc/openldap/slapd.d
chown -R ldap. /var/lib/ldap
chmod -R 700 /var/lib/ldap
```

### Permanently starting the slapd service

```shell
systemctl start slapd
systemctl enable slapd
```

### Admin password generation

```shell
slappasswd
New password:
Re-enter new password:
{SSHA}xxxxxxxxxxxxxxxxxxxxxxxx
```

### Creating Admin ldif file

```shell
vi chrootpw.ldif
dn: olcDatabase={0}config,cn=config
changetype: modify
add: olcRootPW
olcRootPW: {SSHA}xxxxxxxxxxxxxxxxxxxxxxxx #give the SSHA password here
```

### Adding admin entry to LDAP DB

```shell
ldapadd -Y EXTERNAL -H ldapi:/// -f chrootpw.ldif
```

### ADDING SCHEMAS

```shell
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/core.ldif

ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif

ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif

ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
```

### Creating the Backend file

Replace dc=,dc= with your own Suffix.
And also give appropriate path for the modules.

```shell
vi backend.ldif

dn: cn=module,cn=config
objectClass: olcModuleList
cn: module
olcModulepath: /usr/lib64/openldap
olcModuleload: back_hdb

dn: olcDatabase=hdb,cn=config
objectClass: olcDatabaseConfig
objectClass: olcHdbConfig
olcDatabase: {1}hdb
olcDbDirectory: /var/lib/ldap
olcDbIndex: objectClass eq,pres
olcDbIndex: ou,cn,mail,surname,givenname eq,pres,sub
olcSuffix: dc=example,dc=com
olcRootDN: cn=Manager,dc=example,dc=com
olcRootPW: {SSHA}xxxxxxxxxxxxxxxxxxxxxxxx
olcAccess: {0}to attrs=userPassword,shadowLastChange by 
  dn="cn=Manager,dc=example,dc=com" write by anonymous auth by self write by * none
olcAccess: {1}to dn.base="" by * read
olcAccess: {2}to * by dn="cn=Manager,dc=example,dc=com" write by * read

```

### Adding the Backend file

```shell
ldapadd -Y EXTERNAL -H ldapi:/// -f backend.ldif
```

### Creating the Basedomain file

Replace dc=,dc= with your own Suffix.

```shell
vi basedomain.ldif

dn: dc=example,dc=com
objectClass: top
objectClass: dcObject
objectclass: organization
o: example com
dc: example

dn: cn=Manager,dc=example,dc=com
objectClass: organizationalRole
cn: Manager
description: Directory Manager

dn: ou=People,dc=example,dc=com
objectClass: organizationalUnit
ou: People

dn: ou=Group,dc=example,dc=com
objectClass: organizationalUnit
ou: Group
```

### Adding the Basedomain file

Make sure you have content in slapd.conf file, And database module selected as hbd and included modules and schemas by removing the hash (#) before it in config file. (try moving back slapd.conf.org to slapd.conf and make the changes)

```shell
ldapadd -x -D cn=Manager,dc=example,dc=com -W -f basedomain.ldif
```

### Creating Certificates

```shell
cd /etc/ssl/private
opensslgenrsa -des3 -out server.key 2048
```

### Removing Passphrase

```shell
opensslrsa -in server.key -out server.key
```

### Information for Certificate

Enter the details after running the following command.

```shell
opensslreq -new -days 3650 -key server.key -out server.csr
```

### Requesting key and changing permission

```shell
openssl x509 -in server.csr -out server.crt -req -signkey server.key -days 3650
```

### Configure LDAP TLS

```shell
mkdir /etc/openldap/certs

cp /etc/ssl/private/server.key \
/etc/ssl/private/server.crt \
/etc/ssl/ca-bundle.pem \
/etc/openldap/certs/

chown ldap. /etc/openldap/certs/server.key \
/etc/openldap/certs/server.crt \
/etc/openldap/certs/ca-bundle.pem
```

### Creating SSL ldif file

```shell
vi mod_ssl.ldif

dn: cn=config
changetype: modify
add: olcTLSCACertificateFile
olcTLSCACertificateFile: /etc/openldap/certs/ca-bundle.pem
-
replace: olcTLSCertificateFile
olcTLSCertificateFile: /etc/openldap/certs/server.crt
-
replace: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/openldap/certs/server.key
```

### Adding SSL to LDAP DB

```shell
ldapmodify -Y EXTERNAL -H ldapi:/// -f mod_ssl.ldif
```

### Restarting the slapd service

```shell
systemctl restart slapd
```

## ADDING USERS TO LDAP SERVER

### Generating SSHA Password for user

```shell
slappasswd
New password:
Re-enter new password: {SSHA}xxxxxxxxx
```

### Creating ldif file for adding user entry

```shell
vi ldapuser.ldif

dn: uid=username,ou=People,dc=example,dc=com
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
cn: username
sn: user_surname
userPassword: {SSHA}xxxxxxxxxxxxxxx
loginShell: /bin/bash
uidNumber: user_id_number
gidNumber: group_id_number
homeDirectory: /home/username
```

### Adding the user to LDAP DB

```shell
ldapadd -x -D cn=Manager,dc=example,dc=com -W -f ldapuser.ldif
```

## CONFIGURING SYSTEM AS LDAP CLIENT

### Installing Packages

```shell
zypper -n install openldap2-client sssd pam_ldap nss_ldap
```

### Taking Backup of sssd.conf file

```shell
mv /etc/sssd/sssd.conf /etc/sssd/sssd.conf.org
```

### Creating New sssd.conf file

Note here that the suffix(dc=,dc=) matches as same as in the ldap server.

```shell
vi /etc/sssd/sssd.conf

[domain/default]
id_provider = ldap
auth_provider = ldap
ldap_uri = ldap://ldap.server.ip.or.dns
ldap_search_base = dc=example,dc=com
cache_credentials = True
ldap_tls_cacertdir = /etc/openldap/certs
ldap_tls_reqcert = allow

[sssd]
config_file_version = 2
services = nss, pam, sudo
domains = default

[nss]
filter_users = root
filter_groups = root
```

### Changing permission of sssd.conf file

```shell
chmod 600 /etc/sssd/sssd.conf
```

### Updating common-account file in pam.d folder

```shell
vi /etc/pam.d/common-account
# Do the changes as follows

account requisite       pam_unix.so     try_first_pass
account sufficient      pam_localuser.so
account required        pam_sss.so      use_first_pass
```

### Updating common-auth file in pam.d folder

```shell
vi /etc/pam.d/common-auth
# Do the changes as follows

auth    required        pam_env.so
auth    optional        pam_gnome_keyring.so
auth    sufficient      pam_unix.so     try_first_pass
auth    required        pam_sss.so      use_first_pass
```

### Updating common-password file in pam.d folder

```shell
vi /etc/pam.d/common-password
# Do the changes as follows

password        requisite       pam_cracklib.so
password        optional        pam_gnome_keyring.so    use_authtok
password        sufficient      pam_unix.so     use_authtok nullok shadow try_first_pass
password        required        pam_sss.so      use_authtok
```

### Updating common-session file in pam.d folder

```shell
vi /etc/pam.d/common-session
# Do the changes as follows

session required        pam_limits.so
session required        pam_unix.so     try_first_pass
session optional        pam_sss.so
session optional        pam_umask.so
session optional        pam_systemd.so
session optional        pam_gnome_keyring.so    auto_start only_if=gdm,gdm-password,lxdm,lightdm
session optional        pam_env.so
session optional        pam_mkhomedir.so skel=/etc/skel umask=0022
```

### Changing the parameter's value in nnsswitch.conf

```shell
passwd: compat sss
group: compat sss
```

### Restart the sssd & nscd service

```shell
systemctl enable sssd nscd
```
