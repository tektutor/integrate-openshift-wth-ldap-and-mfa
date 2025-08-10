# Integrate Openshift With LDAP and MFA

## Install install openldap in Ubuntu
```
sudo apt update
sudo apt install slapd ldap-utils -y
```
<img width="1920" height="1168" alt="image" src="https://github.com/user-attachments/assets/2a214c7d-3e1d-40ed-9b70-a21ca80fd7fc" />
<img width="1920" height="1168" alt="image" src="https://github.com/user-attachments/assets/c609ae2f-5ac3-45a1-a351-519950c11145" />
<img width="1920" height="1168" alt="image" src="https://github.com/user-attachments/assets/e0cde99f-4043-434e-a6fb-8b3a8b9362d2" />
<img width="1920" height="1168" alt="image" src="https://github.com/user-attachments/assets/214f5a4c-f4c1-4bf1-8bef-fe5e4aec308a" />
<img width="1920" height="1168" alt="image" src="https://github.com/user-attachments/assets/f9ee4146-5911-4eab-b799-b06bd05e523d" />
<img width="1920" height="1168" alt="image" src="https://github.com/user-attachments/assets/d6ac1bbc-df36-4954-a7c4-678acb0ebb26" />
<img width="1920" height="1168" alt="image" src="https://github.com/user-attachments/assets/9ff9e11d-0538-45ee-8e5b-65ff9d9cf1f5" />
<img width="1920" height="1168" alt="image" src="https://github.com/user-attachments/assets/79477edd-f1bf-4173-95b7-77b566b397a5" />

Verify openldap is installed properly
```
sudo systemctl status slapd
ldapsearch -x -LLL -H ldap:/// -b dc=tektutor,dc=org
```
<img width="1920" height="1168" alt="image" src="https://github.com/user-attachments/assets/1b04d147-9a9f-42c8-a696-98376793a876" />

Create a Basic LDAP structure ( base.ldif )
```
dn: ou=people,dc=tektutor,dc=org
objectClass: organizationalUnit
ou: people

dn: ou=groups,dc=tektutor,dc=org
objectClass: organizationalUnit
ou: groups
```

Let's add the basic LDAP structure
```
ldapadd -x -D cn=admin,dc=tektutor,dc=org -W -f base.ldif
```

Generate hashed password
```
slappasswd -h {SSHA} -s root@123
```

Add an user, user.ldif
```
dn: uid=john,ou=people,dc=tektutor,dc=org
objectClass: inetOrgPerson
sn: Swaminathan
givenName: Jeganathan
cn: Jeganathan Swaminathan
displayName: Jeganathan Swaminathan
uid: jegan
mail: jegan@tektutor.org
userPassword: {SSHA}hashedpassword
```

Let's see if we can search
```
ldapsearch -x -LLL -H ldap:/// -b dc=tektutor.org,dc=org
```
Check logs
```
sudo journalctl -u slapd
```
