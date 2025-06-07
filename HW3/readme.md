# Mail Server

## DNS

```shell
sudo vim /etc/bind/db.12.nasa
```

```conf
@       IN      MX      10      mail.12.nasa.
mail    IN      A       192.168.12.25
```

改完之後要resign

##  Postfix

```shell
sudo apt install postfix
sudo vim /etc/postfix/main.cf
```

基本Postfix + STARTTLS + TLS
```conf
mydomain = 12.nasa
myhostname = mail.12.nasa

# TLS parameters
smtpd_tls_cert_file=/etc/ssl/certs/mail.12.nasa.pem
smtpd_tls_key_file=/etc/ssl/private/mail.12.nasa.key
smtpd_use_tls=yes
smtpd_tls_security_level=may

smtp_tls_CApath=/etc/ssl/certs/nasa.crt
smtp_tls_security_level=may
smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache
```

Test command
```shell
openssl s_client -connect 192.168.12.25:25 -starttls smtp
openssl s_client -connect 192.168.12.25:143 -starttls imap

```

##  IMAP / SASL

```shell
sudo apt-get install dovecot-imapd
sudo vim /etc/dovecot/conf.d/10-master.conf
```
把原本給的設定解註解就可以了
```conf
service auth {
  unix_listener /var/spool/postfix/private/auth {
    mode = 0660
  }
}
```

```shell
sudo vim /etc/dovecot/conf.d/10-ssl.conf
```
```
# 那個'<'的符號很重要
ssl_cert = </etc/ssl/certs/mail.pem 
ssl_key = </etc/ssl/private/mail.key
```

設定Login要求
```shell
sudo vim /etc/dovecot/conf.d/10-auth.conf
```
```conf
auth_mechanisms = plain login
```

```conf
# dovecot
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes
smtpd_sender_login_maps = hash:/etc/postfix/login_maps

smtpd_recipient_restrictions =
                        reject_unknown_recipient_domain
                        # This configuration will be used for Greylist later. You can leave it blank for now when you reach this point.                 
                        check_policy_service inet:127.0.0.1:10023

smtpd_sender_restrictions =
        reject_unauthenticated_sender_login_mismatch
        reject_authenticated_sender_login_mismatch
        reject_unlisted_sender
        # sender access is used to reject the null sender
        check_sender_access hash:/etc/postfix/sender_access

```

Create the login map file this is used to prevent user from spoofing other.
```shell
sudo vim login_maps
```
```conf
atree atree
TA ta
cool-TA cool-ta
```

Create the sender access file

```shell
sudo vim sender_access
```
```conf
<> REJECT
```
```shell
sudo postmap /etc/postfix/sender_access
```

##  Securing Mail Service

On DNS server
```conf
# mail security
# SPF
@       IN      TXT     "v=spf1 ip4:192.168.12.25 -all"

# dkim
selector._domainkey     IN      TXT     ( "v=DKIM1; h=sha256; k=rsa; t=y; "
          "p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAybgSFU9/9zkw39VO2NodvfbwrLNsQUjceqRigypgTnj+3WyK5u/ugev7Hmh5ohLbhDhobj3HnaYaTlWpzLGnvkw9Yvje6BqB48pSR4BwLzCQxCPZP26HgE/rJEuZmDCN4quQS3aeo+N/yzKvaDQ/aZk8JMnTndBjpitij9Ropu9UH/o8j6AS7LMqr9XV/eXX7ZWzy4qZBVnNLp"
          "j+s7gvFm5vleUTYqh3lT+wowvwGzqbD2gQ5izPIVpb4td+rBGRgJnPbpn6lObJvHj0B25NEWu9wqAEHIA1wI2i2aiXfZ46t4fVhQKTt57Ree5Z56xTmFySLQi/PLdaCIGx/N0EcwIDAQAB" )  ; ----- DKIM key selector for 12.nasa

# DMARC

_dmarc  IN      TXT     "v=DMARC1;p=reject; rua=mailto:dmarc-report-rua@12.nasa; adkim=s; aspf=s;"
```

