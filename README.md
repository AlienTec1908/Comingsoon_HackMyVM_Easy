# ComingSoon - HackMyVM (Easy)

![Comingsoon Icon](Comingsoon.png)

## Übersicht

*   **VM:** ComingSoon
*   **Plattform:** [HackMyVM](https://hackmyvm.eu/machines/machine.php?vm=Comingsoon)
*   **Schwierigkeit:** Easy
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 2022-11-14
*   **Original-Writeup:** https://alientec1908.github.io/Comingsoon_HackMyVM_Easy/
*   **Autor:** Ben C.

## Kurzbeschreibung

Die virtuelle Maschine "ComingSoon" von HackMyVM (Schwierigkeitsgrad: Easy) wurde durch eine Kette von Schwachstellen kompromittiert. Der initiale Zugriff erfolgte durch die Aktivierung eines versteckten Dateiuploaders auf der Webseite mittels Manipulation eines Base64-kodierten Cookies. Eine PHP-Reverse-Shell wurde hochgeladen (als `.phtml` getarnt), was zu einer Shell als `www-data` führte. Die Privilegienerweiterung zu Root wurde durch die Ausnutzung der bekannten Kernel-Schwachstelle CVE-2022-0847 ("Dirty Pipe") erreicht, wobei Metasploit Framework zur Identifizierung und Ausführung des Exploits verwendet wurde.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `nmap`
*   `gobuster`
*   `nikto`
*   `curl`
*   `base64` (für Cookie-Manipulation)
*   Burp Suite (impliziert für Cookie-Manipulation und Upload-Modifikation)
*   `nc` (netcat)
*   `python3` (`pty` für Shell-Stabilisierung, `http.server` für File-Hosting)
*   `stty`
*   Standard Linux-Befehle (`find`, `ls`, `cat`, `ss`, `cd`, `env`, `getcap`, `wget`, `chmod`, `id`)
*   `pspy64`
*   Metasploit Framework (`msfconsole`):
    *   `exploit/multi/handler`
    *   `post/multi/manage/shell_to_meterpreter`
    *   `post/multi/recon/local_exploit_suggester`
    *   `exploit/linux/local/cve_2022_0847_dirtypipe`

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "ComingSoon" erfolgte in diesen Schritten:

1.  **Reconnaissance & Web Enumeration:**
    *   Ziel-IP (`192.168.2.116`, Hostname `coming.hmv`) via `arp-scan` und `/etc/hosts` identifiziert.
    *   `nmap` zeigte offene Ports 22 (SSH 8.4p1) und 80 (Apache 2.4.51, Titel "Bolt - Coming Soon Template").
    *   `gobuster` fand `index.php`, `/assets/`, `license.txt` und `notes.txt`.
    *   `notes.txt` enthielt Hinweise auf einen "built-in image uploader" und dass SSH-Passphrasen gleich Passwörtern seien.
    *   `nikto` meldete fehlende Sicherheitsheader und ein Cookie `RW5hYmxlVXBsb2FkZXIK` (Base64 für "EnableUploader") ohne `httponly`-Flag.

2.  **Initial Access (Hidden Uploader & Web Shell):**
    *   Der Wert des Cookies `EnableUploader` wurde von Base64("false") (`ZmFsc2UK`) auf Base64("true") (`dHJ1ZQ`) geändert.
    *   Dies schaltete einen Uploader unter dem Pfad `/5df03f95b4ff4f4b5dabe53a5a1e15d7.php` frei.
    *   Eine PHP-Reverse-Shell (`rev.php`) wurde vorbereitet. Um Dateiendungsfilter zu umgehen, wurde die Anfrage mit Burp Suite abgefangen und der Dateiname im `Content-Disposition`-Header zu `rev.phtml` geändert.
    *   Nach erfolgreichem Upload und Aufruf der Datei wurde eine Reverse Shell als Benutzer `www-data` empfangen.
    *   Die Shell wurde stabilisiert.

3.  **Privilege Escalation Preparation (als www-data):**
    *   Enumeration als `www-data` zeigte einen Benutzer `scpuser`. Die `user.txt` war ein Symlink auf `/media/flags/user.txt` und nicht lesbar. `.oldpasswords` war ebenfalls nicht lesbar.
    *   Beschreibbare Systemd-Service-Dateien wurden gefunden, aber nicht primär genutzt.
    *   `sudo` war nicht verfügbar. Keine ungewöhnlichen SUID-Dateien oder Capabilities.
    *   `pspy64` wurde hochgeladen und ausgeführt, zeigte aber keine sofort ausnutzbaren Cron-Jobs.
    *   Die Kernel-Version wurde als `Linux comingsoon.hmv 5.10.0-9-amd64 #1 SMP Debian 5.10.70-1 (2021-09-30)` identifiziert, bekannt als anfällig für CVE-2022-0847 (Dirty Pipe).

4.  **Privilege Escalation (Dirty Pipe via Metasploit):**
    *   Die `www-data`-Shell wurde in eine Metasploit-Session (Session 1) überführt (`exploit/multi/handler`).
    *   Diese Shell-Session wurde zu einer Meterpreter-Session (Session 2) aufgewertet (`post/multi/manage/shell_to_meterpreter`).
    *   Der `post/multi/recon/local_exploit_suggester` wurde auf Session 2 ausgeführt und identifizierte `exploit/linux/local/cve_2022_0847_dirtypipe` als potenziell erfolgreich.
    *   Das `dirtypipe`-Modul wurde konfiguriert (Session 2, `WRITABLE_DIR=/tmp`) und ausgeführt.
    *   Der Exploit war erfolgreich und öffnete eine neue Meterpreter-Session (Session 3) mit Root-Rechten.
    *   Innerhalb der Root-Meterpreter-Session wurde eine System-Shell geöffnet (`shell`).
    *   Die User- und Root-Flags wurden gelesen.

## Wichtige Schwachstellen und Konzepte

*   **Client-Side Control Bypass:** Aktivierung einer versteckten Funktion (Uploader) durch Manipulation eines Cookies.
*   **Unsicherer Dateiuploader:** Erlaubte das Hochladen und Ausführen einer PHP-Web-Shell (Umgehung von Filtern durch `.phtml`).
*   **Kernel Exploit (CVE-2022-0847 - Dirty Pipe):** Ausnutzung einer bekannten Schwachstelle im Linux-Kernel zur lokalen Privilegienerweiterung.
*   **Verwendung von Metasploit Framework:** Für Session-Management, Exploit-Suche und Ausführung.
*   **Informationslecks in Notizdateien.**
*   **Fehlende Sicherheitsheader.**

## Flags

*   **User Flag (`/home/scpuser/user.txt` -> `/media/flags/user.txt`):** `HMV{user:comingsoon.hmv:58842fc1a7}`
*   **Root Flag (`/root/root.txt`):** `HMV{root:comingsoon.hmv:2339dc81ca}`

## Tags

`HackMyVM`, `ComingSoon`, `Easy`, `Web`, `File Upload`, `Cookie Manipulation`, `PHP Shell`, `Dirty Pipe`, `CVE-2022-0847`, `Kernel Exploit`, `Metasploit`, `Linux`
