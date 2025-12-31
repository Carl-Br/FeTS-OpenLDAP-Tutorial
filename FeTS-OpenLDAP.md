# OpenLDAP Replikation

**Setup-Anleitung für Provider-Consumer-Replikation**

---
Es wird `ITS_Client_Debian12_AMD64_V02.ova`verwendet

## 1. Vorbereitung - IPv4-Adressen konfigurieren

Sicherstellen, dass beide VMs unterschiedliche IPv4-Adressen haben.
### Aktuelle IP-Adresse prüfen
```bash
ip -4 addr
```
### IPv4-Adresse ändern
Konfigurationsdatei bearbeiten:
```bash
vim /etc/network/interfaces
```
### Netzwerk neu starten
```bash
sudo systemctl restart networking
ip -4 addr
```
---
## 2. OpenLDAP Installation (auf beiden VMs)
### Arbeitsverzeichnis erstellen
```bash
cd; mkdir openldap; cd openldap
```
### OpenLDAP herunterladen
```bash
wget --no-check-certificate \
https://mirror.eu.oneandone.net/software/openldap/openldap-release/openldap-2.6.10.tgz
```
### Archiv entpacken
```bash
gunzip -c openldap-2.6.10.tgz | tar xvfB -
cd openldap-2.6.10
```
### Kompilierung
```bash
./configure
make depend
make
sudo make install
```
### Datenverzeichnis erstellen
```bash
sudo mkdir -p /usr/local/var/openldap-data
sudo chmod 700 /usr/local/var/openldap-data
```
---
## 3. Basiskonfiguration
### Konfigurationsdatei bearbeiten
Datei `/usr/local/etc/openldap/slapd.conf` anpassen:
```bash
sudo vim /usr/local/etc/openldap/slapd.conf
```
```
suffix "dc=fets,dc=local"
rootdn "cn=Manager,dc=fets,dc=local"
```
### Konfiguration testen
```bash
sudo /usr/local/sbin/slaptest -u -f /usr/local/etc/openldap/slapd.conf
```
### Base LDIF erstellen
Datei erstellen:
```bash
sudo vim /usr/local/etc/openldap/base.ldif
```
```ldif
dn: dc=fets,dc=local
objectClass: dcObject
objectClass: organization
o: fets company
dc: fets

dn: cn=Manager,dc=fets,dc=local
objectClass: organizationalRole
cn: Manager
```
---
## 4. Provider-Konfiguration (VM1)
### 4.1 SLAPD starten
```bash
sudo /usr/local/libexec/slapd \
  -f /usr/local/etc/openldap/slapd.conf \
  -h "ldap://0.0.0.0:389"
```
#### SLAPD-Start überprüfen
```bash
ldapsearch -x -b '' -s base '(objectclass=*)' namingContexts
```
### 4.2 Base-Daten einfügen

```bash
ldapadd -x -H ldap://127.0.0.1:389 \
  -D "cn=Manager,dc=fets,dc=local" -w secret \
  -f /usr/local/etc/openldap/base.ldif
```
#### LDAP-Funktionalität bestätigen
```bash
ldapsearch -x -H ldap://127.0.0.1:389 \
  -D "cn=Manager,dc=fets,dc=local" -w secret \
  -b "dc=fets,dc=local"
```
*Erwartetes Ergebnis:* Die Ausgabe sollte die konfigurierte Domain enthalten.
### 4.3 Replikations-Konfiguration
In `/usr/local/etc/openldap/slapd.conf` im MDB-Datenbankbereich folgende Änderungen vornehmen:
```bash
sudo vim /usr/local/etc/openldap/slapd.conf
```
#### Am Anfang des Datenbankbereichs einfügen
```
serverID 1
```
#### Am Ende des Datenbankbereichs hinzufügen
```
index entryUUID eq
index entryCSN eq

access to *
  by dn.exact="cn=replicator,dc=fets,dc=local" read
  by  * read

# Pflicht Overlay für Replikation
overlay syncprov
syncprov-checkpoint 100 10
syncprov-sessionlog 100
```
#### Komplette MDB-Datenbank-Konfiguration
```
#######################################################################
# MDB database definitions
#######################################################################
serverID 1

database        mdb
maxsize         1073741824
suffix          "dc=fets,dc=local"
rootdn          "cn=Manager,dc=fets,dc=local"
rootpw          secret
directory       /usr/local/var/openldap-data
index   objectClass     eq
index entryUUID eq
index entryCSN eq

access to *
  by dn.exact="cn=replicator,dc=fets,dc=local" read
  by  * read

overlay syncprov
syncprov-checkpoint 100 10
syncprov-sessionlog 100

#######################################################################
```
### 4.4 Replicator-Benutzer erstellen
SLAPD beenden:
```bash
sudo pkill slapd
```
Datei erstellen:
```bash
sudo vim /usr/local/etc/openldap/replicator.ldif
```
```ldif
dn: cn=replicator,dc=fets,dc=local
objectClass: simpleSecurityObject
objectClass: organizationalRole
cn: replicator
userPassword: secret
description: replication user
```

