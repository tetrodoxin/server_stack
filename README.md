# Installing a server stack with Reverse Nginx, MailCow and Nextcloud based on docker

Das ganze basiert auf diesem Tutorial: https://techdudes.de/2167/reverse-proxy-nextcloud-mailcow-server-installieren
Die dort verwendeten Skripte/Konfig-Dateien sind im unten stehenden Projekt vorbereitet und angepasst. Es bietet sich an, das Tutorial zumindest mal gesehen/gelesen zu haben, es nebenbei mit geöffnet zu haben sollte nicht notwendig sein.

Die Architektur sieht zentralen Reverse-Proxy vor, der Anfragen für die einzelnen (Unter-)Seiten entegennimmt und entsprechend weiterleitet. das Ganze basiert auf _docker_ bzw. _docker compose_.

## Docker installieren

(Debian/Ubuntu like)

1. Installieren
   `sudo apt install docker.io`
2. Automatisch starten lassen
   `sudo systemctl enable --now docker`
3. Optional: Entsprechende User-Accounts zur Gruppe `docker` hinzufügen, sonst _sudo_ für Docker notwendig.
   `usermod -aG docker USER_ACCOUNT_NAME`
4. Docker Compose installieren
   `sudo apt install docker-compose`

## Projektvorlage von _github_ klonen

Das Original schlägt eine Installation unter `/opt` vor, was auch der normalen Linux-Philosophie folgt. Für dieses Verzeichnis Root-Zugriff benötigt man root-Rechte (für Befehle aus diesem Dokument), da man es vermeiden sollte, anderen Benutzern/Gruppen Schreibberechtigungen auf `/opt` Verzeichnisse zu gewähren.

```sh
cd /opt
git clone --recurse-submodules https://github.com/tetrodoxin/server_stack.git server_stack
```

Das sollte als `root` ohne _sudo_ klappen und ein Verzeichnis `/opt/server_stack` erstellen. Das `--recurse-submodules` ist wichtig, damit man das `mailcow-dockerized` sub-module nach dem Klonen nicht noch per Hand nachladen muss.

Das Projekt enthält bereits die im o.g. Tutorial beschriebenen Konfigdateien und benötigt nur noch eine Handvoll Anpassungen.

## Reverse-Proxy

Der Reverse-Proxy beinhaltet einen _NGINX_ samt _Let's-Encrypt_ Client. Letzterer sorgt automatisch für alle "eingeklinkten" Seiten für gültige TLS-Zertifikate. Hier kommt eine Kombination aus dem _jwilder/nginx-proxy_ sowie dem _jrcs/letsencrypt-nginx-proxy-companion_ zum Einsatz.

Die Docker Dateien sind dafür fertig, es reicht also:

`cd frontproxy`
`docker-compose up -d`

das `-d` bewirkt, dass man nach der Ausführung wieder zum Command-Line-Prompt zurückkehrt und nicht im Docker-Prozess samt Ausgaben verbleibt. Wie man sich die Log-Ausgaben eines solchen "detached" Prozesses anschaut, kann man in der Docker Compose Doku lesen (Stichwort `docker-compose logs ...`).

Dann sollte am Ende soetwas stehen wie:
```sh
[+] Running 3/3
 Network frontproxy_network          Created
 Container frontproxy                Started
 Container frontproxy-letsencrypt    Started
```

Der Frontproxy besteht also aus 2 Containern und einem Netzwerk. Er wird erstellte Zertifikate unter `/etc/frontproxy/nginx-certs` ablegen, eine Festlegung in `frontproxy/docker-compose.yml`.
Zudem wurde ein Docker-Netzwerk mit dem Namen `frontproxy_network` erstellt, d.h. ein virtuelles Netzwerk, das von anderen Docker-Containern verwendet werden kann, in unserem Beispiel später die MailCow- und Nextcloud-Komonenten (also Container, Plural).

Danach zurück in das Hauptverzeichnis mit `cd ..` oder auch `cd /opt/server_stack`.

## DNS Einträge für die Postkuh

Hier verwende ich `acme.fuck` als Example-Root-Domain und die Server-IP `1.2.3.4`

### A-Record
- Name : mail 
- Type : A 
- TTL : default 		// oder 300
- Value : 1.2.3.4 

Kann man testen mit (evt Paket `bind-tools` installieren):
```sh
> dig mail.acme.fuck +noall +answer
mail.acme.fuck.	7194	IN	A	1.2.3.4	# Beispielantwort
```

### CNAME records 
- Name : autoconfig 
- Type : CNAME 
- TTL : default 		// oder 14400
- Alias : mail.acme.fuck

- Name : autodiscover 
- Type : CNAME 
- TTL : default 		// oder 14400
- Alias : mail.acme.fuck

Wieder evt. Testen:
```sh
> dig autoconfig.acme.fuck +noall +answer
autoconfig.acme.fuck.	7167	IN	CNAME	mail.acme.fuck
mail.acme.fuck.		7194	IN	A	1.2.3.4	

> dig autodiscover.acme.fuck +noall +answer
autodiscover.acme.fuck.	7170	IN	CNAME	mail.acme.fuck
mail.acme.fuck.		6167	IN	A	1.2.3.4	
```

