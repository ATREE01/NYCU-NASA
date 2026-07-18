# LDAP Server and Client Configuration Tutorial

This tutorial guides you through setting up an LDAP server, configuring various features like TLS, SUDOers, password policies, and setting up an LDAP client.

## Part 1: LDAP Server Setup

### 1.1. DNS/DHCP Prerequisite

Ensure your DNS is configured for the LDAP server.
LDAP Server IP: `192.168.12.1`
LDAP Workstation: `192.168.12.2`

Example DNS record:
```conf
ldap  IN  A 192.168.12.1
workstation IN A 192.168.12.2
```

### 1.2. Installing LDAP Packages
```bash
sudo apt install slapd ldap-utils libpam-ldapd
```

### 1.3. Basic LDAP Configuration
```bash
sudo dpkg-reconfigure slapd
```
Follow the prompts to set up your domain (e.g., `dc=12,dc=nasa`) and admin password.

### 1.4. Configuring TLS

#### 1.4.1. Prepare Certificates
Place your CA certificate, server certificate, and private key in `/etc/ldap/tls/`.
- CA Certificate: `ca-certificate.crt`
- Server Full Chain Certificate: `fullchain.pem`
- Server Private Key: `private.key`

Copy the CA certificate to the system's trusted store:
```bash
sudo cp /etc/ldap/tls/ca-certificate.crt /usr/local/share/ca-certificates/ldap-ca.crt
sudo update-ca-certificates
```

#### 1.4.2. Apply TLS Configuration to LDAP
Create `tls-config.ldif`:
```ldif
# tls-config.ldif
dn: cn=config
changetype: modify
add: olcTLSCACertificateFile
olcTLSCACertificateFile: /etc/ldap/tls/ca-certificate.crt
-
add: olcTLSCertificateFile
olcTLSCertificateFile: /etc/ldap/tls/fullchain.pem
-
add: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/ldap/tls/private.key
```
Apply the configuration:
```bash
sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f tls-config.ldif
```

#### 1.4.3. Disable Non-TLS Access (Enforce LDAPS)
Edit `/etc/default/slapd`:
```bash
sudo vim /etc/default/slapd
```
Modify `SLAPD_SERVICES` to:
```txt
SLAPD_SERVICES="ldaps:/// ldapi:///"
```
Restart LDAP server:
```bash
sudo systemctl restart slapd
```

#### 1.4.4. Test TLS Configuration
Verify TLS connection:
```bash
# Ensure ldap.12.nasa resolves correctly
openssl s_client -connect ldap.12.nasa:636 -CAfile /etc/ldap/tls/ca-certificate.crt
```
Check LDAP TLS settings:
```bash
sudo ldapsearch -Y EXTERNAL -H ldapi:/// -b cn=config | grep olcTLS
```
Test search over LDAPS:
```bash
ldapsearch -H ldaps://ldap.12.nasa -x -b dc=12,dc=nasa
```

### 1.5. Adding Organizational Units (OUs)
Create `ou-config.ldif`:
```ldif
#ou-config.ldif
dn: ou=People,dc=12,dc=nasa
objectClass: top
objectClass: organizationalUnit
ou: People

dn: ou=Group,dc=12,dc=nasa
objectClass: top
objectClass: organizationalUnit
ou: Group

dn: ou=Ppolicy,dc=12,dc=nasa
objectClass: top
objectClass: organizationalUnit
ou: Ppolicy

dn: ou=SUDOers,dc=12,dc=nasa
objectClass: top
objectClass: organizationalUnit
ou: SUDOers

dn: ou=Fortune,dc=12,dc=nasa
objectClass: top
objectClass: organizationalUnit
ou: Fortune
```
Add the OUs:
```bash
ldapadd -x -D "cn=admin,dc=12,dc=nasa" -W -f ou-config.ldif -H ldaps://ldap.12.nasa:636
```

### 1.6. Adding Groups
Create `group-config.ldif`:
```ldif
#group-config.ldif
dn: cn=ta,ou=Group,dc=12,dc=nasa
objectClass: top
objectClass: posixGroup
cn: ta
gidNumber: 10000

dn: cn=stu,ou=Group,dc=12,dc=nasa
objectClass: top
objectClass: posixGroup
cn: stu
gidNumber: 20000
```
Add the groups:
```bash
ldapadd -x -H ldaps://ldap.12.nasa:636 -D "cn=admin,dc=12,dc=nasa" -W -f group-config.ldif
```