SLAPD neu starten:
```bash
sudo /usr/local/libexec/slapd \
  -f /usr/local/etc/openldap/slapd.conf \
  -h "ldap://0.0.0.0:389"
```

Replicator-Benutzer hinzufügen:
```bash
ldapadd -x -H ldap://127.0.0.1:389 \
  -D "cn=Manager,dc=fets,dc=local" -w secret \
  -f /usr/local/etc/openldap/replicator.ldif
```

#### Provider bestätigen
```bash
ldapwhoami -x -H ldap://127.0.0.1:389 \
  -D "cn=replicator,dc=fets,dc=local" -w secret
```

*Erwartete Ausgabe:* `dn: cn=replicator,dc=fets,dc=local`

---
## 5. Consumer-Konfiguration (VM2)
### 5.1 SLAPD-Konfiguration anpassen

In `/usr/local/etc/openldap/slapd.conf` folgende Konfiguration verwenden:
```bash
sudo vim /usr/local/etc/openldap/slapd.conf
```

> [!Important]
> `IP_Provider` muss 2 mal durch die tatsächliche IP-Adresse des Providers ersetzt werden!

```
#######################################################################
# MDB database definitions
#######################################################################
serverID 2

database        mdb
maxsize         1073741824
suffix          "dc=fets,dc=local"
rootdn          "cn=Manager,dc=fets,dc=local"
rootpw          secret
directory       /usr/local/var/openldap-data
index   objectClass     eq
index entryUUID eq
index entryCSN eq

syncrepl rid=001
  provider=ldap://IP_Provider:389
  type=refreshAndPersist
  searchbase="dc=fets,dc=local"
  bindmethod=simple
  binddn="cn=replicator,dc=fets,dc=local"
  credentials="secret"
  retry="5 5 300 5"
  timeout=1
 
updateref ldap://IP_Provider:389

#######################################################################
```

### 5.2 SLAPD neu starten

```bash
sudo pkill -9 slapd

sudo /usr/local/libexec/slapd \
  -f /usr/local/etc/openldap/slapd.conf \
  -h "ldap://0.0.0.0:389"
```

#### SLAPD-Start überprüfen
```bash
ldapsearch -x -b '' -s base '(objectclass=*)' namingContexts
```

---

## 6. Replikation validieren
Folgenden Befehl auf **beiden VMs** ausführen:
```bash
ldapsearch -x -H ldap://127.0.0.1:389 \
  -D "cn=Manager,dc=fets,dc=local" \
  -w secret \
  -b "dc=fets,dc=local"
```
**Erwartetes Ergebnis:** Die Ausgabe sollte auf beiden VMs identisch sein. Dies bestätigt, dass die Replikation erfolgreich funktioniert.

---

## 7. Demo - Benutzer und Organisationsstruktur anlegen

### 7.0 Schemas für Benutzer-Objekte prüfen und hinzufügen
> [!Important]
> Dieser Schritt muss auf beiden VMS gemacht werden.

Bevor wir Benutzer anlegen können, müssen die erforderlichen Schemas geladen sein.

**1. SLAPD stoppen:**
```bash
sudo pkill slapd
```

**2. slapd.conf bearbeiten:**
```bash
sudo vim /usr/local/etc/openldap/slapd.conf
```

**3. Am Anfang der Datei (nach den Kommentaren) die Schemas hinzufügen:**
> [!Important]
> Die Reihenfolge muss beibehalten werden
```
include         /usr/local/etc/openldap/schema/core.schema
include         /usr/local/etc/openldap/schema/cosine.schema
include         /usr/local/etc/openldap/schema/inetorgperson.schema
```