On Mail Server
## 設定dkim

```shell
sudo apt install opendkim opendkim-tools
```

```shell
sudo mkdir -p /etc/opendkim/keys/12.nasa
cd /etc/opendkim/keys/12.nasa

sudo opendkim-genkey -s selector -d 12.nasa
```

```shell
sudo vim /etc/opendkim.conf
```

```conf
Domain                  12.nasa
Selector                selector
KeyFile                 /etc/opendkim/keys/12.nasa/selector.private

Socket                  inet:8891@localhost
```

改`main.cf`檔案
```shell
sudo vim /etc/postfix/main.cf
```

```conf
# dkim
milter_default_action = accept
milter_protocol = 6
smtpd_milters = inet:localhost:8891
non_smtpd_milters = inet:localhost:8891
```

## PCRE
這個是用來解正規表達式的
```
sudo apt install postfix-pcre
```

## Greylisting

```shell
sudo apt install postgrey
sudo vim /etc/postgrey/whitelist_recipients.local
```
設定白名單
```conf
ta@ta.nasa
```
改`postgrey`設定
```shell
sudo /etc/default/postgrey
```
```conf
POSTGREY_OPTS="--inet=10023 --delay=30"
```

## Alias

```conf
#alias
alias_maps = hash:/etc/aliases
alias_database = hash:/etc/aliases
virtual_alias_maps = pcre:/etc/postfix/virtual
```
```shell
sudo vim /etc/aliases
```
```conf
NASATA: ta
```

```shell
sudo vim /etc/postfix/virtual
```
```conf
/^(\w+)\+(\w+)@(.*)$/ $1@$3
```

```shell
sudo vim /etc/postfix/main.cf
```

## rewrite
```conf
smtp_generic_maps = pcre:/etc/postfix/generic_maps
masquerade_domains = 12.nasa
```

```shell
sudo vim /etc/postfix/generic_maps
```
```conf
/^([\w-]*)@mail.12.nasa$/ $1@12.nasa
/^cool-TA@([\w\-.]*)$/ supercooool-TA@$1
```

## SPAM

```shell
sudo vim /etc/postfix/main.cf
```

```conf
# incomeing filter
content_filter = smtp-amavis:[127.0.0.1]:10024
# outgoing filter
header_checks = pcre:/etc/postfix/header_checks
```
### Outgoing
```shell
sudo vim /etc/postfix/header_checks
```

```conf
/^Subject:.*(Graduate School|博士班|=\?UTF-8\?B\?5Y2a5aOr54\+t\?=).*$/   REJECT
```
### Incomeing
```shell
sudo apt install spamassassin amavis clamav-daemon
sudo usermod amavis -aG clamav
```

```shell
sudo vim /etc/amavis/conf.d/05-node_id
```
```conf
$myhostname = "mail.12.nasa";
```

```shell
sudo vim /etc/amavis/conf.d/15-content_filter_mode
```

```conf
@bypass_virus_checks_maps = (
   \%bypass_virus_checks, \@bypass_virus_checks_acl, \$bypass_virus_checks_re);

@bypass_spam_checks_maps = (
   \%bypass_spam_checks, \@bypass_spam_checks_acl, \$bypass_spam_checks_re);
```

```shell
sudo vim /etc/amavis/conf.d/50-user
```

這兩個設定應該是有重疊的 應該是可以不用特別設定spamassassin的那個
```conf
$inet_socket_port = 10024;
$forward_method = 'smtp:127.0.0.1:10025';
$sa_spam_subject_tag = '**SPAM**';
$sa_kill_level_deflt = 1300.0;
$final_virus_destiny      = D_PASS;  # (data not lost, see virus quarantine)
$final_banned_destiny     = D_PASS;
$subject_tag_maps_by_ccat{+CC_VIRUS} = [ '**SPAM**' ];
```

```shell
sudo vim /etc/spamassassin/local.cf
```

```conf
rewrite_header Subject **SPAM**
```

權限設定

```shell
sudo adduser clamav amavis
```