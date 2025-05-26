# Galera - HackMyVM (Hard)
 
![Galera.png](Galera.png)

## Übersicht

*   **VM:** Galera
*   **Plattform:** ([https://hackmyvm.eu/machines/machine.php?vm=Galera](https://hackmyvm.eu/machines/machine.php?vm=Galera))
*   **Schwierigkeit:** Hard
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 23. Mai 2025
*   **Original-Writeup:** [https://alientec1908.github.io/Galera_HackMyVM_Hard/](https://alientec1908.github.io/Galera_HackMyVM_Hard/)
*   **Autor:** Ben C.

## Kurzbeschreibung

Die Herausforderung "Galera" auf HackMyVM (Schwierigkeit: Hard) erforderte eine mehrstufige Kompromittierung. Das Ziel war es, Root-Zugriff auf der Maschine zu erlangen. Kernpunkte der Lösung waren die Ausnutzung einer unsicheren Galera Cluster-Konfiguration für den initialen Datenbankzugriff, das Ändern eines Admin-Passworts über die replizierte Datenbank, das Platzieren einer Web-Shell mittels SQL `INTO OUTFILE` und schließlich das Auslesen des Root-Passworts aus dem TTY-Speicher.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `nmap`
*   `nikto`
*   `gobuster` / `ffuf` / `dirbuster`
*   `curl`
*   Burp Suite (impliziert für Request-Analyse/Modifikation)
*   `mysql` (MariaDB Client)
*   Python (für Brute-Force-Skript)
*   `openssl`
*   `wget`
*   `binwalk`
*   `zlib-flate`
*   PhotoRec / `foremost`
*   `whatweb`
*   `hydra`
*   `ssh`
*   Standard Linux-Befehle (`vi`, `grep`, `ls`, `cat`, `id`, `w`, `su`, etc.)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Galera" gliederte sich in folgende Phasen:

1.  **Reconnaissance & Enumeration:**
    *   IP-Findung mit `arp-scan` (192.168.2.201).
    *   Umfassender Portscan mit `nmap` identifizierte offene Ports: SSH (22), HTTP (80) und einen unbekannten Dienst auf Port 4567 (später als Galera Cluster-Kommunikation identifiziert).
    *   Webserver-Enumeration mit `nikto`, `gobuster`, `ffuf` und `dirbuster` deckte `info.php`, `config.php`, ein `/upload`-Verzeichnis und eine `private.php` (initial 403) auf.

2.  **Schwachstellenanalyse & Vorbereitung:**
    *   Analyse von `info.php` zeigte FFI-Support mit `preload` und den Document-Root `/var/www/html`.
    *   Identifizierung einer Path-Traversal-ähnlichen Schwachstelle (`/private.php/../`), um Zugriffsbeschränkungen zu umgehen und frische CSRF-Tokens für das Login-Formular zu erhalten.
    *   Analyse eines heruntergeladenen Bildes (`galera.png`) mit `binwalk` und `PhotoRec` offenbarte versteckte MySQL/MariaDB-Datenbankdateien, die jedoch nicht direkt zum Erfolg führten.
    *   Recherche bestätigte, dass Port 4567 wahrscheinlich für Galera Cluster-Kommunikation verwendet wird.

3.  **Initial Access (Galera Cluster Takeover & SQL-Manipulation):**
    *   Konfiguration der eigenen MariaDB-Instanz, um dem Galera-Cluster des Ziels auf Port 4567 als neuer Knoten beizutreten.
    *   Erfolgreiche Synchronisation (SST) der `galeradb`-Datenbank auf die Angreifer-Maschine.
    *   Auslesen des bcrypt-Passwort-Hashes für den `admin`-Benutzer aus der `users`-Tabelle.
    *   Generieren eines eigenen bcrypt-Hashes für ein bekanntes Passwort und Aktualisieren des `admin`-Passworts in der lokalen Datenbank, was zurück in den Cluster repliziert wurde.
    *   Erfolgreicher Login in die Webanwendung als `admin` mit dem neu gesetzten Passwort.

4.  **Post-Exploitation / Web-Shell & Informationsbeschaffung:**
    *   Überprüfung von `secure_file_priv` (leer) und `FILE`-Privilegien des Datenbankbenutzers.
    *   Hochladen einer PHP-Web-Shell (`revshell.php`) in das Verzeichnis `/var/www/html` mittels `SELECT ... INTO OUTFILE` über die MariaDB-Verbindung.
    *   Ausnutzung einer Schwachstelle, bei der in die Datenbank geschriebener PHP-Code (im E-Mail-Feld eines Testbenutzers) serverseitig ausgeführt wurde, um den Inhalt von `/etc/passwd` (base64-kodiert) zu exfiltrieren.
    *   Identifizierung des Benutzers `donjuandeaustria` aus `/etc/passwd`.

5.  **Privilege Escalation (von Web-Shell/DB zu `donjuandeaustria`):**
    *   Brute-Force-Angriff auf das SSH-Konto von `donjuandeaustria` mit `hydra` und der `rockyou.txt`-Liste, was zum Passwort `amorcito` führte.
    *   Erfolgreicher SSH-Login als `donjuandeaustria`. User-Flag (`user.txt`) gefunden.

6.  **Privilege Escalation (von `donjuandeaustria` zu root):**
    *   Identifizierung einer aktiven Root-Sitzung auf `tty20` mittels des `w`-Befehls.
    *   Auslesen des Inhalts von `tty20` mit `cat /dev/vcs20`.
    *   Auffinden des Root-Passworts (`saG58zJxs8crgQa366Uw`) in Klartext, das als "Reminder" auf der Konsole angezeigt wurde.
    *   Wechsel zum Root-Benutzer mit `su root` und dem gefundenen Passwort. Root-Flag (`root.txt`) gefunden.

## Wichtige Schwachstellen und Konzepte

*   **Unsichere Galera Cluster-Konfiguration:** Erlaubte einem nicht authentifizierten Knoten den Beitritt zum Cluster und die Synchronisation/Manipulation der Datenbank.
*   **SQL `INTO OUTFILE` zum Schreiben von Web-Shells:** Ausnutzung von `FILE`-Privilegien und einer unsicheren `secure_file_priv`-Einstellung in MariaDB.
*   **PHP Code Injection über Datenbankfelder:** Unsachgemäße Behandlung von Datenbankinhalten führte zur serverseitigen Ausführung von PHP-Code, der in Benutzerprofilen gespeichert war (ähnlich einer Stored XSS, die zu RCE eskaliert).
*   **Path Traversal (URL-basiert):** Umgehung einer Zugriffsbeschränkung (403 Forbidden) auf `private.php` durch Anhängen von `/../` an die URL, um CSRF-Tokens zu erhalten.
*   **Passwort-Brute-Force (SSH):** Erfolgreicher Angriff auf ein schwaches Benutzerpasswort mittels `hydra`.
*   **Klartext-Passwort im TTY-Speicher:** Auslesen des Root-Passworts aus `/dev/vcs20` aufgrund einer unsicheren "Reminder"-Nachricht und potenziell lockeren Berechtigungen auf die Gerätedatei.
*   **Steganographie (versucht):** Analyse von Bilddateien auf versteckte Daten (obwohl dies nicht der direkte Weg zum Erfolg war, ist es ein wichtiges Konzept).

## Flags

*   **User Flag (`/home/donjuandeaustria/user.txt`):** `072f9d8c26547db59e65d7aa3e55747b`
*   **Root Flag (`/root/root.txt`):** `6a0d424c13321ca6e3b2deb2295fcc26`

## Tags

`HackMyVM`, `Galera`, `Hard`, `Linux`, `Web`, `Apache`, `PHP`, `MariaDB`, `Galera Cluster`, `SQL Injection`, `INTO OUTFILE`, `Web Shell`, `CSRF Bypass`, `Path Traversal`, `TTY Exploit`, `Password Cracking`, `Hydra`, `Privilege Escalation`