**4. Konfiguration testen:**
```bash
sudo /usr/local/sbin/slaptest -u -f /usr/local/etc/openldap/slapd.conf
```

*Erwartete Ausgabe:* `config file testing succeeded`

**5. SLAPD neu starten:**
```bash
sudo /usr/local/libexec/slapd \
  -f /usr/local/etc/openldap/slapd.conf \
  -h "ldap://0.0.0.0:389"
```


### 7.1 Organisationsstruktur aufbauen
> [!Important]
> Jetzt wieder auf den Provider wechseln
#### Organizational Units (OUs) erstellen
Datei `/usr/local/etc/openldap/demo-structure.ldif` erstellen:
```bash
sudo vim /usr/local/etc/openldap/demo-structure.ldif
```

```ldif
dn: ou=People,dc=fets,dc=local
objectClass: organizationalUnit
ou: People
description: Alle Benutzer

dn: ou=Groups,dc=fets,dc=local
objectClass: organizationalUnit
ou: Groups
description: Alle Gruppen

dn: ou=Departments,dc=fets,dc=local
objectClass: organizationalUnit
ou: Departments
description: Abteilungen
```

Struktur einfügen (auf Provider/VM1):
```bash
ldapadd -x -H ldap://127.0.0.1:389 \
  -D "cn=Manager,dc=fets,dc=local" -w secret \
  -f /usr/local/etc/openldap/demo-structure.ldif
```

### 7.2 Benutzer anlegen

Datei `/usr/local/etc/openldap/demo-users.ldif` erstellen:
```bash
sudo vim /usr/local/etc/openldap/demo-users.ldif
```

```ldif
dn: uid=mmueller,ou=People,dc=fets,dc=local
objectClass: inetOrgPerson
uid: mmueller
cn: Max Mueller
sn: Mueller
givenName: Max
mail: max.mueller@fets.local
userPassword: geheim123

dn: uid=aschmidt,ou=People,dc=fets,dc=local
objectClass: inetOrgPerson
uid: aschmidt
cn: Anna Schmidt
sn: Schmidt
givenName: Anna
mail: anna.schmidt@fets.local
userPassword: geheim456
```

Benutzer hinzufügen (auf Provider/VM1):
```bash
ldapadd -x -H ldap://127.0.0.1:389 \
  -D "cn=Manager,dc=fets,dc=local" -w secret \
  -f /usr/local/etc/openldap/demo-users.ldif
```

#### Benutzer überprüfen
```bash
ldapsearch -x -H ldap://127.0.0.1:389 \
  -D "cn=Manager,dc=fets,dc=local" -w secret \
  -b "ou=People,dc=fets,dc=local"
```

#### Login-Test
```bash
ldapwhoami -x -H ldap://127.0.0.1:389 \
  -D "uid=mmueller,ou=People,dc=fets,dc=local" \
  -w geheim123
```

*Erwartete Ausgabe:* `dn:uid=mmueller,ou=People,dc=fets,dc=local`

### 7.3 Gruppen erstellen

Datei `/usr/local/etc/openldap/demo-groups.ldif` erstellen:
```bash
sudo vim /usr/local/etc/openldap/demo-groups.ldif
```

```ldif
dn: cn=developers,ou=Groups,dc=fets,dc=local
objectClass: groupOfNames
cn: developers
description: Entwickler-Team
member: uid=mmueller,ou=People,dc=fets,dc=local

dn: cn=admins,ou=Groups,dc=fets,dc=local
objectClass: groupOfNames
cn: admins
description: Administratoren
member: uid=aschmidt,ou=People,dc=fets,dc=local
```

Gruppen hinzufügen:
```bash
ldapadd -x -H ldap://127.0.0.1:389 \
  -D "cn=Manager,dc=fets,dc=local" -w secret \
  -f /usr/local/etc/openldap/demo-groups.ldif
```

### 7.4 Replikation prüfen

Auf **VM2 (Consumer)** sollten die Benutzer automatisch repliziert werden:

```bash
# Auf VM2 ausführen
ldapsearch -x -H ldap://127.0.0.1:389 \
  -D "cn=Manager,dc=fets,dc=local" -w secret \
  -b "ou=People,dc=fets,dc=local"
```