### 1.7. Adding OpenSSH LPK Schema for SSH Public Keys
Create `openssh-lpk.ldif`:
```ldif
dn: cn=openssh-lpk,cn=schema,cn=config
objectClass: olcSchemaConfig
cn: openssh-lpk
olcAttributeTypes: ( 1.3.6.1.4.1.24552.500.1.1.1.13 NAME 'sshPublicKey' DESC 'MANDATORY: OpenSSH Public key' EQUALITY octetStringMatch SYNTAX 1.3.6.1.4.1.1466.115.121.1.40 )
olcObjectClasses: ( 1.3.6.1.4.1.24552.500.1.1.2.0 NAME 'ldapPublicKey' DESC 'MANDATORY: OpenSSH LPK objectclass' SUP top AUXILIARY MAY ( sshPublicKey $ uid ) )
```
Add the schema:
```bash
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f openssh-lpk.ldif
```

### 1.8. Creating Users
Create `user-config.ldif` (example users):
```ldif
dn: uid=generalta,ou=People,dc=12,dc=nasa
objectClass: top
objectClass: account
objectClass: posixAccount
objectClass: ldapPublickey
cn: generalta
uid: generalta
uidNumber: 10000
gidNumber: 10000
homeDirectory: /home/generalta
userPassword: {SSHA}uLdEVgymn5+LJOcudbOuSoWADs2wA82a
sshPublicKey: ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIFfg2DMY3DfBBvZCnqN8Az5tUnVQca+qXkJ9HceOcRAy 2025-na-hw4

dn: uid=stu12,ou=People,dc=12,dc=nasa
objectClass: top
objectClass: account
objectClass: posixAccount
objectClass: ldapPublickey
cn: stu12
uid: stu12
uidNumber: 20012
gidNumber: 20000
homeDirectory: /home/stu12
userPassword: {SSHA}uLdEVgymn5+LJOcudbOuSoWADs2wA82a
sshPublicKey: ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIFfg2DMY3DfBBvZCnqN8Az5tUnVQca+qXkJ9HceOcRAy 2025-na-hw4
```
Add the users:
```bash
ldapadd -x -H ldaps://ldap.12.nasa:636 -D "cn=admin,dc=12,dc=nasa" -W -f user-config.ldif
```

#### 1.8.1. Testing User Creation
Search for users:
```bash
ldapsearch -D "cn=admin,dc=12,dc=nasa" -W -H ldapi:/// -b "ou=People,dc=12,dc=nasa" "(objectClass=posixAccount)"
```
Test `ldapwhoami`:
```bash
ldapwhoami -x -H ldaps://ldap.12.nasa -D "uid=stu12,ou=People,dc=12,dc=nasa" -W
```

#### 1.8.2. Deleting a Specific User
```bash
ldapdelete -x -H ldaps://ldap.12.nasa:636 -D "cn=admin,dc=12,dc=nasa" -W "uid=<uid>,ou=People,dc=12,dc=nasa"
```

### 1.9. Configuring SUDOers in LDAP

#### 1.9.1. Install SUDO-LDAP and Convert Schema
```bash
sudo apt install schema2ldif sudo-ldap
sudo cp /usr/share/doc/sudo-ldap/schema.OpenLDAP /etc/ldap/schema/
sudo bash -c 'schema2ldif /etc/ldap/schema/schema.OpenLDAP > /etc/ldap/schema/OpenLDAP.ldif'
```
Add the SUDO schema to LDAP:
```bash
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/ldap/schema/OpenLDAP.ldif
```

#### 1.9.2. Define SUDOer Roles
Create `SUDOers-config.ldif`:
```ldif
dn: cn=ta,ou=SUDOers,dc=12,dc=nasa
objectClass: top
objectClass: sudoRole
cn: ta
sudoUser: %ta
sudoHost: ALL
sudoRunAsUser: ALL
sudoRunAsGroup: ALL
sudoCommand: ALL

dn: cn=stu,ou=SUDOers,dc=12,dc=nasa
objectClass: top
objectClass: sudoRole
cn: stu
sudoUser: %stu
sudoHost: ALL
sudoRunAsUser: ALL
sudoRunAsGroup: ALL
sudoCommand: /usr/bin/ls
```
Add SUDOer roles to LDAP:
```bash
sudo ldapadd -x -D "cn=admin,dc=12,dc=nasa" -W -H ldapi:/// -f SUDOers-config.ldif
```

#### 1.9.3. Configure `sudo-ldap.conf` (on Server and Clients)
Edit `/etc/sudo-ldap.conf`:
```bash
sudo vim /etc/sudo-ldap.conf
```
Add/modify the following:
```txt
sudoers_base ou=SUDOers,dc=12,dc=nasa
BASE    dc=12,dc=nasa
URI     ldaps://ldap.12.nasa
```

