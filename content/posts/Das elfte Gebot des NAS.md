---
title: "Das elfte Gebot des NAS"
date: 2024-12-12
description:  "Du sollst kein 100 TB großes Volume in einem RAID 5 Array fahren!"

---


Dinge gibt es, die gibt es nicht. Leider scheint es aus aktuellem Anlass aber so, als müsste man das trotzdem einmal für die Nachwelt festhalten. Wer auch immer in Zeiten von günstigem Cloudstorage, aus anderen Gründen gezwungen ist große Datenmengen auf eigenen NAS-Systemen zu sichern. Bitte! Spare nicht, wähle Weise beim Raid Level und plane Hardwareredundanz ein! Hier sind meine Argumente:


# 1. Langsamer Rebuild nach Hardwarefehlern und drohender Datenverlust

RAID 5 bietet eine gute Balance zwischen Datensicherheit und gewonnener Speicherkapazität. Die Rekonstruierung des Systems kann nach Hardwarefehlern bei sehr großen Volumen jedoch extrem zeitaufwändig sein. Ab +100 TB Nettospeicher würde der Rebuild-Prozess möglicherweise Wochen dauern, abhängig von der Leistung deiner Festplatten und der verwendeten Hardware. Solange das System im "Degraded" Modus läuft, ist jedes Arbeiten mit den vorhandenen Daten, dass schnelle Lese- und Schreibzugriffe benötigt extrem eingeschränkt.  

Die Wiederherstellungszeit könnte so lang sein, dass du den Ausfall weiterer Festplatten riskierst. Das ist nicht mal so weit her, da diese vielleicht zeitgleich beschafft wurden. Ein Ausfall einer Festplatte kann bei Raid5 Arrays ohne Datenverlust überbrückt werden. Bei der zweiten wird es kritisch.


# 2. (Echt) Schlechte Performance 

Mit zunehmender Größe des RAID-Volumes nimmt die Lesegeschwindigkeit rapide ab. 
Langsame Lesezeiten können verherend sein, wenn häufig auf große Datenmengen und Sammlungen vieler einzelner, kleiner Dateien zugegriffen wird. Das kann der Fall sein, wenn das Volumen zb. als ICSI Laufwerk in einer virtuellen Maschine genutzt wird. 



# 3. Mögliche Alternativen beim NAS Setup 

Anstatt sich auf ein einzelnes riesiges Volume zu konzentrieren, sollten wir also über folgende Alternativen nachdenken!

* Aufteilen des Datensatzes in kleinere, unabhängige Volumes für bessere Performance und Wartbarkeit. Aufteilung in kritische und unkritische Volumes. 

* Implementierung einer Backup-Lösung für kritische Daten und/oder zumindest einer Spiegelung auf ein Zweitgerät

* Nutzung von Raid 6 Arrays für eine höhere Ausfallsicherheit 

* Nutzung von Cloud-Speicherdiensten für Archivierung und Backup in Form einer Drittsicherung.


# Zusammenfassung

Während es natürlich technisch möglich ist, ein +100 TB großes RAID 5 Volume auf einem NAS zu fahren, ist es reines Glücksspiel, dass es hier nicht einmal zu einem kompletten Datenverlust kommt. Das Argument, "dass eine Hotspare verbaut sei" genügt in diesem Fall nicht. Während des ewig langen Rebuild-Prozesses wirst du mit diesem Wissen und falls es kein gutes Backup gibt, Nachts kein Auge zu bekommen!  
