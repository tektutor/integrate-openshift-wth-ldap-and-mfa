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

Open LDAP Port in firewall
```
sudo ufw allow 389/tcp
```

Let's see if we can search
```
ldapsearch -x -LLL -H ldap:/// -b dc=tektutor.org,dc=org
```
Check logs
```
sudo journalctl -u slapd
```

<img width="1920" height="1168" alt="image" src="https://github.com/user-attachments/assets/1478b637-ce11-45c3-a81f-af52d6db5c14" />


## Deploy keycloak in Openshift
```
oc new-project keycloak
oc apply -f https://raw.githubusercontent.com/keycloak/keycloak-quickstarts/latest/kubernetes/keycloak.yaml
```

In case Keycloak is running in development mode
```
oc set env deployment/keycloak \
  KC_PROXY=edge \
  KC_HOSTNAME=<your-route-hostname> \
  KC_HOSTNAME_STRICT=false \
  KC_HTTP_ENABLED=true \
  KC_HOSTNAME_STRICT_HTTPS=false \
  PROXY_ADDRESS_FORWARDING=true \
  -n keycloak

oc rollout restart deployment/keycloak -n keycloak
```

Edit the keycloak deployment and replace start-dev to start
```
command:
  - /opt/keycloak/bin/kc.sh
  - start
```

<img width="1920" height="1168" alt="image" src="https://github.com/user-attachments/assets/b2fca212-a3c2-4fd0-b9c6-b0d73a83bbff" />
<img width="1920" height="1168" alt="image" src="https://github.com/user-attachments/assets/4d47c4d5-b4a5-4ad0-b581-efd46a43e46c" />


#### Configure Keycloak for LDAP + OTP
Create a Realm for OpenShift:
Keycloak Admin Console → Add Realm → Name: openshift
<img width="1920" height="1168" alt="image" src="https://github.com/user-attachments/assets/6a46c0fa-d5b5-4598-8d90-59e93a521de4" />

Add LDAP User Federation
Go to User Federation → Add Provider → ldap
<img width="1920" height="1168" alt="image" src="https://github.com/user-attachments/assets/e4ca3bfc-bcbb-4fe1-bece-44d94f333e25" />
<img width="1920" height="1168" alt="image" src="https://github.com/user-attachments/assets/b6e4d52f-9dbd-4e44-99e6-7fd2044cd135" />

Configure:

Vendor: Active Directory / Other

Connection URL: ldap://192.168.0.112:389

Bind DN: LDAP bind account

Bind Credential: root@123

Users DN: ou=people,dc=tektutor,dc=org

Enable Sync Registrations = ON if you want to auto-create Keycloak users from LDAP.

Enable OTP in Realm:

Realm Settings → OTP Policy

Set:

Type: totp

Algorithm: HmacSHA1

Digits: 6

Period: 30

Set "OTP Required for Login" = ON

Force OTP Enrollment:

Authentication → Required Actions → Enable Configure OTP

Set as Default Action.

Add Keycloak as an Identity Provider in OpenShift
```
oc edit oauth cluster
```

<pre>
spec:
  identityProviders:
  - name: keycloak
    mappingMethod: claim
    type: OpenID
    openID:
      claims:
        preferredUsername:
        - LDAP-OTP-MFA
        name:
        - name
        email:
        - email
      clientID: openshift
      clientSecret:
        name: keycloak-client-secret
      issuer: https://192.168.1.80/realms/openshift  
</pre>

Register OpenShift in Keycloak
In Keycloak:

Go to Clients → Create:

Client ID: openshift

Access Type: confidential

Root URL: https://oauth-openshift.apps.<cluster-domain>/oauth2callback/keycloak

Valid Redirect URIs:
```
https://oauth-openshift.apps.<cluster-domain>/*
```

Save, then generate Client Secret.

Store it in OpenShift:
```
oc create secret generic keycloak-client-secret \
  --from-literal=clientSecret='<secret>' \
  -n openshift-config
```

Test the Flow
Go to OpenShift console → Click Log in with keycloak

Keycloak → Prompt LDAP credentials

If OTP not set up → Enroll OTP app (Google Authenticator, FreeOTP, Authy)

Enter OTP

Redirect back to OpenShift with authenticated session

<img width="1920" height="1168" alt="image" src="https://github.com/user-attachments/assets/97d2af5f-1db7-4b2f-9395-8e0fd1852c4e" />
