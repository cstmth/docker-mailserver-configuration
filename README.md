# Mailserver einrichten

Diese Repository greift auf das [docker-mailserver](https://github.com/docker-mailserver/docker-mailserver) Image zurück, um einen simplen Mailserver zur Verfügung zu stellen.

Dieses Repository enthält die nötigen Konfigurationsdateien (`mailserver.env` und `docker-compose.yml`). Alternativ kann der Mailserver mit der untenstehenden Anleitung auch händisch eingerichtet werden.

# Inhaltsverzeichnis

- [Offizielle Dokumentation](#offizielle-dokumentation)
- [Docker](#docker)
- [DNS Einträge anlegen](#dns-einträge-anlegen)
- [Konfigurationsdateien beschaffen und befüllen](#konfigurationsdateien-beschaffen-und-befüllen)
  - [Möglichkeit 1: Repository klonen](#möglichkeit-1-repository-klonen)
  - [Möglichkeit 2: Selber machen](#möglichkeit-2-selber-machen)
    - [1. working directory anlegen](#1-working-directory-anlegen)
    - [2. Konfigurationsdateien herunterladen](#2-konfigurationsdateien-herunterladen)
    - [3. docker-compose.yml anpassen](#3-docker-composeyml-anpassen)
    - [4. mailserver.env anpassen](#4-mailserverenv-anpassen)
- [Hier fortfahren: Erstes Postfach anlegen](#hier-fortfahren-erstes-postfach-anlegen)
- [SSL Zertifikat erstellen](#ssl-zertifikat-erstellen)
- [postmaster-Alias anlegen](#postmaster-alias-anlegen)
- [DKIM einrichten](#dkim-einrichten)
- [Konfiguration testen](#konfiguration-testen)

# Offizielle Dokumentation
Die offizielle Anleitung ist hier zu finden: https://docker-mailserver.github.io/docker-mailserver/latest/usage/

# Docker

Es wird davon ausgegangen, dass Docker auf dem Host-System installiert ist.

# DNS Einträge anlegen

Folgende DNS Records müssen gesetzt werden:

Typ | Host eigentlich | Host für netcup-Formular* | Wert | Bemerkung
--- | --- | --- | --- | ---
A | mail.carldressler.de | mail | 188.68.35.217 | Auflösung nur (!) von Subdomains
A | @ | @ | 186.68.35.217  | Auflösung von carldressler.de, für SSH Zugriff nötig
MX | @ | @ | mail.carldressler.de | wohin Mails kommen und gehen sollen
TXT | mail.carldressler.de | mail | v=spf1 ~all | SPF Record -> wer Mails von carldressler.de senden darf
TXT | _dmarc.mail.carldressler.de | _dmarc | v=DMARC1; p=quarantine; rua=mailto:postmaster@carldressler.de; ruf=mailto:postmaster@carldressler.de; fo=0:1; | DMARC Record -> was mit Mails mit schlechtem Score passieren soll
TXT | mail._domainkey | mail._domainkey | muss neu generiert werden | DKIM -> Signatur von E-Mails, dass sie wirklich von carldressler.de stammen

⚠️ *[netcup erwartet die Host-Einträge ohne Domain](https://www.netcup-wiki.de/wiki/Domains_CCP#DNS). Hier ist ein Beispiel: https://imgur.com/a/6Iu0dHd

# Konfigurationsdateien beschaffen und befüllen

## Möglichkeit 1: Repository klonen

Dieses Repository enthält die Konfigurationsdateien `mailserver.env` und `docker-compose.yml`.

⚠️ Das Repository nutzt `mail.carldressler.de` als Domain für den Mailserver. Die Domain sollte jedoch problemlos anpassbar sein. Dann müssen aber auch die [DNS Einträge](#dns-einträge-anlegen) angepasst werden.

```bash
git clone https://github.com/cstmth/docker-mailserver-config.git ~/docker-mailserver &&
cd ~/docker-mailserver
```

## Möglichkeit 2: Selber machen

### 1. working directory anlegen
```bash
mkdir ~/docker-mailserver &&
cd ~/docker-mailserver
```

### 2. Konfigurationsdateien herunterladen
```bash
wget https://raw.githubusercontent.com/docker-mailserver/docker-mailserver/master/mailserver.env && 
wget https://raw.githubusercontent.com/docker-mailserver/docker-mailserver/master/docker-compose.yml
``` 
### 3. docker-compose.yml anpassen

1. `hostname: mail.carldressler.de`
2. Volumes ergänzen um `/etc/letsencrypt:/etc/letsencrypt`
3. SSL_TYPE setzen:
```yaml
environment:
  - SSL_TYPE=letsencrypt
```
### 4. mailserver.env anpassen

1. `TZ=` > `TZ=Europe/Berlin`
2. `SSL_TYPE` > `SSL_TYPE=letsencrypt`

Damit ist die Konfiguration abgeschlossen.

# Hier fortfahren: Erstes Postfach anlegen

Jetzt muss das Image gestartet werden. Dann haben wir 120 Sekunden Zeit, ein Postfach anzulegen. Ist das geschafft fährt der Server herunter weil er feststellt, dass kein gültiges SSL-Zertifikat vorliegt.

```bash
docker compose up -d &&
docker exec -ti mailserver setup email add ich@carldressler.de
```

# SSL Zertifikat erstellen

Jetzt legen wir mit Certbot von Let'sEncrypt ein SSL-Zertifikat an, damit E-Mails verschlüsselt sind und der Mailserver nicht herunterfährt.

```bash
sudo snap install core; sudo snap refresh core &&
sudo snap install --classic certbot &&
sudo ln -s /snap/bin/certbot /usr/bin/certbot &&
sudo certbot certonly
# ToS mit Y zustimmen, Werbung mit N ablehnen
# wenn gefragt wird mit "1" temporären Webserver starten
# danach als Domain `mail.carldressler.de` angeben
```

Danach muss der Container neugestartet werden, damit das Zertifikat übernommen wird.

```bash
docker compose down &&
docker compose up -d
```

# postmaster-Alias anlegen
Den postmaster-Alias zu setzen empfiehlt sich, weil dorthin mögliche Fehlermeldungen gesendet werden. Diese werden dann an `ich@carldressler.de` weitergeleitet.

```bash
docker exec -ti mailserver setup alias add postmaster@carldressler.de ich@carldressler.de
```

# DKIM einrichten

Zuerst legen wir einen DKIM Key an. Danach muss der Container neugestartet werden.

```bash
docker exec -ti mailserver setup config dkim keysize 2048 && 
docker compose down &&
docker compose up -d
```

Danach muss der DNS Record für DKIM gesetzt werden. Ausgeben lässt du ihn dir so:

```bash
sudo less ./docker-data/dms/config/opendkim/keys/carldressler.de/mail.txt
``` 

Zeilenumbrüche und Anführungszeichen entfernen, dann beginnend mit `v=DKIM1` bis zum Ende der langen Zeichenkette kopieren und als TXT Record mit dem Host `mail._domainkey` setzen.

# Konfiguration testen

Prüfe zuerst, ob [alle notwendigen Records](#dns-einträge-anlegen) propagiert sind: https://dnschecker.org/all-dns-records-of-domain.php

Dann kannst du mit https://www.mail-tester.com/ testen, ob alles funktioniert.
