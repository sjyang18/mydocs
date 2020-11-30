# Adding Organization Unit (OU) and User& Service Principals with OpenLdap utilities to Azure AD DS

In the previous article ([a relative link](../wo_hdinsight/accumulo_aad_wo_hdinsight.md)), we show how to use a domain-joined Window Server to add OU, users, service principals, and generate keytab file. In this article, we will capture the similar steps with OpenLdap utilties and give a guidance to potentially come up with an automation solution in the future. 

## Setup 

Install OpenLDAP tools first. If your linux distribution is Ubuntu, 
```
sudo apt-get update
sudo apt-get install ldap-utils
```

Then, add your LDAP connection info to ~/.ldaprc file. For example, in my case,

```
BASE    DC=agceci,DC=onmicrosoft,DC=com
URI     ldaps://ldaps.agceci.onmicrosoft.com
BINDDN  CN=Seokwon Yang,OU=AADDC Users,DC=agceci,DC=onmicrosoft,DC=com
```
Note that I set my BASE and BINDDN as well in the example. Change according to your setting.

Install your self-signed ldaps.crt certificates if your domain is using a self-signed certificate. Download the ldaps.crt and install with following commands. You will have ldap connection error if you do not install your self-signed CA certificate.
```
sudo cp ldaps.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates
```

Test ldap connectivity with the following `ldapsearch` command. You will be prompt for the password for BINDDN and should see directory object info. 

```
ldapsearch -x -W
```

## Add your Organization Unit (OU)
To add your OU and maintain the new list of users and computers under the OU, you create a ldif file. The example below is my experiement (i.e. OU Name: MyCustomOU2) and you will update according to your domain and OU name.

```
dn: OU=MyCustomOU2,DC=agceci,DC=onmicrosoft,DC=com
objectClass: organizationalUnit
ou: MyCustomOU2
```

Save it to newOU.ldif and run `ldapadd` command.
```
ldapadd -x -W -f ./newOU.ldif
```

## Add users and mapped service principal to your OU
In this example, I am creating a azureuser in testvm2 (which is domain joined Azure VM in my domain) and set the serviePrincipalName. You will modify the values according to your domain, OU you created in the setup, your domain-joined vm name, user name, and its password. Note the password is base-encoded with `UTF-16LE` output. Update the unicodePwd with your base-encoding one. 

```
dn: CN=azureuser/testvm2.agceci.onmicrosoft.com,OU=MyCustomOU2,DC=agceci,DC=onmicrosoft,DC=com
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: user
cn: azureuser/testvm2.agceci.onmicrosoft.com
sAMAccountName: azureuser_testvm2
userPrincipalName: azureuser/testvm2.agceci.onmicrosoft.com@AGCECI.ONMICROSOFT.COM
servicePrincipalName: azureuser/testvm2.agceci.onmicrosoft.com
accountExpires: 0
userAccountControl: 66048
unicodePwd::IgBoAGEAZABvAG8AcABSAG8AYwBrAHMAMQAyADMAIQAiAA==
```

Save your change to azureuser.testvm2.ldif and run `ladpadd` command.
```
ldapadd -x -W -f ./azureuser.testvm2.ldif
```

## Verfication with ldapsearch
You may run `ldapsearch` command to verify the creation of OU and users.

```
ldapsearch -x -W -b "OU=MyCustomOU2,DC=agceci,DC=onmicrosoft,DC=com"
```

## Keytab file for service principal
ktutil will be used to generate a keytab file for the service principal you created. We may adapt & script this step with one of methods from (https://stackoverflow.com/questions/37454308/script-kerberos-ktutil-to-make-keytabs).

```
azureuser@testvm1:~/ldap$ ktutil
ktutil:  addent -password -p azureuser/testvm2.agceci.onmicrosoft.com@AGCECI.ONMICROSOFT.COM -k 1 -e RC4-HMAC
Password for azureuser/testvm2.agceci.onmicrosoft.com@AGCECI.ONMICROSOFT.COM:
ktutil:  wkt azureuser.keytab
ktutil:  q

azureuser@testvm1:~/ldap$ klist -kt ./azureuser.keytab
Keytab name: FILE:./azureuser.keytab
KVNO Timestamp         Principal
---- ----------------- --------------------------------------------------------
   1 11/23/20 23:52:01 azureuser/testvm2.agceci.onmicrosoft.com@AGCECI.ONMICROSOFT.COM
azureuser@testvm1:~/ldap$ kinit -kt ./azureuser.keytab azureuser/testvm2.agceci.onmicrosoft.com@AGCECI.ONMICROSOFT.COM
azureuser@testvm1:~/ldap$ klist
Ticket cache: FILE:/tmp/krb5cc_1000
Default principal: azureuser/testvm2.agceci.onmicrosoft.com@AGCECI.ONMICROSOFT.COM

Valid starting     Expires            Service principal
11/23/20 23:56:54  11/24/20 00:26:54  krbtgt/AGCECI.ONMICROSOFT.COM@AGCECI.ONMICROSOFT.COM
        renew until 11/30/20 23:56:54

```