### 1.10. Configuring SSHD for LDAP Public Keys (Server & Client)

#### 1.10.1. Create SSH Key Lookup Script
This script is needed on both the server and clients that will use LDAP for SSH key authentication.
Create `/etc/ssh/script.sh`:
```bash
sudo vim /etc/ssh/script.sh
```
Add the following content:
```sh
#!/bin/sh
ldapsearch -x -H ldaps://ldap.12.nasa '(&(objectClass=posixAccount)(uid='"$1"'))' 'sshPublicKey' | sed -n '/^ /{H;d};/sshPublicKey:/x;$g;s/\n *//g;s/sshPublicKey: //gp'
```
Make it executable:
```bash
sudo chmod +x /etc/ssh/script.sh
```
Test the script:
```bash
sudo -u nobody /etc/ssh/script.sh stu12
```

#### 1.10.2. Configure SSHD
Edit `/etc/ssh/sshd_config`:
```bash
sudo vim /etc/ssh/sshd_config
```
Ensure these options are set (some might be defaults):
```
PubkeyAuthentication yes
AuthorizedKeysCommand /etc/ssh/script.sh # Add this for LDAP public keys
AuthorizedKeysCommandUser nobody         # Add this for LDAP public keys
KbdInteractiveAuthentication yes
PasswordAuthentication yes
```
For server-specific restrictions (example: disable password auth for `stu` group):
```
# Server才要match group
Match Group stu
    PasswordAuthentication no
    KbdInteractiveAuthentication no
    # PubkeyAuthentication no # This would disable public key auth for stu group, usually not desired.
                              # If you want to enforce only specific auth methods, adjust accordingly.
```
Restart SSH service:
```bash
sudo systemctl restart ssh
```

### 1.11. Access Control Lists (ACLs)
This example replaces default ACLs. Be cautious.
Create `acl-config.ldif`:
```ldif
# acl-config.ldif
# Default ACLs for reference (your current ones might differ):
# olcAccess: {0}to attrs=userPassword,shadowLastChange by self write by anonymous auth by dn="cn=admin,dc=example,dc=com" write by * none
# olcAccess: {1}to dn.base="" by * read
# olcAccess: {2}to * by self write by dn="cn=admin,dc=example,dc=com" write by * read

# Example replacement ACLs:
dn: olcDatabase={1}mdb,cn=config
changetype: modify
replace: olcAccess
# 0: Allow generalta manage access to the People subtree
olcAccess: {1}to dn.subtree="ou=People,dc=12,dc=nasa" by dn.exact="uid=generalta,ou=People,dc=12,dc=nasa" manage
  by self read
  by * read
# 1: Allow generalta manage access to the Group subtree
olcAccess: {2}to dn.subtree="ou=Group,dc=12,dc=nasa" by dn.exact="uid=generalta,ou=People,dc=12,dc=nasa" manage
  by self read
  by * read
# 2: Allow users to change their own password
olcAccess: {0}to attrs=userPassword by self write by anonymous auth by * none
# 3: Allow users to change their own shadowLastChange, and others can read shadowLastChange (often needed)
olcAccess: {3}to attrs=shadowLastChange by self write by * read
# 4: Allow general read access to all standard attributes for everyone (needed for getent, logins, etc.)
olcAccess: {4}to * by * read
# Implicit deny for anything not explicitly allowed by the above rules
```
Apply ACL changes:
```bash
sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f acl-config.ldif
```
Restart related services (if nslcd/nscd are used on the server for itself):
```bash
sudo systemctl restart nslcd
sudo systemctl restart nscd
```
Verify ACLs:
```bash
sudo ldapsearch -Y EXTERNAL -H ldapi:/// -b "olcDatabase={1}mdb,cn=config" -s base olcAccess olcDatabase olcSuffix
```
Test password change (as a user):
```bash
ldappasswd -x -H ldaps://ldap.12.nasa -D "uid=stu12,ou=People,dc=12,dc=nasa" -W -S
```

### 1.12. Password Policy (PPolicy)

