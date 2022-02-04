
# Installieren von Smartstore Core auf Ubuntu 21.10 auf einem VPS bei OVHcloud

## Voraussetzungen

 - Smartstore Core für Linux X64
 - Ubuntu 20.04.03
 - Nicht root Benutzer mit sudo-Rechten

## Ablauf

- .NET Core installieren
 - NGINX Reverse Proxy installieren
 - FTP-Server installieren
 - Firewall anpassen
 - MySQL installieren
 - Smartstore installieren
 - Tipps & Tricks

## Allgemeines zum VPS

 - SSH ist installiert und akzeptiert problemlos eine Verbindung z.B. per PuTTY.
 - FTP ist nicht installiert

## .NET Core installieren
>**Wichtig** Dieses Kapitel kann übersprungen werden, wenn ein frameworkunabhängiges Release von Smartstore installiert werden soll.

Microsoft-Paketsignaturschlüssel zu der Liste vertrauenswürdiger Schlüssel einfügen
```bash
wget https://packages.microsoft.com/config/ubuntu/20.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
rm packages-microsoft-prod.deb
```
Installieren der Runtime
```bash
sudo apt-get update; \
  sudo apt-get install -y apt-transport-https && \
  sudo apt-get update && \
  sudo apt-get install -y dotnet-sdk-6.0
  ```

Prüfen der Installation
```bash
dotnet --info
```

## NGINX installieren

Prüfen, ob NGINX bereits installiert ist:
   ```bash
systemctl status nginx
   ```

 ### Das System aktualisieren und NGINX installieren:

   ```bash
   sudo apt-get update
   ```
   ```bash
   sudo apt-get install nginx
   ```

 ### Installation prüfen
   ```bash
   nginx -v
   ```

 ### NGINX starten, stoppen, neu starten und Konfiguration neu laden:
   ```bash
sudo systemctl start nginx
```
 ```bash
sudo systemctl stop nginx
  ```
```bash
sudo systemctl restart nginx
 ```
 ```bash
sudo systemctl reload nginx
   ```

 ### NGINX-Dienst zum Systemstart hinzufügen:
   ```bash
sudo systemctl enable nginx
   ```
 ### Die NGINX-Konfiguration neu laden und Dienst starten:
 ```bash
sudo systemctl reload nginx
  ```


  

## Firewall regeln anpassen

> ** Wichtig ** Auf meinem Test-VPS war die Firewall bei Auslieferung ausgeschaltet.
> Wenn die Firewall aktiviert werden soll, dann muss neben dem Nginx Full-Profil auch das OpenSSH-Profil aktiviert werden!

 ### Liste der bereits eingerichteten Anwendungsprofile ausgeben:

   ```bash
   sudo ufw app list
   ```
> **Hinweis:** Wird der Befehl mit 
> ```bash
> sudo: ufw: Befehl nicht gefunden
 >  ```
 > quittiert, dann ist keine Firewall installiert und dieser Punkt kann erst einmal übersprungen werden.

 ### In der ausgebenen Liste sind drei NGINX-Profile vorhanden:
 
- **Nginx Full**: Dieses Profil öffnet Port 80 und 443 für NGINX
- **Nginx HTTP**: Dieses Profil öffnet nur Port 80 für NGINX
- **Nginx HTTPS**: Dieses Profil öffnet nur Port 443 für NGINX
 
### Port 80 und Port 443 für NGINX zulassen:
   ```bash
   sudo ufw allow 'Nginx FULL'
   ```
   
### OpenSSH zulassen:
   ```bash
   sudo ufw allow 'OpenSSH'
   ```
   
### Das Ergebnis prüfen:
   ```bash
   sudo ufw status
   ```

### Firewall aktivieren
   ```bash
   sudo ufw enable
   ```
 

 ## NGINX einrichten
 ### Standard NGINX-Seite aufrufen
 - IP-Adresse herausfinden, wenn unbekannt
    ```bash
   hostname -I
   ```
- Im Browser NGINX-Startseite per IP aufrufen
	 ```bash
	http://ip-adresse
	```
