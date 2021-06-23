ADPassword
======================

A Zimbra server extension to change Active Directory passwords from the Zimbra web client.


The original project by Antonio Messina (antonio.messina@icar.cnr.it) https://github.com/xMAnton/ADPassword this version is tested on Zimbra 8.8.15 patch 5 and Windows 2016. Zimbra 9 patch 14 and Windows 2019.

I recommend the cli install from below.

To get the jar follow the instruction below:
https://github.com/Zimbra-Community/ADPassword#installation-via-the-cli

## How does it work?

1. The user clicks change password in the Zimbra UI.
2. The extension will find the user's DN in external AD/LDAP based on zimbraAuthLdapSearchFilter in zimbraAuthLdapSearchBase. 
3. The extension will not search in case you have set `External LDAP account for Authentication`. The DN from  zimbraAuthLdapExternalDn will then be used.
4. Finally the extension will change the password using bind credentials over a secure connection.

Please note: ADPassword does not honor password history (https://blogs.technet.microsoft.com/fieldcoding/2013/01/09/resetting-passwords-honoring-password-history-or-whats-happening-under-the-hood-when-changing-resetting-passwords/)

## Add the certificate from your Active Directory to the Zimbra server trust
If you use the same SSL certificate on your AD as on Zimbra there is a good change you can skip this step. If you already use your AD server for external auth, you can probably skip this as well. If you are not sure, configure your domain to auth against AD first before installing this extension. As of Zimbra 8.8.15 you MUST configure your AD server by it's DNS FQDN, you cannot use the change password extension by using the IP of your AD. This is because Java only proceeds if the SSL certificate matches the domain name in the configuration. (zimbraAuthLdapURL must be a domain, example of self signed windows cert: `zmprov md barrydegraaff.tk zimbraAuthLdapURL "ldaps://WIN-M7ME1BSBTRY.barrydegraaff.tk:636"`)

* /opt/zimbra/common/bin/keytool -import -alias win2012 -keystore /opt/zimbra/common/etc/java/cacerts -trustcacerts -file your-exported-cert.cer
* default password: changeit

To extract the SSL cert of the AD use:

```
echo -n | openssl s_client -connect DC_HOSTNAME:3269 -servername DC_HOSTNAME | openssl x509 > your-exported-cert.cer
```

Please note that on recent Zimbra versions and especially with AD/Samba4, you must use a DNS domain name to connect to the AD server, using IP addresses no longer works and is not secure SSL/TLS.

## Installation via the cli

Review your LDAP configuration in the commands below and then copy-paste them:

      mkdir -p /opt/zimbra/lib/ext/adpassword
      wget https://github.com/Zimbra-Community/ADPassword/releases/download/0.0.7/ADPassword.jar -O /opt/zimbra/lib/ext/adpassword/adPassword.jar       
      su zimbra
      zmprov md domain.ext zimbraAuthLdapBindDn "%u@domain.ext"
      zmprov md domain.ext zimbraAuthLdapSearchBase "CN=Users,DC=DOMAIN,DC=EXT"
      zmprov md domain.ext zimbraAuthLdapSearchBindDn "CN=serviceAccount,CN=Users,DC=DOMAIN,DC=EXT"
      zmprov md domain.ext zimbraAuthLdapSearchBindPassword "your-password-here"
      zmprov md domain.ext zimbraAuthLdapSearchFilter "(samaccountname=%u)"
      zmprov md domain.ext zimbraAuthLdapURL "ldaps://ad-server-dns-name:636"
      zmprov md domain.ext zimbraExternalGroupLdapSearchBase "CN=Users,DC=DOMAIN,DC=EXT"
      zmprov md domain.ext zimbraExternalGroupLdapSearchFilter "(samaccountname=%u)"
      zmprov md domain.ext zimbraAuthMech "ad"
      zmprov md domain.ext zimbraAuthMechAdmin "ad"
      zmprov md domain.ext zimbraPasswordChangeListener ADPassword
      zmprov gd domain.ext | grep -i ldap | grep -v Gal
      zmprov gd domain.ext | grep -i zimbraPasswordChangeListener
      zmprov md domain.ext zimbraAuthFallbackToLocal FALSE
      zmcontrol restart

* If you want a custom password complexity rules, see: https://github.com/Zimbra-Community/ADPassword/wiki/Adding-a-password-policy-check
* Sometimes when the user clicks the change password option, Zimbra goes to a URL on port 8443. To fix: `zmprov mcf zimbraChangePasswordURL https://your-zimbra-server.com/h/changepass?skin=harmony`

## Zimbra only accounts with local password

By setting `zimbraAuthFallbackToLocal` to *TRUE* you can skip AD password update, which allows creating a mailbox 
without a corresponding user using Zimbra password backend. If you require this, it's **recommended** to enable it 
only after successfully testing a password update against AD. 

## Support for Zentyal

ADPassword also supports Zentyal as directory server, please check [the wiki](https://github.com/Zimbra-Community/ADPassword/wiki/Support-for-Zentyal) for 
configuration details.

## Debugging
Do a password change while you run the following command:

     tail -f /opt/zimbra/log/zmmailboxd.out

You should find *ADPassword* messages passing by explaining what's going on.

Verify your configuration:

     zmprov gd domain.ext | grep -i ldap | grep -v Gal

Example issues:

     Wrong bind DN:
     LDAP: error code 34 - 0000208F: NameErr: DSID-03100225, problem 2006 (BAD_NAME)
     
     Forgot to set zimbraAuthLdapSearchFilter or other required attribute:
     A network service error has occurred
     system failure: java.lang.NullPointerException

## License* 
* Copyright (C) 2016-2021  Barry de Graaff [Zeta Alliance](https://zetalliance.org/)
* packaging, fixes and adjustments for ZCS 8.5/8.6 Copyright 2016 VNC AG
* originally Copyright 2012 Antonio Messina (a.messina@iknowconsulting.it)

Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with the License. You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0
Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