### MX-Record
- Name : {leer} 
- Type : MX
- Priority : 10
- TTL : default 		// oder 86400
- Value : mail.acme.fuck

### SPF-Record
- Name : {leer} 
- Type : TXT
- TTL : default 		// oder 14400
- Value : v=spf1 mx ~all

```sh
> dig acme.fuck TXT
acme.fuck.	7194	IN	TXT	"v=spf1 mx ~all"
```

### DKIM-Record
Später, da PublicKey noch nicht erstellt


### DMARC-Record
Man muss eine EMail-Adresse als DMARC-Kontakt eintragen, hier beispielhaft `dmarc-inmail@acme.fuck`.

- Name : _dmarc
- Type : TXT
- TTL : default 		// oder 14400
- Value : v=DMARC1; p=reject; rua=mailto:dmarc-inmail@acme.fuck


### Reverse DNS (rDNS)
Beim Hoster entsprechenden rDNS-Eintrag auf `mail.acme.fuck` setzen. Testen:

```sh
> dig -x 1.2.3.4 +noall +answer
1.2.3.4.in-addr.arpa	86400	IN	PTR	mail.acme.fuck
```

## MailCow Server

Als erstes `cd mailcow`

Dort die Mini-Konfig ausführen `./generate_config.sh`.
  _Sollte es hier zu einem Fehler kommen, kann es sein, dass man tatsächlich noch `chmod +x generate_config.sh` ausführen muss_

Der letzte Befehl erzeugt die datei `mailcow.conf`, die man als nächstes noch editiert, um folgendes zu ändern (zu setzen):

```conf
HTTP_PORT=8080
HTTP_BIND=127.0.0.1

HTTPS_PORT=8443
HTTPS_BIND=127.0.0.1



SKIP_LETS_ENCRYPT=y
LETSENC_EMAIL=YOUR_EMAIL_HERE@acme.fuck
```

Die datei `docker-compose.yml` aus dem Tutorial ist bereits vorbereitet. Aber um das manuelle Anpassen für die Umgehung des "Bugs" zu vermeiden, existieren zwei Versionen dieser Datei. Eine für das First-Setup und eine für den Betrieb.

Also, zuerst:
`cp docker-compose.no_certs.yml docker-compose.yml`
Damit wird die "First-Use" Version verwendet. Der Rest ist bereits eingerichtet, da auch festgelegt ist, wo die TLS-Zertifikate liegen (hard coded :-( ). Also.. let's go:

`docker-compose up -d`

Wenn man jetzt anschließend mal spickt:

`ls -la /etc/frontproxy/nginx-certs` sollte dort ein Verzeichnis für die oben eingegebene (also bei `generate_config.sh`) Mail-Domain existieren, mit `key.pem` und `fullchain.pem` darin. Das ist wichtig, sonst kann man TLS vergessen und der ganze Quatsch läuft nicht. Evt. mal ein Minütchen warten.

Wenn die Zertifikate da sind, toll, dann wieder beenden:

`docker-compose down`

und gleich wieder hochfahren
`docker-compose up --build -d`

Damit sollte der MailCow Server laufen und unter der eingegebenen Domain erreichbar sein, sodass man mit dessen Online-Einrichtung (inkl. DKIM) beginnen kann.

## NextCloud

Also entweder mit `cd ..` zurück oder eben wieder `cd /opt/server_stack`
udn dann `cd nextcloud` (duh)

Dort existiert die `.env` Datei noch nicht. Am besten Erstellen mit `cp .env.template .env` und mit dem Lieblingseditor die Werte von `MYSQL_ROOT_PASSWORD`, `MYSQL_PASSWORD`, `NEXTCLOUD_HOSTNAME` sowie `LETSENCRYPT_EMAIL` anpassen. Das `MYSQL_PASSWORD` gehört dann zum User mit dem Namen aus `MYSQL_USER`, der per default 'nextcloud' sein sollte (und darf), das Passwort braucht man später bei der Online-Einrichtung der Nextcloud.

Ja und auch dann nur noch:
`docker-compose up -d`

Dann kann man die NextCloud online unter der konfigurierten Domain erreichen und einrichten. Man muss dort unter dem anzulegendem Benutzer zunächst die DB-Optionen ändern, da die Nextcloud per Default SQLite benutzt und sich auf der Willkommensseite auch darüber beschwert, wegen Performance und so. Dort wählt man dann MySQL aus und gibt die Verbindungsdaten an, also 'nextcloud' als User mit dem Passowrt oben (nicht ROOT), der Datenbank 'nextcloud' und als Server einfach `db`. Der Servername wird in der `docker-compose.yml` durch den Namen des Services festgelegt.

Ja, das war's dann eigentlich im Normalfall.

## Andere Webseiten

Man kann auf die gleiche Weise weitere Dienste einbinden, am besten mit Docker-Compose, die man auf das Docker Netzwerk `frontproxy_network` (als `extern: true` markieren) verweist. Der Frontproxy erkennt diese Container dann v.a. durch die Zuweisung der Variablen `VIRTUAL_HOST` und `LETSENCRYPT_HOST` (siehe bspw docker-compose.yml von Nextcloud). TLS und Weiterleitung macht er dann automatisch.


