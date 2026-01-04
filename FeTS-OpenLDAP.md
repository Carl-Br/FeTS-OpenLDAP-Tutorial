# OpenLDAP Replikation

**Setup-Anleitung für Provider-Consumer-Replikation**

---

## Inhaltsverzeichnis

1. [Vorbereitung - IPv4-Adressen konfigurieren](#1-vorbereitung---ipv4-adressen-konfigurieren)
2. [OpenLDAP Installation (auf beiden VMs)](#2-openldap-installation-auf-beiden-vms)
3. [Basiskonfiguration](#3-basiskonfiguration)
4. [Provider-Konfiguration (VM1)](#4-provider-konfiguration-vm1)
   - 4.1 SLAPD starten
   - 4.2 Base-Daten einfügen
   - 4.3 Replikations-Konfiguration
   - 4.4 Replicator-Benutzer erstellen
5. [Consumer-Konfiguration (VM2)](#5-consumer-konfiguration-vm2)
6. [Replikation validieren](#6-replikation-validieren)
7. [Demo - Benutzer und Organisationsstruktur anlegen](#7-demo---benutzer-und-organisationsstruktur-anlegen)
   - 7.0 Schemas für Benutzer-Objekte prüfen und hinzufügen
   - 7.1 Organisationsstruktur aufbauen
   - 7.2 Benutzer anlegen
   - 7.3 Gruppen erstellen
8. [Vikunja (Todo App) mit LDAP-Authentifizierung](#8-Vikunja-(Todo-App)-mit-LDAP-Authentifizierung)
   - 8.0 nginx proxy
   - Vikunja

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

## 8. Vikunja (Todo App) mit LDAP-Authentifizierung
Erstelle eine dritte VM und gehe wieder sicher, dass die VM eine einmalige IP hat.

### 8.1 NGINX Proxy 

```bash
sudo apt update
sudo apt install nginx
```

```bash
sudo vim /etc/nginx/nginx.conf
```

> [!Important]
> Ersetze `IP_Provider` und `IP_Consumer`
```conf
user www-data;
worker_processes auto;
pid /run/nginx.pid;

include /usr/share/nginx/modules-available/mod-stream.conf;

events {
    worker_connections 768;
}

stream {
    upstream ldap_backend {
        server IP_Provider:389 max_fails=2 fail_timeout=5s;
        server IP_Consumer:389 backup;
    }

    server {
        listen 389;
        proxy_pass ldap_backend;
    }
}

http {
        access_log /var/log/acces.log;
}
```

```bash
sudo apt install nginx libnginx-mod-stream
```
config testen
```bash
sudo nginx -t
```

restart
```bash
sudo systemctl restart nginx
sudo systemctl enable nginx
```

### 8.2 vikunja installieren

```bash
sudo apt install docker-compose
```

```bash
cd; mkdir vikunja; cd vikunja
```

```bash
sudo vim docker-compose.yaml
```

### 8.3 Demo-Webseiten erstellen

#### Öffentliche Startseite
```bash
sudo vim /var/www/html/index.html
```
ersetze den Inhalt mit:
```

version: '3'

services:
  db:
    image: mariadb:10
    command: --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci
    environment:
      MYSQL_ROOT_PASSWORD: ein_sehr_sicheres_passwort
      MYSQL_USER: vikunja
      MYSQL_PASSWORD: vikunja_db_passwort
      MYSQL_DATABASE: vikunja
    volumes:
      - ./db:/var/lib/mysql
    restart: unless-stopped

  vikunja:
    image: vikunja/vikunja
    container_name: vikunja
    environment:
      # Datenbank-Anbindung
      VIKUNJA_DATABASE_HOST: db
      VIKUNJA_DATABASE_PASSWORD: vikunja_db_passwort
      VIKUNJA_DATABASE_TYPE: mysql
      VIKUNJA_DATABASE_USER: vikunja
      VIKUNJA_DATABASE_DATABASE: vikunja

      # Öffentliche URL (Wichtig für das Frontend)
      VIKUNJA_SERVICE_PUBLICURL: http://localhost:3456

      # --- LDAP KONFIGURATION ---
      VIKUNJA_AUTH_LDAP_ENABLED: "true"
      # Hier die IP deines Nginx-LDAP-Proxys eintragen:

      VIKUNJA_AUTH_LDAP_USETLS: "false"
      VIKUNJA_AUTH_LDAP_HOST: "192.168.205.53"
      VIKUNJA_AUTH_LDAP_PORT: 389
      VIKUNJA_AUTH_LDAP_BASEDN: "ou=People,dc=fets,dc=local"
      # Der User, mit dem Vikunja im LDAP sucht (Bind User)
      VIKUNJA_AUTH_LDAP_BINDDN: "cn=Manager,dc=fets,dc=local"
      VIKUNJA_AUTH_LDAP_BINDPASSWORD: "secret"
      # Suchfilter: %[1]s wird durch den eingegebenen Nutzernamen ersetzt
      VIKUNJA_AUTH_LDAP_USERFILTER: "(&(objectClass=inetOrgPerson)(uid=%[1]s))"
      # Welches Attribut im LDAP ist der Username?
      VIKUNJA_AUTH_LDAP_ATTRIBUTE_USERNAME: "uid"

    ports:
      - "3456:3456"
    volumes:
      - ./files:/app/vikunja/files
    depends_on:
      - db
    restart: unless-stopped
```

### starte vikunja
```bash
sudo docker-compose up -d
```

Öffne localhost:3456 im Browser und melde dich an. Du kannst zum testen eine der beiden OpenLDAP VMs aussschalten und das anmleden funktioniert immer noch.

#### Login-Credentials
-  `mmueller`: `geheim123`
-  `aschmidt`: `geheim456`


