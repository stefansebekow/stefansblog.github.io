---
title: "best practice: Erste Schritte nach dem Aufsetzen eines Servers unter Cent OS"
date: 2022-10-16
description: Erhöhe die Sicherheit auf deinem CentOS 8 Server und richte dir als allererstes einen Alltagsnutzer ein!

---

# Einleitung

Wenn du das erste Mal einen CentOS 8 Server aufsetzt, gibt es einige Einstellungen, die du als Grundlagenkonfiguration übernehmen kannst und dieser Beitrag richtet sich an dich.  Ich habe mir diese im Zuge meiner LPIC-1 Vorbereitung ein paar anbsolut grundlegende Schrite zusammenrecherchiert und trage dieser hier zusammen. Mit folgender Anleitung erhöhst du die Sicherheit und Nutzbarkeit deines CentOS Servers und erhältst eine gute Basis für dein individuelles Setup. Du solltest hierfür bereits wissen, wie du dich per SSH auf eine virtuelle Maschine verbindest. 

# Schritt 1 — I am (g)root: Logge dich als Root ein

Um dich beispielsweise per SSH in deinem Server einzuloggen wirst du seine öffentliche IP-Adresse, das Passwort und gegebenenfalls den jeweiligen, privaten SSH-Schlüssel benötigen, um dich als Root-User zu authentifizieren. Wie und mit welchen Werkzeugen du dich per "Secure Shell Script" mit einem virtualisiertem Server verbinden kannst, zeige ich dir in einer anderen Anleitung.

Um dich also von einem anderen System als Root anzumelden, öffne die Konsole und ersetze die Adresse mit der öffentlichen IP-Adresse deines CentOS-Servers:
```
    ssh root@192.168.1.125
```
Da dir die Maschine bekannt ist, bestätige die anschließenden Warnhinweise und wähle dich entweder mit deinem Passwort oder mit deinem passwortgesicherten SSH-Schlüssel ein. Bei der ersten Anmeldung als "root" wirst du aufgefordert dein Root-Passwort anschließend zu ändern.

# Was ist ein Root User?

Der Root User ist die administrative Funktion in jedem Linux System, die stets und ohne Einschränkungen über die weitesten Berechtigungen verfügt. Wer den Zugang zu dieser Rolle besitzt, besitzt letztendlich das System. Aufgrund dieser dauerhaft erhöhten Rechte ist davon abzuraten regelmäßig als Root-User zu agieren! In dieser Rolle sind destruktive Änderungen möglich, die nicht rückgängig zu machen sind. 

Im nächsten Schritt setzen wir daher erst einmal einen zweiten Useraccount auf, mit dem wir im Alltag arbeiten können. Keine Sorge! Auch dieser User ist natürlich in Lage sein sich temporär Schreibrechte zu geben.

# Schritt 2 — Wir erstellen ein Benutzerkonto

Sobald du als Root angemeldet bist, kannst du das neue Benutzerkonto erstellen, mit dem wir uns von nun an anmelden werden.

In diesem Beispiel wird ein neuer Benutzer namens "Paul" erstellt. Sie sollten ihn jedoch durch einen beliebigen Benutzernamen ersetzen:
```
    adduser Paul
```
Nächster Schritt, ein starkes Kennwort für den User setzen:
```
    passwd Paul
```
Es folgt die Aufforderung, das Passwort zweimal einzugeben. Danach ist der Benutzer Paul einsatzbereit, aber zunächst erteilen wir diesem Benutzer zusätzliche Berechtigungen zur Verwendung des sudo-Befehls. Dadurch können wir bei Bedarf Befehle als Root ausführen. Das Erstellen von Nutzerkonten wirkt zwar trivial aber es ist wichtig nicht statt des Kommandos adduser - useradd zu verwenden. Das zu erläutern sprengt hier den Umfang und du findest das schnell durch ausprobieren selbst raus.  

# Schritt 3 – Gewährung von Administratorrechten

Jetzt haben wir ein neues Benutzerkonto mit regulären Kontoberechtigungen. Manchmal müssen wir jedoch möglicherweise Adminaufgaben erledigen.

Um zu vermeiden, dass wir uns von unserem normalen Benutzer abmelden und wieder als Root-Konto anmelden müssen, können wir für unser normales Konto sogenannte „Superuser“- oder Root-Rechte einrichten. Dadurch kann unser normaler Benutzer Befehle mit Administratorrechten ausführen, indem er vor jedem Befehl das Wort „sudo“ einfügt.

Um diese Berechtigungen unserem neuen Benutzer hinzuzufügen, müssen wir den neuen Benutzer zur sogenannten "Wheel" Gruppe hinzufügen. Standardmäßig dürfen Benutzer, die zur Wheel-Gruppe gehören, unter CentOS 8 den Befehl sudo verwenden.

Führe als Root diesen Befehl aus, um den Nutzer Paul zur Wheel-Gruppe hinzuzufügen:

```
    usermod -aG wheel Paul
```

# Schritt 4 — Wir richten eine einfache Firewall ein

Firewalls bieten ein grundlegendes Maß an Sicherheit für unseren Server und sind unabhängig vom Verwendungszweck fast obligatorisch. Wie werden, den Datenverkehr zu jedem Port auf unserem Server verweigern und nach und nach per Ausnahmen einzelne Ports freigeben. CentOS verfügt über einen Service namens firewalld, um diese Funktion auszuführen. Ein Tool namens firewall-cmd wird zum Konfigurieren der Firewall-Richtlinien von firewalld verwendet.