**Erwartetes Ergebnis:** Alle Änderungen vom Provider sollten automatisch auf dem Consumer sichtbar sein.

---

## 8. Apache Webserver mit LDAP-Authentifizierung

Diese Demo zeigt, wie man einen Apache-Webserver mit LDAP-Login einrichtet. Dies kann auf **beiden VMs** gemacht werden, allerdings zeigt es den praktischen Nutzen besser, wenn es auf einer VM läuft, die auch die Benutzer repliziert hat.

### 8.1 Apache installieren

```bash
sudo apt-get update
sudo apt-get install apache2 apache2-utils
```

### 8.2 LDAP-Module aktivieren

```bash
sudo a2enmod authnz_ldap
sudo a2enmod ldap
sudo systemctl restart apache2
```

### 8.3 Demo-Webseiten erstellen

#### Öffentliche Startseite
```bash
sudo vim /var/www/html/index.html
```

Inhalt:
```html
<!-- generated by claude.ai -->
<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <title>LDAP Demo Portal</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background: #f5f5f5;
            padding: 50px;
            margin: 0;
        }
        .container {
            max-width: 800px;
            margin: 0 auto;
        }
        .header {
            background: #2196F3;
            color: white;
            padding: 30px;
            border-radius: 10px;
            text-align: center;
        }
        .content {
            background: white;
            padding: 30px;
            margin-top: 20px;
            border-radius: 10px;
            box-shadow: 0 2px 5px rgba(0,0,0,0.1);
        }
        .login-button {
            display: inline-block;
            background: #4CAF50;
            color: white;
            padding: 15px 30px;
            text-decoration: none;
            border-radius: 5px;
            font-size: 18px;
            margin: 20px 0;
        }
        .login-button:hover {
            background: #45a049;
        }
        table {
            width: 100%;
            border-collapse: collapse;
            margin: 20px 0;
        }
        th, td {
            padding: 12px;
            text-align: left;
            border-bottom: 1px solid #ddd;
        }
        th {
            background: #2196F3;
            color: white;
        }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>🌐 OpenLDAP Demo Portal</h1>
            <p>Willkommen zur LDAP-Authentifizierungs-Demo</p>
        </div>

        <div class="content">
            <h2>Öffentlicher Bereich</h2>
            <p>Diese Seite ist öffentlich - kein Login erforderlich.</p>

            <center>
                <a href="/protected/" class="login-button">
                    🔐 Zum geschützten Bereich
                </a>
            </center>

            <h3>📋 Test-Benutzer für LDAP-Login:</h3>
            <table>
                <tr>
                    <th>Benutzername</th>
                    <th>Passwort</th>
                    <th>Name</th>
                </tr>
                <tr>
                    <td><code>mmueller</code></td>
                    <td><code>geheim123</code></td>
                    <td>Max Mueller</td>
                </tr>
                <tr>
                    <td><code>aschmidt</code></td>
                    <td><code>geheim456</code></td>
                    <td>Anna Schmidt</td>
                </tr>
            </table>

            <h3>💡 Was passiert beim Login?</h3>
            <ol>
                <li>Du klickst auf "Zum geschützten Bereich"</li>
                <li>Browser zeigt Login-Dialog</li>
                <li>Du gibst Username + Passwort ein</li>
                <li>Apache sendet Anfrage an LDAP-Server</li>
                <li>LDAP prüft die Credentials</li>
                <li>Bei Erfolg: Zugriff gewährt! ✅</li>
            </ol>

            <h3>⚠️ Hinweis zum Abmelden</h3>
            <p>Bei HTTP Basic Auth gibt es kein echtes Logout. Um dich abzumelden:</p>
            <ul>
                <li>Schließe <strong>alle</strong> Browser-Fenster/Tabs</li>
                <li>Oder nutze einen <strong>privaten/inkognito</strong> Browser-Tab für Tests</li>
                <li>Oder lösche die Browser-Cookies</li>
            </ul>
        </div>
    </div>
</body>
</html>
```

#### Geschützter Bereich
```bash
sudo mkdir -p /var/www/html/protected
sudo vim /var/www/html/protected/index.html
```

