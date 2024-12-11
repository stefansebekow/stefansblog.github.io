---
title: "Das elfte Gebot des NAS"
date: 2024-12-12
description:  "Das elfte Gebot des NAS: Du sollst kein 100 TB großes Volume in einem RAID 5 Array fahren!"

---


Dinge gibt es, die gibt es nicht. Leider scheint es aber aus meinen aktuellen Erfahrungen so, als müsste man das trotzdem einmal für die Nachwelt hier festhalten. Wer auch immer das liest und in Zeiten von günstigem Cloudstorage, aus "Gründen" trotzdem gezwungen ist große Datenmengen lokal auf eigenen NAS-Systemen zu sichern. Bitte! Spare nicht und plane Redundanz ein!  


# Langsamer Rebuild nach Hardwarefehlern

RAID 5 bietet zwar eine gute Balance zwischen Datensicherheit und gewonnener Speicherkapazität. Jedoch kann die Rekonstruierung des Systems nach Hardwarefehlern bei sehr großen Volumes extrem zeitaufwändig sein. Bei 100 TB Daten würde der Rebuild-Prozess Tage oder sogar eher Wochen dauern, abhängig von der Leistung deiner Festplatten und dem verwendetem NAS. Solange das System im Degradedmodus läuft, ist jedes Arbeiten mit den vorhandenen Daten extrem eingeschränkt.  


# Das eigentliche Hauptrisiko 

Die Wiederherstellungszeit könnte so lang sein, dass du riskierst, dass weitere Festplatten ausfallen. Das ist nicht mal so weit her, da diese vielleicht zeitgleich beschafft wurden. Damit ist der nächste Ausfall vielleicht nicht soweit wie erhofft. Ein Ausfall einer Festplatte kann bei Raid5 Arrays ohne Datenverlust überbrückt werden. Bei der zweiten ist meist Schluss.  


# Schlechte Performance 

Mit zunehmender Größe des RAID-Volumes nimmt die Lesegeschwindigkeit rapide ab. 
Langsame Lesezeiten können dann störend sein, wenn häufig auf große Datenmengen und Sammlungen vieler einzelner, kleiner Dateien zugegriffen wird.



# Wie stattdessen? 

Anstatt sich auf ein einzelnes riesiges Volume zu konzentrieren, sollten wir über folgende Alternativen nachdenken!

• Aufteilen des Datensatzes in kleinere, unabhängige Volumes für bessere Performance und Wartbarkeit. Aufteilung in kritische und unkritische Volumes. 

• Implementierung einer Backup-Lösung für kritische Daten.

• Benutze Raid 6 Arrays 

• Nutzung von Cloud-Speicherdiensten für Archivierung und Backup-Zwecke.


# Das Fazit

Während es natürlich technisch möglich ist, ein 100 TB großes Volume mit RAID 5 auf einem NAS zu implementieren, ist es reines Glücksspiel, dass es hier nicht einmal zu einem kompletten Datenverlust kommt. Das Argument, "dass eine Hotspare verbaut sei" genügt in diesem Fall nicht. Während des ewig langen Rebuild-Prozesses wirst du mit diesem Wissen und falls es kein gutes Backup gibt, Nachts kein Auge zu bekommen!  



