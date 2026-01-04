# Demo
## Checks
```bash
ip -4 addr
```
Vikunja anmelden
```bash
sudo systemctl restart nginx
sudo systemctl enable nginx
```
```bash
cd; cd vikunja
sudo docker-compose up -d
```

## OpenLDAP
### starten
```bash
sudo /usr/local/libexec/slapd \
  -f /usr/local/etc/openldap/slapd.conf \
  -h "ldap://0.0.0.0:389"
```
### LDAP search
#### Benutzer überprüfen
```bash
ldapsearch -x -H ldap://127.0.0.1:389 \
  -D "cn=Manager,dc=fets,dc=local" -w secret \
  -b "ou=People,dc=fets,dc=local"
```

auf den Provider
```bash
sudo tee /usr/local/etc/openldap/demo-users-2.ldif > /dev/null << 'EOF'
dn: uid=pschmidt,ou=People,dc=fets,dc=local
objectClass: inetOrgPerson
uid: pschmidt
cn: Peter Schmidt
sn: Schmidt
givenName: Peter
mail: peter.schmidt@fets.local
userPassword: geheim456
EOF
```

#### Benutzer hinzufügen (auf Provider):
```bash
ldapadd -x -H ldap://127.0.0.1:389 \
  -D "cn=Manager,dc=fets,dc=local" -w secret \
  -f /usr/local/etc/openldap/demo-users-2.ldif
```

#### Benutzer überprüfen
```bash
ldapsearch -x -H ldap://127.0.0.1:389 \
  -D "cn=Manager,dc=fets,dc=local" -w secret \
  -b "ou=People,dc=fets,dc=local"
```

Vikunja anmelden
User: `pschmidt`
PW: `geheim456`

### Provider VM aussschalten
und dann wider bei Vikunja anmelden

