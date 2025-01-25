---
title: "Von Legacy/MBT zu Uefi und Windows 11"
date: 2025-01-25
description:  "Nächster Halt, Windows 11"

---

# Von MBR, GPT, Bios und UEFI

Mir begegnen erstaunlicherweise immer noch viele Unternehmen mit Windows PCs im Legacy Mode. Damit wird in naher Zukunft Schluss sein. Mit dem umstieg auf Windows 11 werden die meisten dieser Clients das zeitliche segnen, denn spätestens hier muss über UEFI gebooted werden. 

Betroffen sind ältere oder günstigere Flotten an Geräten, die ihren regelmäßigen Austausch verpasst haben oder beim Umstieg auf zb. Windows 10 nie vollständig auf moderne Technologien wie UEFI aktualisiert wurden.

Bei der zwingenden Vorbereitung einer größeren Anzahl von Clients auf Windows 11, mit der Vorgabe im Idealfall ein InPlace Upgrade und keine Neuinstallation auf Windows 11 ohne Daten oder Konfigurationsverluste durchzuführen, wird es definitiv zu Problemen kommen. Die sauberste Migration ist eine Neuinstallation. Das steht fest. 

Sollen die Clients aber auf UEFI und damit GPT umgestellt werden muss man sich etwas einfallen lassen. Zum Glück gibt es dafür in Windows 10 Bordmittel. An anderer Stelle hätte ich jetzt gerne etwas zu SCCM oder den Gefahren dieses Vorgehens geschrieben aber das soll hier nicht das Thema sein.

# Booten über BIOS und UEFI 

Sucht man den Weg von Legacysystemen zu UEFI, dann kann man sich den Unterschied zwischen den Partionsstilen MBR und GPT ansehen.

MBR steht für Master Boot Record. Dieser ist in keiner eigenen Partition und befindet sich als meist 512 Byte großer Block am Anfang des Speichersystems im Sektor 0. Es handelt sich um zwei Teile, wobei der erste ein in Assembler geschriebenen Codeabschnitt zum Systemstart und der zweite eine alle Informationen über die vorhandenen Partitionen und Dateisysteme enthaltene Partitionstabelle ist.

Beim Boot über MBR startet das BIOS zuerst einen Selbsttest und prüft die Systemhardware. Im zweiten Schritt liest das BIOS den MBR Sektor 0 in den RAM und überträgt anschließend die Kontrolle über den Startvorgang an MBR, um das Betriebssystem zu starten.

Dieser Minibootloader ist sozusagen die Phase I des Bootvorgangs. Sind die Schritte oben durchgelaufen, wird der VBR (Volume Boot Record) in diesem Fall "Windows-Bootloader" auf der Primärpartition des Betriebssystems ausgeführt, womit Phase II beginnt.

Es gibt viele technische Gründe (zb. größere mögliche Festplattengröße) für den Umstieg von Legacy Bios zu UEFI. Konzentriert man sich auf den Bootvorgang, so
ist ein wichtiger Unterschied das hier in GPT formatierte Speichersystem. UEFI unterstützt die GUID-Partitionstabelle (GPT), die mehr Partitionen und größere Festplatten erlaubt als das traditionelle MBR. Der Bootvogang läuft hier direkt über eine eigene in FAT32 formatierte Bootpartition (EFI System Partition). Damit laufen die zwei Startphasen deutlich schneller ab. Zusätzlich kann diese Partition mehrere parallele Bootloader enthalten. UEFI lädt den VBR, also hier den Windows Bootmanager also direkt aus der EFI-Systempartition und gibt dann die Kontrolle an diesen weiter.

Um also von MBR zu GPR zu wechseln, benötigt man also einerseits eine Partitionskonvertierung in GPT und eine neue EFI-Systempartition, die den MBR Sektor ersetzt. Andererseits benötigt es noch den richtigen Bootloader als ESP Eintrag. Daher ist der Wechsel mit einer bloßen Partitionierung und Konvertierung nicht erledigt und der Umstieg per zb. Diskpart nicht trivial.

# Konvertierung mit MBR2GPT

Mit den neueren Windows 10 Versionen und Windows 11 gibt es für die Umstellung von MBR zu GPT daher das kostenlose Tool MBR2GPT von Microsoft:

MBR2GPT kann mit Glück ohne Datenverlust durchgeführt werden, aus Windows PE oder dem vollständigen Windows-System ausgeführt werden und BitLocker-Volumes können unterstützt werden (Ausschalten erforderlich).

MBR2GPT.EXE ist ein Werkzeug, das eine Festplatte vom Master Boot Record (MBR)-Format ins GUID Partition Table (GPT)-Format konvertiert, ohne dabei Daten auf der Platte zu löschen oder zu überschreiben.

* Es überprüft zunächst die Layout- und Geometrieeigenschaften der ausgewählten Festplatte, um sicherzustellen, dass sie die Anforderungen für eine Konvertierung erfüllt. Dabei prüft es unter anderem auf bestimmte Sektorenanordnungen am Anfang und Ende der Platte.

* Wichtig! Es wird freier Speicher auf der Platte am Anfang und Ende benötigt! Verkleinere also die Datenpartition am Ende der Platte, falls vorhanden!

* Anschließend wandelt MBR2GPT die Partitionstypen um

* Für das Booten in UEFI-Modus wird die EFI-Systempartition erstellt. Die bestehenden Partitionen werden in das neue GPT-Format umgewandelt, wobei neue GUIDs für die Partitionen generiert werden.

* Schließlich werden automatisiert neue Bootdateien installiert, um das System zum Starten in UEFI-Modus zu ermöglichen.

# Nach der Konvertierung

Nach der erfolgreichen Umstellung auf GPT kann der Versuch unternommen werden die Festplatte aus der alten Hardware in ein neues Gerät, dass zum Beispiel TPM 2.0 unterstützt, umzuziehen ohne das System neu aufsetzen zu müssen.

Aber auch hier kann es bei größeren Clientzahlen erfahrungsgemäß zu massiven Problemen kommen. Zuerst müssen die passenden Gerätetreiber ergänzt werden und nach Firmwareupgrades funktionierten bei einer Reihe an Systemen, die von HP zu Dell zogen auf einmal keine Remotedesktopverbindungen in Windows mehr. Dies wurde durch die Aktivierung der Uefi-Option "Intel Trusted Execution Technology" wieder möglich. Das nächste Stichwort wären vermutlich Probleme mit dem Chipsatz. Um einen anschließenden Langzeittest wird man hier also nicht herumkommen.

# Fazit

Die Umstellung auf UEFI/GPT ist lange überfällig und sollte in den meisten Unternehmen schon mit dem Umstieg zu Windows 10 erfolgt sein. Der gründliche Weg führt daher größtenteils über eine Formatierung und Neuinstallation der vorliegenden Systemfestplatten oder einem Komplettaustausch. Es gibt jedoch Möglichkeiten ohne Datenverluste und Neuinstallation umzuziehen. Auch diese nicht ganz ohne Risiko und Komplikationen. Dies ist aber gerade dort essenziell, wo lückenhafte Dokumentation über den Konfigurationszustand der vorliegenden Clients vorliegt und gleichzeitig Zeitdruck besteht.