Inhalt:
```html
<!-- generated by claude.ai -->
<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <title>LDAP Login Erfolg</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            padding: 50px;
            margin: 0;
        }
        .container {
            max-width: 800px;
            margin: 0 auto;
            background: rgba(255, 255, 255, 0.1);
            padding: 40px;
            border-radius: 15px;
            backdrop-filter: blur(10px);
        }
        h1 { font-size: 2.5em; margin-bottom: 20px; }
        .success { 
            background: rgba(76, 175, 80, 0.3);
            padding: 20px;
            border-radius: 10px;
            margin: 20px 0;
        }
        .info {
            background: rgba(33, 150, 243, 0.3);
            padding: 20px;
            border-radius: 10px;
            margin: 20px 0;
        }
        ul { text-align: left; }
        li { margin: 10px 0; }
        a { color: white; }
    </style>
</head>
<body>
    <div class="container">
        <h1>🔒 Geschützter Bereich</h1>
        
        <div class="success">
            <h2>✅ Login erfolgreich!</h2>
            <p>Du hast dich erfolgreich mit deinen LDAP-Credentials authentifiziert.</p>
        </div>
        
        <div class="info">
            <h3>🎉 Das funktioniert hier:</h3>
            <ul>
                <li>✅ Apache Webserver ist mit OpenLDAP verbunden</li>
                <li>✅ Dein Browser hat Username + Passwort gesendet</li>
                <li>✅ LDAP-Server hat deine Credentials geprüft</li>
                <li>✅ Authentifizierung war erfolgreich!</li>
                <li>✅ Du siehst jetzt diese geschützte Seite</li>
            </ul>
        </div>

        <div class="info">
            <h3>📋 Anwendungsfälle:</h3>
            <ul>
                <li>Intranet-Seiten schützen</li>
                <li>Web-Anwendungen mit LDAP verbinden</li>
                <li>Single Sign-On für alle Web-Services</li>
                <li>Zentrale Benutzerverwaltung</li>
            </ul>
        </div>

        <p><a href="/">← Zurück zur Startseite</a></p>
    </div>
</body>
</html>
```

### 8.4 Apache Virtual Host konfigurieren

```bash
sudo vim /etc/apache2/sites-available/ldap-demo.conf
```

Inhalt (ersetze `127.0.0.1` mit der IP des LDAP-Servers falls nicht lokal):

```apache
<VirtualHost *:80>
    ServerAdmin admin@fets.local
    DocumentRoot /var/www/html
    ServerName ldap-demo.fets.local

    ErrorLog ${APACHE_LOG_DIR}/ldap-demo-error.log
    CustomLog ${APACHE_LOG_DIR}/ldap-demo-access.log combined

    # Geschützter Bereich mit LDAP
    <Directory /var/www/html/protected>
        AuthType Basic
        AuthName "LDAP Login - Bitte Username und Passwort eingeben"
        AuthBasicProvider ldap
        
        # LDAP-Server-Konfiguration
        AuthLDAPURL "ldap://127.0.0.1:389/ou=People,dc=fets,dc=local?uid?sub?(objectClass=inetOrgPerson)"
        AuthLDAPBindDN "cn=Manager,dc=fets,dc=local"
        AuthLDAPBindPassword secret
        
        # Alle gültigen LDAP-Benutzer erlauben
        Require valid-user
    </Directory>

    # Öffentlicher Bereich
    <Directory /var/www/html>
        Options Indexes FollowSymLinks
        AllowOverride None
        Require all granted
    </Directory>

</VirtualHost>
```

### 8.5 Site aktivieren

```bash
# Default-Site deaktivieren (damit keine Konflikte entstehen)
sudo a2dissite 000-default.conf

# LDAP-Demo-Site aktivieren
sudo a2ensite ldap-demo.conf

# Apache neu starten
sudo systemctl restart apache2
```

### 8.6 Testen

#### IP-Adresse ermitteln
```bash
hostname -I
```

#### Im Browser öffnen
```
http://IP_DES_SERVERS/
```

**Öffentliche Seite:** Sollte ohne Login sichtbar sein

```
http://IP_DES_SERVERS/protected/
```

**Geschützter Bereich:** Browser fordert Login-Dialog

#### Login-Credentials
-  `mmueller`: `geheim123`
-  `aschmidt`: `geheim456`

