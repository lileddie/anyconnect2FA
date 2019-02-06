# anyconnect2FA

Guide to implementing an Open Source two factor authentication solution to your existing Anyconnect VPN configuration.

## Assumptions

This guide assumes our primary source is MS Active Directory, although it could easily be modified to accommodate LDAP users.  We will need to have root access to a CentOS 7 node that will proxy our AD requests for initial user setup and perform secondary authentication.

## Set SELINUX to Permissive

For google-authenticator to work, SELINUX must be in "Permissive" mode.
Edit /etc/selinux/config and set SELINUX=permissive

Configure current session to permissive as well:

```
setenforce 0
```

## Install and Configure the required packages

```
yum -y install freeradius freeradius-utils epel-release google-authenticator sssd realmd adcli oddjob oddjob-mkhomedir samba-common-tools
ln -s /etc/raddb/mods-available/pam /etc/raddb/mods-enabled/pam
```
* edit /etc/raddb/radiusd.conf and set user and group to 'root'

* edit /etc/raddb/sites-enabled/default - under authenticate section uncomment 'pam'

* edit /etc/raddb/clients.conf and add config for your ASA node(s):
```
client my-asa {
ipaddr = 10.X.X.X
secret = mysecret12345
require_message_authenticator = no
nas_type = other
}
client my-backup-asa {
ipaddr = 10.X.X.X
secret = mysecret12345
require_message_authenticator = no
nas_type = other
}
```

* Update the default group used in in /etc/raddb/users:
```
DEFAULT Group == "disabled", Auth-Type := Reject
DEFAULT Reply-Message = "Bad password or disabled account."
DEFAULT Auth-Type := PAM
```

* Enable SSSD and join your domain:
```
systemctl enable sssd
systemctl start sssd
realm join --user=admin your.domain
```

* Modify the SSSD configuration file at /etc/sssd/sssd.conf:
```
[sssd]
domains = your.domain
default_domain_suffix = your
config_file_version = 2
services = nss, pam, pac

[nss]
filter_groups = root
filter_users = root
entry_cache_timeout = 300
entry_cache_nowait_percentage = 75

[pam]
offline_credentials_expiration = 2
offline_failed_login_attempts = 3
offline_failed_login_delay = 5

[domain/your.domain]
ad_domain = your.domain
krb5_realm = YOUR.DOMAIN
realmd_tags = manages-system joined-with-samba
cache_credentials = True
id_provider = ad
krb5_store_password_if_offline = True
default_shell = /bin/bash
ldap_id_mapping = True
use_fully_qualified_names = True
fallback_homedir = /home/%u@%d
access_provider = ad
auth_provider = ad
chpass_provider = ad
```

* Enable and start the realmd service
```
systemctl enable realmd
systemctl start realmd
```

* Allow radius auth to the node by updating firewalld:
```
firewall-cmd --permanent --zone=public --add-service=radius
firewall-cmd --reload
systemctl enable radiusd
systemctl start radiusd
systemctl restart sssd
```

* Update the sudo users to allow your support teams (as defined in Active Directory Groups) to manage the server.  This assumes the domain is your.domain and the 1st Level support team is AD group SUPPORT1ST and our 2nd Level is SUPPORT2ND.  This allows both teams to use all the sudo commands:
```
visudo
## update the following section
## Allow root to run any commands anywhere
root    ALL=(ALL)       ALL
%your\\SUPPORT1ST ALL=(ALL)      ALL
%your\\SUPPORT2ND ALL=(ALL)  ALL
```

* Setup your 2FA - end users will each have to perform this step.
```
ssh USERID@your@SERVER_IP_OR_HOSTNAME
google-authenticator
```
* I suggest using an app like Authy that doesn't require you regenerate the QR code if you switch devices, take a scan the code presented in the CLI and answer 'y' to each subsequent prompt.