The [ppolicy schema](https://www.zytrax.com/books/ldap/ape/ppolicy.html) can be found on this website. Load it using a method similar to the one described in section 1.9.1 for the SUDO schema.

> **Note:** Verify whether this schema has already been loaded.

#### 1.12.1. Load PPolicy Module
Create `ppolicy-module.ldif`:
```ldif
# ppolicy-module.ldif
dn: cn=module{0},cn=config
changetype: modify
add: olcModuleLoad
olcModuleLoad: ppolicy.la
```
Load the module:
```bash
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f ppolicy-module.ldif
```
Test module load:
```bash
sudo ldapsearch -Y EXTERNAL -H ldapi:/// -b "cn=module{0},cn=config" olcModuleLoad
```

#### 1.12.2. Define Default Password Policy
Create `ppolicy-default.ldif`:
```ldif
# ppolicy-default.ldif
dn: cn=default,ou=Ppolicy,dc=12,dc=nasa
objectClass: pwdPolicy
objectClass: pwdPolicyChecker
objectClass: device
objectClass: top
cn: default
pwdAttribute: userPassword
# Import pwdPolicyChecker for this one
pwdUseCheckModule: TRUE # Enable external check module
# I set the 1 so that if the password comes in pre-hashed form it will just let it pass.
pwdCheckQuality: 1      # 0=off, 1=check if possible, 2=check mandatory
pwdMinLength: 8
pwdInHistory: 2
pwdMustChange: FALSE
pwdAllowUserChange: TRUE
pwdSafeModify: TRUE
#pwdLockout: TRUE
#pwdMaxFailure: 5
#pwdLockoutDuration: 300
#pwdFailureCountInterval: 30
pwdGraceAuthNLimit: 0
pwdExpireWarning: 600
```
Add the default policy:
```bash
sudo ldapadd -D "cn=admin,dc=12,dc=nasa" -W -H ldapi:// -f ppolicy-default.ldif
```
Test policy creation:
```bash
ldapsearch -x -D "cn=admin,dc=12,dc=nasa" -W -b "cn=default,ou=Ppolicy,dc=12,dc=nasa"
```

#### 1.12.3. Custom Password Syntax Check
Create `checkclass.c`:
```c
// save as checkclass.c
#include <string.h>
#include <ctype.h>

// OpenLDAP expects this function name
int check_password(const char *pw) {
    int has_upper = 0, has_lower = 0, has_digit = 0, has_special = 0;
    int len = 0;

    for (const char *p = pw; *p; ++p, ++len) {
        if (isupper((unsigned char)*p)) has_upper = 1;
        else if (islower((unsigned char)*p)) has_lower = 1;
        else if (isdigit((unsigned char)*p)) has_digit = 1;
        else has_special = 1;
    }

    int classes = has_upper + has_lower + has_digit + has_special;

    // Require at least 3 of the 4 character classes
    return (classes >= 3) ? 0 : 1; // 0 for success, non-zero for failure
}
```
Compile the shared library:
```bash
gcc -fPIC -Wall -shared -o checkclass.so checkclass.c
sudo cp checkclass.so /usr/lib/ldap/
```

#### 1.12.4. Apply PPolicy Overlay
Create `ppolicy-overlay.ldif`. **Adjust `olcDatabase` if it's not `{1}mdb`.**
```ldif
# ppolicy-overlay.ldif
# Adjust olcDatabase={1}mdb if your main database has a different index
dn: olcOverlay=ppolicy,olcDatabase={1}mdb,cn=config
changetype: add
objectClass: olcOverlayConfig
objectClass: olcPPolicyConfig
olcOverlay: ppolicy
olcPPolicyDefault: cn=default,ou=Ppolicy,dc=12,dc=nasa # Adjusted DN
olcPPolicyHashCleartext: TRUE
olcPPolicyUseLockout: TRUE # Set to FALSE if not using lockout features from ppolicy-default.ldif
olcPPolicyCheckModule: /usr/lib/ldap/checkclass.so # Path to your custom check module
```
Add the overlay:
```bash
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f ppolicy-overlay.ldif
```
Test overlay application:
```bash
sudo ldapsearch -Y EXTERNAL -H ldapi:/// -b "cn=config" "(olcOverlay=ppolicy)"
```
> **Note on `pwdCheckQuality`**:
> *   `0`: No syntax checking.
> *   `1`: Server checks syntax; if unable (e.g., client-side hashed password), it's accepted.
> *   `2`: Server checks syntax; if unable, returns an error.
>
> References:
> 1.  [man slapo-ppolicy](https://man.archlinux.org/man/slapo-ppolicy.5.en)
> 2. https://github.com/ltb-project/slapd-cli/issues/46

### 1.13. Fortune Service (SSSVLV - Server Side Sort and Virtual List View)

#### 1.13.1. Load SSSVLV Module
Create `sssvlv-module.ldif`:
```ldif
# sssvlv-module.ldif
dn: cn=module{0},cn=config
changetype: modify
add: olcModuleLoad
olcModuleLoad: {2}sssvlv.la # Ensure index {2} is available or adjust
```
Load the module:
```bash
sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f sssvlv-module.ldif
```
Test module load:
```bash
sudo ldapsearch -Y EXTERNAL -H ldapi:/// -b cn=config '(objectClass=olcModuleList)' olcModuleLoad
```

#### 1.13.2. Apply SSSVLV Overlay
Create `sssvlv-overlay.ldif`. **Adjust `olcDatabase` if it's not `{1}mdb`.**
```ldif
# sssvlv-overlay.ldif
# Adjust olcDatabase={1}mdb if your main database has a different index
dn: olcOverlay=sssvlv,olcDatabase={1}mdb,cn=config
objectClass: olcOverlayConfig
objectClass: olcSssVlvConfig
olcOverlay: sssvlv
# olcSssVlvMax: <integer> # Optional: max entries for a VLV request
# olcSssVlvMaxKeys: <integer> # Optional: max keys for sort control
```
Add the overlay:
```bash
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f sssvlv-overlay.ldif
```
Test overlay application:
```bash
sudo ldapsearch -Y EXTERNAL -H ldapi:/// -b "olcOverlay=sssvlv,olcDatabase={1}mdb,cn=config"
```

#### 1.13.3. Define Fortune Schema
Create `fortune-schema.ldif`:
```ldif
# fortune-schema.ldif
dn: cn=fortune,cn=schema,cn=config
objectClass: olcSchemaConfig
cn: fortune
olcAttributeTypes: {0}( 1.3.6.1.4.1.99999.1.1 NAME 'author' DESC 'Author name' EQUALITY caseIgnoreMatch SUBSTR caseIgnoreSubstringsMatch ORDERING caseIgnoreOrderingMatch SYNTAX 1.3.6.1.4.1.1466.115.121.1.15 )
olcAttributeTypes: {1}( 1.3.6.1.4.1.99999.1.2 NAME 'id' DESC 'Fortune ID' EQUALITY integerMatch ORDERING integerOrderingMatch SYNTAX 1.3.6.1.4.1.1466.115.121.1.27 )
# olcAttributeTypes: {2}( 1.3.6.1.4.1.99999.1.3 NAME 'cn' DESC 'Common Name' EQUALITY caseIgnoreMatch SUBSTR caseIgnoreSubstringsMatch ORDERING caseIgnoreOrderingMatch SYNTAX 1.3.6.1.4.1.1466.115.121.1.15 ) # cn is usually already defined
olcObjectClasses: {0}( 2.25.84266982321428815378665320284111781862 NAME 'fortune' SUP top STRUCTURAL MUST ( id $ author $ description $ cn ) )
```
Add the schema:
```bash
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f fortune-schema.ldif
```
Test schema addition:
```bash
sudo ldapsearch -Y EXTERNAL -H ldapi:/// -b "cn=schema,cn=config" "(cn=fortune)"
```

#### 1.13.4. Add Fortunes
Create `fortunes.ldif` (example):
```ldif
# fortunes.ldif
dn: cn=fortune-7,ou=Fortune,dc=12,dc=nasa
objectClass: fortune
objectClass: top
id: 7
cn: fortune-7
author: Gandhi
description: Happiness is when what you think, what you say, and what you do are in harmony.
```
You can write a script to convert YAML or other formats to LDIF for bulk import.
Add the fortunes:
```bash
sudo ldapadd -x -D "cn=admin,dc=12,dc=nasa" -W -f fortunes.ldif -H ldaps://ldap.12.nasa # Use ldaps
```
Test fortune retrieval:
```bash
sudo ldapsearch -x -H ldaps://ldap.12.nasa -b "ou=Fortune,dc=12,dc=nasa" "(cn=fortune-7)"
```
Delete a fortune (example):
```bash
sudo ldapdelete -x -D "cn=admin,dc=12,dc=nasa" -W -H ldaps://ldap.12.nasa "cn=fortune-7,ou=Fortune,dc=12,dc=nasa"
```

## Part 2: LDAP Client Setup

### 2.1. Add Server's CA Certificate to Client
Copy the LDAP server's CA certificate (`ca-certificate.crt` from server's `/etc/ldap/tls/`) to the client machine.
Append it to the client's CA bundle (path might vary, common is `/usr/local/share/ca-certificates/` then create a `.crt` file, or directly to `/etc/ssl/certs/ca-certificates.crt` - be careful with direct modification).
A safer way:
```bash
# On client, assuming you copied ca-certificate.crt from server to current dir
sudo cp ca-certificate.crt /usr/local/share/ca-certificates/ldap-server-ca.crt
sudo update-ca-certificates
```

### 2.2. Install LDAP Client Packages
```bash
sudo apt install libnss-ldapd libpam-ldapd nslcd sudo-ldap
```
During installation, you'll be prompted for LDAP server URI (e.g., `ldaps://ldap.12.nasa`) and search base (e.g., `dc=12,dc=nasa`).

### 2.3. Configure PAM
Run `pam-auth-update` to enable LDAP authentication modules.
```bash
sudo pam-auth-update
```
Ensure "LDAP Authentication" and other relevant options are selected.

### 2.4. Configure `nsswitch.conf`
Edit `/etc/nsswitch.conf`:
```bash
sudo vim /etc/nsswitch.conf
```
Modify/ensure these lines to include `ldap`:
```txt
passwd:         files ldap systemd
group:          files ldap systemd
shadow:         files ldap
sudoers:        ldap files
```

### 2.5. Configure `sudo-ldap.conf` (Client-side)
This should be the same as on the server (Section 1.9.3).
Edit `/etc/sudo-ldap.conf`:
```bash
sudo vim /etc/sudo-ldap.conf
```
Ensure it contains:
```txt
sudoers_base ou=SUDOers,dc=12,dc=nasa
BASE    dc=12,dc=nasa
URI     ldaps://ldap.12.nasa
TLS_CACERT /etc/ssl/certs/ldap-server-ca.pem # Or the path to your CA cert on the client
                                            # This might be /etc/ssl/certs/ca-certificates.crt if globally trusted
```

### 2.6. Configure SSHD for LDAP Public Keys (Client-side)
This is the same script and `sshd_config` modification as on the server (Section 1.10), if the client also acts as an SSH server or needs to look up its own keys via LDAP for some reason. Typically, `AuthorizedKeysCommand` is for servers. If this client is purely a client, these SSHD changes might not be necessary unless it's also an SSH server.

If you need SSH key lookup for outgoing connections (e.g. `ssh user@host`), that's usually handled by the SSH client itself, not `sshd_config` on the client.

### 2.7. Restart Services
```bash
sudo systemctl restart nslcd
sudo systemctl restart nscd # nscd is a caching daemon, restart if used
sudo systemctl restart sshd # If sshd_config was changed
```

### 2.8. Test Client Configuration
- Try logging in as an LDAP user: `ssh ldapuser@localhost` (if on the client machine) or `ssh ldapuser@client-ip`.
- Check user information: `getent passwd ldapuser`
- Test sudo: `sudo -l -U ldapuser`

## Part 3: Bonus - Mail Server Integration

> The Postfix and Dovecot set up is very complicate. The part I record here is what I remembered. There are many problem you need to deal with yourself. But most of it can be solve through asking ChatGPT

> **Note**: The greylistening set up from HW3 will cause the just can't sent the mail to the server at first time. Need to by pass it or just turn it off.

### 3.1. Install Mail Server Packages

#### 3.1.1. Install Postfix LDAP Package
On the Mail Server:
```bash
sudo apt install postfix-ldap
```

#### 3.1.2. Install Dovecot LDAP Package
On the Mail Server:
```bash
sudo apt install dovecot-ldap
```

### 3.2. Configure Postfix

> **Security Note**: The following configurations include LDAP bind passwords (`bind_pw`). In a production environment, ensure these files have restricted permissions (e.g., readable only by root and the postfix user) or use more secure methods for password management if available.

#### 3.2.1. Create Postfix LDAP Lookup Files
Create `/etc/postfix/ldap-alias.cf`:
```cf
# /etc/postfix/ldap-alias.cf
server_host = ldaps://ldap.12.nasa
server_port = 636
search_base = ou=People,dc=12,dc=nasa

bind = yes
bind_dn = cn=admin,dc=12,dc=nasa
bind_pw = {your_password}
version = 3

query_filter = (&(objectClass=posixAccount)(uid=%u))
result_attribute = uid
result_format = %s@12.nasa

tls_ca_cert_file = /etc/ssl/certs/ca-certificates.crt
```

Create `/etc/postfix/ldap-mailbox.cf`:
```cf
# /etc/postfix/ldap-mailbox.cf
server_host = ldaps://ldap.12.nasa
server_port = 636
search_base = ou=People,dc=12,dc=nasa

bind = yes
bind_dn = cn=admin,dc=12,dc=nasa
bind_pw = {your_password}
version = 3

query_filter = (&(objectClass=posixAccount)(uid=%u))
result_attribute = uid
result_format = %s # Delivers to mailbox named after uid

tls_ca_cert_file = /etc/ssl/certs/ca-certificates.crt
```

Create `/etc/postfix/ldap-senders.cf`:
```cf
# /etc/postfix/ldap-senders.cf
server_host = ldaps://ldap.12.nasa
server_port = 636
search_base = ou=People,dc=12,dc=nasa

bind = yes
bind_dn = cn=admin,dc=12,dc=nasa
bind_pw = {your_password}
version = 3

query_filter = (&(objectClass=posixAccount)(uid=%u))
result_attribute = uid # Used for sender login maps

tls_ca_cert_file = /etc/ssl/certs/ca-certificates.crt
```

#### 3.2.2. Configure Postfix TLS for Chroot
The LDAP lookup files already specify `tls_ca_cert_file = /etc/ssl/certs/ca-certificates.crt`. This implies the CA certificate is trusted system-wide.
If Postfix services run in a chroot environment (e.g., `/var/spool/postfix`) and need access to this CA certificate for TLS verification during LDAP lookups, copy it into the chroot:
```bash
sudo cp /etc/ssl/certs/ca-certificates.crt /var/spool/postfix/etc/ssl/certs/
```

#### 3.2.3. Update Postfix `main.cf`
Edit `/etc/postfix/main.cf`:
```bash
sudo vim /etc/postfix/main.cf
```
Add/modify the following, ensuring `ldap:/etc/postfix/ldap-mailbox.cf` matches the created file:
```cf
# In /etc/postfix/main.cf

# For virtual alias expansion (e.g., user@domain -> anotheruser@anotherdomain or localuser)
# pcre:/etc/postfix/virtual would be for regex-based aliases.
virtual_alias_maps = pcre:/etc/postfix/virtual, ldap:/etc/postfix/ldap-alias.cf

# Settings for virtual mailbox domains handled by this server
virtual_mailbox_domains = 12.nasa

# ldap:/etc/postfix/ldap-mailbox.cf for LDAP users.
virtual_mailbox_maps = ldap:/etc/postfix/ldap-mailbox.cf

# Base directory for virtual mailboxes if Postfix itself is delivering (less relevant if using Dovecot LDA for final delivery based on its own lookup)
virtual_mailbox_base = /var/mail/12.nasa # Ensure this directory exists and has correct permissions for the delivery agent.

# Minimum UID for virtual mailbox users (security measure)
virtual_minimum_uid = 100

# System UID and GID that Postfix will use when delivering mail to virtual mailboxes.
# This should correspond to a dedicated mail user (e.g., vmail).
virtual_uid_maps = static:1000
virtual_gid_maps = static:1000

# For SASL authenticated relay control, mapping authenticated users to permitted sender addresses.
smtpd_sender_login_maps = hash:/etc/postfix/login_maps, ldap:/etc/postfix/ldap-senders.cf
```

#### 3.2.4. Update Postfix `master.cf` for Dovecot LDA
Edit `/etc/postfix/master.cf`:
```bash
sudo vim /etc/postfix/master.cf
```
Ensure the Dovecot Local Delivery Agent (LDA) is configured for delivering mail to Dovecot. This line tells Postfix to pipe mail for local delivery to the Dovecot `deliver` program:
```cf
# In /etc/postfix/master.cf - Example for Dovecot LDA
# This defines a transport named 'dovecot'
dovecot   unix  -       n       n       -       -       pipe
  flags=DRhu user=vmail:vmail argv=/usr/lib/dovecot/deliver -f ${sender} -d ${recipient}
# Ensure your virtual_transport in main.cf is set to 'dovecot' for 12.nasa domain if using this.
# e.g., virtual_transport = dovecot
```

#### 3.2.5. Reload Postfix Configuration
```bash
sudo systemctl reload postfix
```

### 3.3. Configure Dovecot

> **Security Note**: The Dovecot LDAP configuration (`dovecot-ldap.conf.ext`) includes the LDAP bind password (`dnpass`). Ensure this file has restricted permissions (e.g., readable only by root and the dovecot user).

#### 3.3.1. Configure Dovecot LDAP Settings
Edit `/etc/dovecot/dovecot-ldap.conf.ext`:
```bash
sudo vim /etc/dovecot/dovecot-ldap.conf.ext
```
Add/modify the following:
```ini
# /etc/dovecot/dovecot-ldap.conf.ext
uris = ldaps://ldap.12.nasa:636

# DN and password for Dovecot to bind to LDAP for lookups (e.g., iterating users, or if auth_bind=no)
dn = cn=admin,dc=12,dc=nasa
dnpass = {your_password}

# Path to the CA certificate for LDAPS
tls_ca_cert_file = /etc/ssl/certs/ca-certificates.crt

# Use LDAP for authentication (passdb) and user information (userdb)
# For user authentication, Dovecot will attempt to bind as the user themselves.
auth_bind = yes
auth_bind_userdn = uid=%u,ou=People,dc=12,dc=nasa # Template for user's DN

# Base DN for user searches
base = ou=People,dc=12,dc=nasa

# User attributes to fetch from LDAP.
# mail: Specifies mail location. Ensure /var/mail/vmail exists and is writable by the 'vmail' user/group (see 10-mail.conf).
# homeDirectory: User's home directory (optional for mail, but can be useful).
# uidNumber/gidNumber: Can be used by Dovecot for quota or other system integration if needed, maps to system UIDs/GIDs.
user_attrs = homeDirectory=home,=mail=maildir:/var/mail/vmail/%u,uidNumber=5000,gidNumber=5000

# LDAP filter to find users
user_filter = (&(objectClass=posixAccount)(uid=%n))

# If passwords are to be verified against an LDAP attribute (e.g. userPassword)
# This is used if auth_bind = no, or as a fallback/check.
# pass_attrs = uid=user,userPassword=password
# pass_filter = (&(objectClass=posixAccount)(uid=%n))
```

#### 3.3.2. Configure Dovecot Authentication (`10-auth.conf`)
Edit `/etc/dovecot/conf.d/10-auth.conf`:
```bash
sudo vim /etc/dovecot/conf.d/10-auth.conf
```
Ensure LDAP authentication is included and configured:
```ini
# /etc/dovecot/conf.d/10-auth.conf

auth_mechanisms = plain login

!include auth-system.conf.ext
!include auth-ldap.conf.ext

```

#### 3.3.3. Configure Dovecot Mail Location (`10-mail.conf`)
Edit `/etc/dovecot/conf.d/10-mail.conf` (original had typo `.colf`):
```bash
sudo vim /etc/dovecot/conf.d/10-mail.conf
```
Set mail location format and the system user/group that owns mail files. This should align with `user_attrs` in `dovecot-ldap.conf.ext` and Postfix's delivery settings.
```ini
# /etc/dovecot/conf.d/10-mail.conf

# Default mail location. %u is replaced with the username.
# This must match the 'mail' attribute format from user_attrs in dovecot-ldap.conf.ext.
mail_location = maildir:/var/mail/vmail/%u

# System user and group that owns the mailboxes.
# Create a 'vmail' user and group if they don't exist.
# The /var/mail/vmail directory should be owned by vmail:vmail.
mail_uid = vmail
mail_gid = vmail

```

#### 3.3.4. Configure Dovecot Master Services (`10-master.conf`)
Edit `/etc/dovecot/conf.d/10-master.conf`:
```bash
sudo vim /etc/dovecot/conf.d/10-master.conf
```
The original content shows adjustments to `auth-userdb` and `stats-writer` listeners. These are often for inter-process communication or specific integrations (e.g., Postfix SMTP AUTH).
```ini
# /etc/dovecot/conf.d/10-master.conf

service auth {
  # Original user content for auth-userdb:
  # This listener is for user database lookups.
  unix_listener auth-userdb {
    mode = 0777 # WARNING: 0777 is very permissive. Default is often 0600 or more restrictive (e.g., 0666 if group access needed).
    user = dovecot # Or specific user like 'vmail' if that's what needs access
    group = dovecot
  }
}

# Original user content for stats:
service stats {
  unix_listener stats-writer {
    mode = 0666 # WARNING: Permissive. Review if world-writable is necessary.
    user = dovecot
    group = dovecot
  }
}

```

#### 3.3.5. Reload/Restart Dovecot
After configuration changes, reload or restart Dovecot:
```bash
sudo systemctl restart dovecot
```

### 3.4. References
The original document included a reference:
> 1. https://www.mail-archive.com/postfix-users@postfix.org/msg66387.html


## Part 4: Debugging LDAP

### Common Commands

Check `nslcd` status (client-side):
```bash
sudo systemctl status nslcd
journalctl -u nslcd
```
Check `slapd` status (server-side):
```bash
sudo systemctl status slapd
journalctl -u slapd
```
Verbose LDAP searches:
```bash
ldapsearch -d1 -x -H ldaps://ldap.12.nasa -b "dc=12,dc=nasa" "(uid=testuser)"
```
Check logs on the server, typically `/var/log/syslog` or dedicated OpenLDAP logs if configured.
Use `tcpdump` or `wireshark` to inspect LDAP traffic (port 389 for LDAP, 636 for LDAPS).
```bash
sudo tcpdump -i any -n port 389 or port 636 -X
```