Wir installieren und konfigurieren also firewalld:
```
    dnf install firewalld -y
```
Die Standardkonfiguration von firewalld erlaubt schlauerweise SSH-Verbindungen per Port 22, sodass wir die Firewall sofort einschalten können:
```
    systemctl start firewalld
```
Wir prüfen, dass der Dienst wirklich läuft:
```
    systemctl status firewalld
```
```
Output
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2020-02-06 16:39:40 UTC; 3s ago
     Docs: man:firewalld(1)
 Main PID: 13180 (firewalld)
    Tasks: 2 (limit: 5059)
   Memory: 22.4M
   CGroup: /system.slice/firewalld.service
           └─13180 /usr/libexec/platform-python -s /usr/sbin/firewalld --nofork --nopid
```
Wie wir sehen können ist firewalld sowohl "enabled" als auch "active", was bedeutet, dass standardmäßig gestartet wird, wenn das System neu gestartet wird.
Nachdem der Dienst ja nun eingerichtet ist, können wir „firewall-cmd“ verwenden, um Regeln für die Firewall abzurufen und festzulegen.

Als Erstes schauen wir wie der aktuelle Stand ist:
```
    firewall-cmd --permanent --list-all
```
```
Output
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: eth0 eth1
  sources:
  services: cockpit dhcpv6-client ssh
  ports:
  protocols:
  masquerade: no
  forward-ports:
  source-ports:
  icmp-blocks:
  rich rules:
```
Hilfreich finde ich mir jetzt erst einmal eine Liste klassischer Dienste, die sich per Namen aktivieren lassen, ausgeben zu lassen!
Also:
```
    firewall-cmd --get-services
```
Hinzufügen können wir dann mit der "--add-service" flag:
```
    firewall-cmd --permanent --add-service=http
```
Das erlaubt uns jetzt zum Beispiel http traffic und eingehende Anfragen über Port 80. Konfigurationsänderungen kommen aber immer erst zum Tragen, wenn der Dienst neu gestartet wurde:
```
    firewall-cmd --reload
```

# Schritt 5 — Wir richten Paul einen externen Zugriff ein 

Da wir nun einen regulären Nicht-Root-Benutzer für den täglichen Gebrauch haben, müssen wir sicherstellen, dass wir ihn für die SSH-Verbindung zu unserem Server verwenden können.


Der Prozess zum Konfigurieren des SSH-Zugriffs für  Paul hängt davon ab, ob das Root-Konto des Servers ein Passwort oder SSH-Schlüssel zur Authentifizierung verwendet.
Wenn das Root-Konto die Passwortauthentifizierung verwendet und wir uns mit einem Passwort mit dem Root-Konto angemeldet haben, ist die Passwortauthentifizierung für SSH ebenfalls aktiviert. Wir können das einmal mit Paul testen: 
```
ssh Paul@your_server_ip
```
Nehmen wir an es funktioniert. Ab jetzt wird Paul das Sudo Kommando benötigen:
```
    sudo command_to_run
```

Als erstes erstellt sich Paul auf einem lokalen und vertrauenswürdigem System ein Schlüsselpaar

```
ssh-keygen -t rsa -b 4096 -C "paul@beispiel.de"
```
Dieser Befehl erstellt ein RSA-Schlüsselpaar mit 4096 Bit Länge. Sie werden aufgefordert, einen Speicherort für den privaten Schlüssel einzugeben. Standardmäßig wird er in auf Linuxsystemen in ~/.ssh/id_rsa gespeichert.

```
cat ~/.ssh/id_rsa.pub
```
Das CAT Kommando oder auch "concatenation" zeigt den Inhalt von Dateien, wie Pauls öffentlichen Schlüssel an. Wir kopieren den Inhalt.


Zu guter Letzt: Hinzufügen des öffentlichen Schlüssels zum Server. Also noch einmal eine SSH Verbindung zum Server aufbauen...

```
ssh paul@server_ip 
mkdir -p ~/.ssh && echo 'paul'@'server_ip' 'Der öffentliche Schlüssel' >> ~/.ssh/authorized_keys"
```
```
chmod -r 600 ~/.ssh
```

Optional können wir uns nun einmal abmelden und den Schlüssel testen! 
Hat alles funktioniert, deaktivieren wir die Passwortanmeldung: 

'''
sed -i '/PasswordAuthentication yes/d' /etc/ssh/sshd_config &&
systemctl restart sshd
'''
Zusammenfassung

Mit diesen Schritten haben wir einen SSH-Schlüssel erstellt und auf unserem Server hinterlegt. 
Paul kann sich nun ohne Passwort anmelden (eine wahre win-win Situation ,), indem folgender Befehl verwendet wird:
``` 
ssh paul@server_ip
```
Wir merken uns: Ein Passwort-Login ist im Falle von SSH nur als Fallback zu gebrauchen. Mit einem Passwort gesicherten SSH-Key erhöhen wir 
die Sicherheit vor unbefugtem Zugriff. Nun wo wir diese Grundlagen können, wird es einfach diesen Prozess zu "ansibilisieren". Dazu kommen wir ein anderes Mal. 
