# ffmuc-offloader_rpb4
## Bau eines Freifunk-Offloaders auf Basis eines Raspberry 4B
Bei größeren Installationen, wie z.B. in Flüchtlingsunterkünften, Wohnheimen, oder Veranstaltungen bzw. Veranstaltungsräumen gelangt man mit den gängigen Plasteroutern sehr schnell an deren Leistungsgrenzen. Die zeigt sich vor allem beim Datendurchsatz, da die in den Routern verbauten CPUs an ihre Grenzen kommen, da diese nicht für eine permanente Ver- und Entschlüsselung von großen Datenmengen ausgelegt sind. 

In diesem Konfigurationsbeispiel wollen wir möglichst einfach und schnell einen Offloader für **[Freifunk München](https://ffmuc.net)** auf Basis eines **[Raspberry 4B](https://www.raspberrypi.org/products/raspberry-pi-4-model-b/)** befassen. Dabei gehen wir auf unterschiedliche Konfigurations-Optionen ein und wollen dennoch die Einstiegshürden für den ungeübteren Ansible und Linux-User möglichst tief ansetzen. 

### Voraussetzungen ###
Nicht zwingende Voraussetzung für den Bau eines Offloaders mit Hilfe des Ansible-Playbooks hier sind Kenntnisse zu **[Secure Shell](https://dokuwiki.nausch.org/doku.php/centos:ssh_c7)** und **[Ansible](https://dokuwiki.nausch.org/doku.php/centos:ansible:start)**. Die erleichtern natürlich den Umgang ungemein.

### Grundlagen zu SSH ###
Grundlegende Informationen rund um die **[Secure Shell](https://www.openssh.com/)** und deren Umgang bei der täglichen Arbeit finden sich im Kapitel **[Secure Shell in Djangos WIKI](https://dokuwiki.nausch.org/doku.php/centos:ssh_c7)**.

#### Erstellen eines SSH-Schlüsselpaares ####
Damit wir selbst für spätere Administrationsaufgaben und auch unser Ansible-Admin-Host Verbindungen mit Hilfe der **SSH** zu unserem Offloader aufbauen kann, benötigen wir natürlich entsprechendes Schlüsselmaterial, abhängig vom jeweiligen Zielsystem bzw. den kryptographischen Eigenschaften.  

Bei der Erstellung wollen wir statt eines **[RSA-Keys](https://de.wikipedia.org/wiki/RSA-Kryptosystem)** auf zeitgemäße und sicherere **[ed25519](http://ed25519.cr.yp.to/)** Schlüssel verwenden. **[Ed25519](https://de.wikipedia.org/wiki/Curve25519)** ist ein Elliptic Curve Signature Schema, welches beste Sicherheit bei vertretbaren Aufwand verspricht, als ECDSA oder DSA dies der Fall ist. Zur Auswahl sicherer kryptografischer Kurven bei der *Elliptic-Curve Cryptography* findet man auf der Seite [hier](https://safecurves.cr.yp.to/) hilfreiche Erklärungen und eine Gegenüberstellung der möglichen verschiedenen Alternativen. Weitere Infos zu den ED25519-Schlüsseln und der SSH finden sich im entsprechenden Kapitel in **[Djangos WIKI](https://dokuwiki.nausch.org/doku.php/centos:ssh_c7#ed25519_key)**.

Wir erstellen uns nun einen **ED25519**-Schlüssel (`-t`), mit einer festen Schlüssellänge. Der Parameter (`-a`) beschreibt dabei die Anzahl der KDF-Schlüsselableitfunktion (siehe manpage von ssh-keygen). Wir verwenden als Beschreibung **FFMUC Remote-User** (`-C`) und als Ziel-/Speicherort ***~/.ssh/id_ed25519_ffmuc*** (`-f`).

` $ ssh-keygen -t ed25519 -a 100 -C 'FFMUC Remote-User' -f ~/.ssh/id_ed25519_ffmuc`

### Grundlagen zu Ansible ###
Grundlegende Informationen zu **[Ansible](https://www.ansible.com/)** und dessen Umgang bei der täglichen Arbeit finden sich im **[entsprechend Kapitel von Djangos WIKI](https://dokuwiki.nausch.org/doku.php/centos:ansible:start)**. 

### Installation von Ansible und Git ###
Je nach verwendeter Systemumgebung installieren wir nun das vom Paketmaintainer zur Verfügung gestellte  **RPM** bzw. **DEB** Paket. Im Falle von RedHat basierenden Systemen wie **[CentOS](https://www.centos.org/)** oder **[Fedora](https://getfedora.org/)** benutzen wir das Paketverwaltungswerkzeug **YUM/DNF** oder im Falle von **[Debian](https://www.debian.org/)** und **[Ubuntu](https://ubuntu.com/)** das Werkzeug **APT**.
* RPM basierende Systeme: ` # dnf install ansible git` bzw. als eingeloggter User via ***sudo*** mit ` $ sudo yum install ansible git`
* DEB basierende Systeme: ` # apt install ansible git` bzw. als eingeloggter User via ***sudo*** mit ` $ sudo apt install ansible git`

### Einrichten der eigenen Ansible-Umgebung ###
#### Ansible Arbeitsverzeichnis ####
Für unsere Ansible-Aufgaben (Playbooks) legen wir uns nun im Home-Verzeichnis unseres Admin-Users ein zugehöriges Verzeichnis an. 

` $ mkdir ~/ansible`

#### Ansible-Konfigurationsdatei ####
Als nächstes kopieren wir uns die Vorlage-Konfiguratinsdatei aus dem Verzeichnis `/etc/ansible/` in unser Homeverzeichnis. 

` $ cp /etc/ansible/ansible.cfg ~/.ansible.cfg`

Unter dem Konfigurationsgruppe **[ defaults ]** setzen wir den Parameter `inventory = ~/ansible/inventories/production/hosts` und `interpreter_python = auto_silent`. Beim ersten Parameter zeigen wir Ansible, wo die Host-Definitionen zu finden sind. beim Parameter `interpreter_python` geben wir an, dass __keine__ Ausgabe zu den Angaben des Pfads zum Python-Interpreter auf dem Raspberry ausgegeben werden soll. Der Parameter `connect_timeout` definiert, wie lange eine persistente Verbindung bestehen darf, bevor diese hart getrennt wird. Weitere Information zu den Konfigurationsparametern finden sie in der  **[Dokumentation der Konfigurationsoptionen](https://docs.ansible.com/ansible/latest/reference_appendices/config.html)** zu Ansible.

` $ vim ~/.ansible.cfg`

Im Ganzen ergibt sich dann hier die doch überschaubare Konfigurationsdatei zu Ansible.

` $ egrep -v '(^.*#|^$)' ~/.ansible.cfg` 

    [defaults]
    inventory          = ~/ansible/inventories/production/hosts
    interpreter_python = auto_silent
    [inventory]
    [privilege_escalation]
    [paramiko_connection]
    [ssh_connection]
    [persistent_connection]
    connect_timeout = 30
    [accelerate]
    [selinux]
    [colors]
    [diff]

#### Host-Definitionsdatei ####
Die Definition unseres Hosts mit den dazugehörigen Variablen beziehen wir später aus dem Ansible Playbook Archiv. Somit müssen wir hier nichts extra vorbereiten!

### Klonen des GIT-Repositories (Ansible-Playbook) ####
Nun klonen wir das Git-Reposotory direkt in unser zuvor angelegtes Verzeichnis **`~/ansible`**

`$ clone https://github.com/Django-BOfH/ffmuc-offloader_rpb4.git ~/ansible`

### Download des auf Debian Buster basierenden Raspbian ###
Im nächsten Arbeitsschritt holen wir uns nun das aktuelle **[Raspberry Pi OS](https://www.raspberrypi.org/software/operating-systems/)** (früher unter dem Namen _Raspbian_ bekannt) auf unseren Rechner. Dies hat mitunter auch noch den Charme, dass wir bei Bedarf alle normalen Anwendungen wie Webserver, Chatserver oder z.B. den Unifi-Controller einfach installieren können. 

Eine Anleitung zur manuellen Installation findet man auf der **[offiziellen Raspbian Seite](https://www.raspberrypi.org/documentation/installation/installing-images/README.md)**. 

` $ wget https://​downloads.raspberrypi.org/raspios_lite_arm64/images/raspios_lite_arm64-2022-01-28/2022-01-28-raspios-bullseye-arm64-lite.zip{,.sha256}`

Bevor wir nun das Archiv entpacken überprüfen wir noch die Integrität der heruntergeladenen Datei. Hierzu berechnen wir erst einmal die **SHA256**-Prüfsumme der Datei **raspbian_lite_latest**.

` $ sha256sum --check 2022-01-28-raspios-bullseye-arm64-lite.zip.sha256`
   
`2022-01-28-raspios-bullseye-arm64-lite.zip: OK`

Da der SHA256-Prüfsummencheck positiv erfolgreich war und mit einem **OK** bestätigt wurde, können wir nun das Archiv entpacken.

` $ unzip 2022-01-28-raspios-bullseye-arm64-lite.zip`

    Archive:  2022-01-28-raspios-bullseye-arm64-lite.zip
    inflating: 2022-01-28-raspios-bullseye-arm64-lite.img</code>

### Kopieren des Raspbian Images auf die microSD-Karte ###
Nun können wir das Image auf die MicroSD Karte, die wir später in den Raspberry 4B stecken kopieren. Wir werfen also am besten einmal einen Blick in das syslog unseres Arbeitsrechners und erkennen so das Device unserer Speicherkarte.

` # tail -f /var/log/messages` bzw. ` $ sudo tail -f /var/log/syslog`

    Sep  5 21:10:57 Djangos-ThinkPad-X230 kernel: [12795.867603] mmc0: new high speed SDHC card at address aaaa
    Sep  5 21:10:57 Djangos-ThinkPad-X230 kernel: [12795.868313] mmcblk0: mmc0:aaaa SC16G 14.8 GiB 
    Sep  5 21:10:57 Djangos-ThinkPad-X230 kernel: [12795.871017]  mmcblk0: p1 p2
    Sep  5 21:10:58 Djangos-ThinkPad-X230 kernel: [12796.199093] FAT-fs (mmcblk0p1): Volume was not properly unmounted. Some data may be corrupt. Please run fsck.
    Sep  5 21:10:58 Djangos-ThinkPad-X230 systemd[1]: Finished Clean the /media/django/boot mount point.
    Sep  5 21:10:58 Djangos-ThinkPad-X230 udisksd[976]: Mounted /dev/mmcblk0p1 at /media/django/boot on behalf of uid 1001
    Sep  5 21:10:58 Djangos-ThinkPad-X230 kernel: [12796.302402] EXT4-fs (mmcblk0p2): recovery complete
    Sep  5 21:10:58 Djangos-ThinkPad-X230 kernel: [12796.303545] EXT4-fs (mmcblk0p2): mounted filesystem with ordered data mode. Opts: (null)
    Sep  5 21:10:58 Djangos-ThinkPad-X230 systemd[1]: Finished Clean the /media/django/rootfs mount point.
    Sep  5 21:10:58 Djangos-ThinkPad-X230 udisksd[976]: Mounted /dev/mmcblk0p2 at /media/django/rootfs on behalf of uid 1001
    Sep  5 21:11:09 Djangos-ThinkPad-X230 gnome-terminal-[8119]: g_menu_insert_item: assertion 'G_IS_MENU_ITEM (item)' failed</code>

In dem gezeigtem Fall handelt es sich also um die Gerätedatei **`/dev/mmcblk0`**. Wir müssen also nun die zuvor heruntergeladene ***Debian Buster Image-Datei*** auf die MicroSD-Karte kopieren. In der Regel hat der "normale Nutzer" keine Rechte um diese Gerätedatei anzusprechen, wir müssen also als Benutzer **root** oder mir den Rechten des Benutzers **root** die  Gerätedatei **`/dev/mmcblk0`** in diesem Konfigurationsbeispiel ansprechen.

     # dd if=/home/django/Freifunk/2022-01-28-raspios-bullseye-arm64-lite.img of=/dev/mmcblk0 \
          bs=4M status=progress conv=fsync`

bzw. 

     $ sudo dd if=/home/django/Freifunk/2022-01-28-raspios-bullseye-arm64-lite.img of=/dev/mmcblk0 \
          bs=4M status=progress conv=fsync`

Da wir später weder Tastatur noch Monitor an unseren Raspberry 4B anstecken wollen, diesen demnach im **headless**-Mode betreiben wollen und werden, legen wir noch eine Datei **`/boot/ssh`** auf der SD-Karte ab. Nach dem erneuten Anstecken der MicroSD-Karte wir der Speicher in der Regel im Verzeichnis **`/run/media/`** oder auch **`/media/`** gemountet. Zum Anlagen der betreffenden Datei  **`ssh`** in dem Verzeichnis reicht also folgender Aufruf, bei dem wir den Usernamen natürlich unseren Gegebenheiten entsprechend anpassen.:

` # touch /run/media/django/boot/ssh`

bzw.

` $ touch /media/django/boot/ssh`

Anschliessend können wir nach einem unmounten des Gerätes **`/dev/mmcblk0`** die Micro-SD-Karte in den Kartenslot des Raspberry 4B stecken und den Kleinstcomputer mit dem Netzwerk sowie dem zugehörigen Netzteil verbinden und starten.

### Ändern des Default-Passwortes und kopieren des SSH-Public-Keys auf den Raspberry 4 ###
Der Benutzername lautet **pi** und das Passwort **raspberry**. Das Passwort dieses Nutzers ändern wir nun als erstes ab, da sonst die **Gefahr** besteht, dass Fremde sich unseres Offloaders bemächtigen!

Wir ändern also das Default-Passwort gleich mal ab und packen auch unseren SSH-Public-key auf den Raspberry 4B. Im nachfolgendem Beispiel gehen wir davon aus, dass der Raspberry 4B von unserem DHCP-Server automatisch eine IP-Adresse zugeteilt bekommen hat und zwar in diesem Konfigurationsbeispiel die **`192.168.0.23`** 

    $ ssh -l pi 192.168.0.25 -o IdentitiesOnly=yes "passwd" && \
          ssh-copy-id -i ~/.ssh/id_ed25519_ffmuc.pub -o IdentitiesOnly=yes pi@192.168.0.25

In dem folgenden Konfigurationsbeispiel vergeben wir für den Benutzer **`pi`** das Passwort **`gECzebzn7GYSLvXueECAxeGm7l7`**. Beim Ändern des bestehenden Passwortes müssen wir einmal das Default-Passwort **`raspberry`** eingeben und dann 2x das neue **`gECzebzn7GYSLvXueECAxeGm7l7`**. Beim Kopieren des Public-Keys müssen wir dann einmalig das neue geänderte Passwort **`gECzebzn7GYSLvXueECAxeGm7l7`** verwenden.

    The authenticity of host '10.0.10.29 (10.0.10.29)' can't be established.
    ECDSA key fingerprint is SHA256:vXhdvud24tyTUtOan4XeTZ7GzxZDQn9Rj0mxuhimkH4.
    Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
    Warning: Permanently added '10.0.10.29' (ECDSA) to the list of known hosts.
    
    pi@10.0.10.29's password: 
    
    Current password: raspberry
    
    New password: gECzebzn7GYSLvXueECAxeGm7l7
    Retype new password: gECzebzn7GYSLvXueECAxeGm7l7
    
    passwd: password updated successfully
    Changing password for pi.
    /usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/home/django/.ssh/id_ed25519_freifunk.pub"
    /usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
    /usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
    pi@10.0.10.29's password: 
    
    Number of key(s) added: 1

    Now try logging into the machine, with:   "ssh -o 'IdentitiesOnly=yes' 'pi@192.168.0.23'"
    and check to make sure that only the key(s) you wanted were added.


    $ ssh -o 'IdentitiesOnly=yes' 'pi@192.168.0.23' 

    Linux raspberrypi 5.4.51-v7l+ #1333 SMP Mon Aug 10 16:51:40 BST 2020 armv7l
    
    The programs included with the Debian GNU/Linux system are free software;
    the exact distribution terms for each program are described in the
    individual files in /usr/share/doc/STERNCHEN/copyright.
    
    Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
    permitted by applicable law.
    
    Wi-Fi is currently blocked by rfkill.
    Use raspi-config to set the country before use.
    
    pi@raspberrypi:~ $ 

### Vorbereitende Arbeiten vor dem Starten des Ansible-Playbooks ###
Beim Abarbeiten des ansible-playbook werden zur Konfiguration des Offloaders und dessen Komponenten/Dienste folgende Parameter benötigt:
* Batman-Release ([Version](https://downloads.open-mesh.org/batman/releases/)) der zum Einsatz kommen soll.
* [Segment/Domäne](https://ffmuc.net/wiki/doku.php?id=knb:gui#domaene) in dem der Offloader betrieben werden soll
* Angaben zur [Freifunk Karte](https://map.ffmuc.net/#!/de/map) zur Darstellung 
  * Hostname des Offloaders
  * Kontakt-Adresse des Node-Betreibers
  * Geographische Breitengrad des Raspberry Offloaders 
  * Geographische Längengrad des Raspberry Offloaders
* Funktionen, die der Raspberry Offloader noch ausführen soll:
  * Soll der Raspberry Offloader ein WLAN ausstrahlen (SSID leitet sich vom Segment-Namen ab)?
  * Soll der Raspberry Offloader ein Client-VLAN zur Verfügung stellen, wenn ja wie lautet die `VLAN-ID`?
  * Soll der Raspberry Offloader ein Mesh-VLAN zur Verfügung stellen, wenn ja wie lautet die `VLAN-ID`? 

Hier werden die zur Konfiguration benötigten Parameter nicht beim Aufruf des Playbooks abgefragt, sondern in zugehörigen **Inventory-Datei** hinterlegt. Das ist im ersten Schritt für den ungeübten Ansible-Nutzer zwar augenscheinlich aufwändiger, hat aber den Vorteil, dass man die zur Konfiguration benötigten Parameter immer sofort "zur Hand" hat und vor allem kann sofort bei Bedarf der Offloader neu gebaut oder bei Problemen im Betrieb nue versorgt werden.

Diese **Inventory-Datei** wurde mit Beispielswerten beim klonen des Repositorys mitgeliefert und findet sich im Pfad

**`~/ansible/inventories/production/host_vars/rpb4-ol-a/individual_host_specification`**

Hier tragen wir nun unsere individuellen Konfigurationsparameter ein, dazu bearbeiten wir die Host-spezifische Variablen-Datei mit dem Editopr unserer Wahl.

` vim ~/ansible/inventories/production/host_vars/rpb4-ol-a/individual_host_specification`

    # IP-Adresse unseres Raspberry in unserem eigenen lokalen Netzwerk
    # Alugehäuse ohne Lüfter und ohne Display
    # MAC: dc:a6:32:5c:46:07
    ansible_ssh_host: 192.168.0.23
    ansible_port: 22
    ansible_user: pi
    ansible_ssh_private_key_file: ~/.ssh/id_ed25519_freifunk
    #
    batman_adv_version:   "2022.0"
    ffmuc_segment:        "muc_ost"
    ffmuc_gateway:        "gw06"
    raspberry_hostname:   "ff_pliening_rpb4_ol_v6a"
    node_contact_address: "hier entlang => https://bit.ly/2VxGoXp"
    raspberry_latitude:   "48.198790640"
    raspberry_longitude:  "11.798020899"
    raspberry_wifi:       "ja"
    raspberry_clientvlan: "7"
    raspberry_meshvlan:   "8"

### Starten des Ansible-Playbooks ###
Nun können wir den bau unseres Offloaders mit Hilfe des Ansible Playbooks starten:

     $ ansible-playbook -i ~/ansible/inventories/production/hosts 
                           ~/ansible/wireguard-offloader.yml 
                           --limit rpb4-ol-a

Nach ca. 10 Minuten, abhängig von der zur Verfügung stehenden Bandbreite am Internet Anschluss, haben wir dann vollständig konfigurierten und funktionsfähigen Offloader für den Anschluss an das Netz von Freifunk München in Händen.

### Hilfe und Support ###
Bei Fragen rund um Freifunk München wenden man sich am besten an die loakle Community:
* https://ffmuc.net
* https://wiki.ffmuc.net
* https://chat.ffmuc.net/login

Der manuelle Konfigurationsablauf ist im **[WIKI von Freifunk München](https://ffmuc.net/wiki/doku.php?id=knb:rpb4_wg)** ausfürlich beschrieben.

Bei Fragen zum Ansible-Playbook findet man entweder in **[Djangos WIKI](https://dokuwiki.nausch.org/doku.php/centos:ansible:ffmuc-rpb4-ol)** ausfürliche Zusatzinformationen und zusätzliche Beschreibungen rund um das Ansible Playbook. Bugs/Anfragen entweder hier im GitHun oder per **eMail** an  <django@nausch.org>** melden.

### Danksagung ###
Als Basis für die Konfiguration setzen wir beim sehr gut dokumentierten Artikel von @awlnx aus dem **[Freifunk WIKI](https://ffmuc.net/wiki/doku.php?id=knb:rpb4_wg)** auf.
Nicht unerwähnt darf auch @lqb und meinem Freund **Klaus (der Spammer)** bleiben, der mit entscheidenden Tips zum Gelingen des Ansible-Playbooks, sowie meinen Kumpel **Oliver M.** der mich beim Thema Git ordendlich motiviertv hat, beigetragen hat! 