- Es wird die Standard-Landingpage für NGINX angezeigt

	![NGINX-Landingspage](https://www.smartstore.com/news/images/Qs7PlUtvga.png)

### NGINX als Reverse-Proxy konfigurieren
Folgende Datei mit einem Editor öffnen und den Inhalt durch den Codeausschnitt ersetzen:
	 ```bash
	/etc/nginx/sites-available/default
	```

```bash
server {
	listen        80;
	server_name   example.com *.example.com;
	location / {
			proxy_pass         http://127.0.0.1:5000;
			proxy_http_version 1.1;
			proxy_set_header   Upgrade $http_upgrade;
			proxy_set_header   Connection keep-alive;
			proxy_set_header   Host $host;
			proxy_cache_bypass $http_upgrade;
			proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
			proxy_set_header   X-Forwarded-Proto $scheme;
    		   }
       }
```

```example.com``` wird durch die eigene Domain ersetzt.
Wenn noch keine Domain vorhanden ist, kann auch eine IP-Adresse eingetragen werden:
```bash
server_name 172.16.1.17;
```
Hiernach muss die NGINX-Konfiguration neu geladen werden:
```bash
sudo systemctl reload nginx
```


## MySQL installieren

 ### Das Paket ```mysql-server``` installieren:

**MySQL Repository konfigurieren**
Lokalen Paketindex aktualisieren:
   ```bash
sudo apt update
   ```
   ```MySQL``` installieren:
   ```bash
sudo apt install mysql-server
   ```

   Sicherheitsskript ausführen:
   ```bash
   sudo mysql_secure_installation
   ```
>Die Einstellungen hier können je nach individuellen Sicherheitsanforderungen variieren.

Um die Benutzerauthentifizierung- und Berechtigungen anzupassen MySQL-Eingabeaufforderung öffnen:
   ```bash
   mysql -u root -p
   ```
Authentifizierungsverfahren prüfen:
```bash
SELECT user,authentication_string,plugin,host FROM mysql.user;
   ```
Wird der ```root``` Benutzer über das ```auth-socket```-Plugin authentifiziert, muss das ```root```-Konto umkonfiguriert werden. Mit diesem Befehl wird das vorherige ```root```-Passwort geändert. Es sollte ein starkes Passwort gewählt werden (```password``` ersetzen durch eigenes).
```bash
   ALTER USER 'root'@'localhost' IDENTIFIED WITH caching_sha2_password BY 'password';
   ```
Berechtigungstabellen neu laden:
```bash
FLUSH PRIVILEGES;
   ```
MySQL-Shell verlassen:
```bash
exit
   ```
Einen dedizierten MySQL-Benutzer für die Nutzung mit Smartstore erstellen:
```bash
mysql -u root -p
   ```
```bash
CREATE USER 'smartstore'@'localhost' IDENTIFIED BY 'password';
   ```
> ```smartstore``` und ```password``` nach belieben ändern

Benutzerberechtigungen erteilen:
```bash
GRANT ALL PRIVILEGES ON *.* TO 'smartstore'@'localhost' WITH GRANT OPTION;
   ```
MySQL-Shell verlassen:
```bash
exit
   ```


## FTP-Server ```vsftp``` installieren

### Installieren
System aktualisieren und ```vsftp``` installieren:
```bash
sudo apt update
   ```
```bash
sudo apt install vsftpd
   ```
 
 Den aktuellen Status prüfen:
```bash
sudo service vsftpd status
   ``` 
 

### Firewall anpassen
Port 20 und 21 für FTP und Ports 40000 bis 50000 für passive FTP-Verbindungen öffnen. Wenn TLS aktiviert werden soll, muss noch Port 990 geöffnet werden:

```bash
sudo ufw allow 20/tcp
   ``` 
```bash
sudo ufw allow 21/tcp
   ``` 
```bash
sudo ufw allow 40000:50000/tcp
   ``` 
```bash
sudo ufw allow 990/tcp
   ``` 
 Status prüfen:
 ```bash
sudo ufw status
   ``` 

### Benutzer anlegen
Einen FTP-Benutzer anlegen:
 ```bash
sudo adduser ftpuser
   ``` 
 Wenn ```ftpuser``` keinen SSH-Zugang haben soll, muss dieser in der SSH-Konfigurationsdatei deaktiviert werden:
  ```bash
sudo nano /etc/ssh/sshd_config
   ``` 
 An das Ende der Konfigurationsdatei einfügen:
   ```
DenyUsers ftpuser
   ``` 
Nachdem Speichern der Konfigurationsdatei den SSH-Dienst neu starten:
  ```bash
sudo service sshd restart
   ``` 

### Ordner-Berechtigungen

Wir möchten Dateien in den ```/var/www/html```-Ordner übertragen, deswegen setzen wir den den Ordner drüber als Home-Ordner für den ```ftpuser```:
  ```bash
sudo usermod -d /var/www ftpuser
   ``` 
 Den   ```ftpuser``` als Besitzer von ```/var/www/html``` setzen:
   ```bash
sudo chown ftpuser:ftpuser /var/www/html
   ``` 

### Konfigurieren

Die vorhandene Konfigurationsdatei umbenennen:
   ```bash
sudo mv /etc/vsftpd.conf /etc/vsftpd.conf.bak
   ``` 
Eine neue Konfigurationsdatei erstellen und mit ```nano``` öffnen:
   ```bash
sudo nano /etc/vsftpd.conf
   ``` 

Folgendes hinzufügen:
```
listen=NO
listen_ipv6=YES
anonymous_enable=NO
local_enable=YES
write_enable=YES
local_umask=022
dirmessage_enable=YES
use_localtime=YES
xferlog_enable=YES
connect_from_port_20=YES
chroot_local_user=YES
secure_chroot_dir=/var/run/vsftpd/empty
pam_service_name=vsftpd
force_dot_files=YES
pasv_min_port=40000
pasv_max_port=50000
```

### ```vsftp``` neu starten
```bash
sudo systemctl restart vsftpd
``` 


## Smartstore installieren
### Dateien übertragen
Die Dateien aus dem Release per FTP auf den Ubuntu-Server in den Ordner ```/var/www/html``` 
übertragen.
      	
### App als Dienst einrichten
Erstellen einer Dienstdefinitionsdatei für ```systemd```:
```bash
sudo nano /etc/systemd/system/kestrel-smartstore.service
``` 
Folgenden Code-Ausschnitt einfügen und speichern:
> Bitte die Hinweise unter dem Codeblock beachten!
```bash
[Unit]
Description=Smartstore Core Web App running on Linux

[Service]
WorkingDirectory=/var/www/html
ExecStart=/usr/bin/dotnet /var/www/html/Smartstore.Web.dll
Restart=always
#Restart service after 10 seconds if dotnet service crashes:
RestartSec=10
KillSignal=SIGINT
SyslogIdentifier=smartstore-core
User=www-data
Environment=ASPNETCORE_ENVIRONMENT=Production
Environment=DOTNET_PRINT_TELEMETRY_MESSAGE=false

[Install]
WantedBy=multi-user.target
``` 
> **Hinweis**: Pfade in ```WorkingDirectory``` und ```ExecStart``` ggf. anpassen.

> **Wichtig**: 
> Code bei **frameworkabhängiger Bereitstellung**:
>```bash
>ExecStart=/usr/bin/dotnet /var/www/html/Smartstore.Web.dll
>```
> Code bei **eigenständiger Bereitstellung**:
>```bash
>ExecStart=/var/www/html/Smartstore.Web
>```

### Dienst aktivieren und starten
Dienst aktivieren:
```bash
sudo systemctl enable kestrel-smartstore.service
``` 
Dienst starten:
```bash
sudo systemctl start kestrel-smartstore.service
``` 
Dienst stoppen:
```bash
sudo systemctl stop kestrel-smartstore.service
```

### ```wkhtmltopdf``` installieren
```bash
sudo apt-get update
```
```bash
sudo apt -y install wkhtmltopdf
```

### Festlegen der Ordnerberechtigungen
Eigenen Benutzer als Besitzer des Website-Ordners mit vollen Lese-, Schreib- und Ausführungsrechten setzen:

> **Hinweis:** ```smartstore``` ist beispielhaft der eigene Benutzer
```bash
chown -R smartstore /var/www/html/
```
Webserver als Gruppen-Besitzer setzen:
```bash
chgrp -R www-data /var/www/html/
```
Rekursiv für alle Dateien und Ordner Lese-, Schreib- und Ausführungsrechte für den Besitzer, Lese- und Ausführungsrechte für den Gruppen-Besitzer und keine Rechte für andere festlegen:

```bash
chmod -R 750 /var/www/html/
```
Gruppenbesitz auf neue Dateien und Ordner vererben:

```bash
chmod g+s /var/www/html/
```
Spezielle Ordner rekursiv mit Schreibrechten für Webserver versehen:
```bash
chmod -R g+w /var/www/html/App_Data
```
```bash
chmod -R g+w /var/www/html/Modules
```

### Smartstore installieren
Die Website per IP-Adresse oder Domainname aufrufen und die erforderlichen Daten eingeben.

![Startseite der Installation](https://www.smartstore.com/news/images/smartstore_installation_de_640px.png)

> Der MySQL-Server ist per localhost erreichbar. Als Anmeldename für die Datenbank ist der für diese Installation eigens angelegte MySQL-Benutzer zu verwenden

Nachdem alle erforderlichen Daten eingegeben wurden, wird die Installation per Klick auf **Installieren** gestartet.
Nach der Fertigstellung der Installation erscheint die Startseite mit den Demo-Daten:

![Startseite](https://www.smartstore.com/news/images/smartstore_core_Startseite-640px.png)

### Tipps & Tricks
Um die **maximale Dateiuploadgröße** zu  ändern wird die Datei ```/etc/nginx/nginx.conf``` mit einem Editor geöffnet.
Die Einstellung kann an zwei Stellen vorgenommen werden:

Im Http-Block: Einstellung gilt für alle virtuellen Hosts.
```bash
http {
    ...
    client_max_body_size 100M;
}
```

Im Server-Block: Einstellung gilt nur für diese spezielle Site/App.
```bash
server {
    ...
    client_max_body_size 100M;
}
```


to be continued...


 
 



