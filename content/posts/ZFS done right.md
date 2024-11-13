---
title: "ZFS done right!"
date: 2024-03-15
---

# Worauf man bei jedem "zpool create" achten sollte !

Ich habe beim Austausch einer defekten SSD im System-ZRaid Pool eines Proxmox Hosts einen erst trivial wirkenden Ratschlag von einem erfahrenen Kollegen bekommen, den ich euch nicht vorenthalten möchte!

Wer bereits einmal einen ZFS Pool eingerichtet hat, kennt folgende Zeilen:

```
zpool create zfspool mirror DEVICE1
DEVICE2 mirror DEVICE3
DEVICE 4 -o ashift=12 -m /mnt/zfspool
```
Ich will hier auf die extreme Wichtigkeit hinweisen die Option **-o ashift=12** mitzunehmen, wenn man kann. Das kann bei einer block size von 4k und aufwärts zu einer beachtlichen Perfomancesteigerung führen.  

#  Was sind die Vorteile des Parameters ashift=12 im Kontext von ZFS-Pools ?

Der Parameter *ashift* steht für "alignment shift" und wird verwendet, um die Größe der Blöcke zu konfigurieren, die beim Schreiben auf das physische Speichermedium verwendet. Die Standardgröße beträgt typischerweise 512 Byte, aber moderne SSDs verwenden oft 4-K-Sektoren. Achte nur darauf, dass du Festplatten verwendest, die dem **Advanced Format* (Kennung meist AF) entsprechen und damit eine Blockgröße von über 528 bytes fahren. Meist sind das 4096, 4112, 4160, or 4224-byte Sektoren. 

Zu den Vorteilen gehören: 

1. Optimierte Leistung --> ZFS kann effizienter schreiben und lesen, da es die nativen Blöcke der SSD berücksichtigt. Dies führt zu einer verbesserten Lesegeschwindigkeit und reduziert den I/O-Stress auf dem Speichermedium!

2. Effiziente Ressourcennutzung: Mit **ashift=12** nutzt das ZFS die gesamte Kapazität des Speichersystems aus, ohne unnötige Leerlaufraum zu verwalten. Das bedeutet mehr Nettodaten!

3. Verbesserter Datentransfer: Höhere Geschwindigkeiten beim Schreiben und Lesen von Daten!

# Achtung, noch ein Hinweis!

Das ganze gilt nicht bei SSD's mit 8K Sektoren. Für SSDs mit 8K Sektoren sollte **"ashift=13"** verwendet werden- Die Hauptvorteile von ashift=13 sind aber eine efizientere Nutzung der Speicherplatzkapazität und eine geringere Write Amplification für kleine Datenblöcke (< 8K). 

Das Gemeine an dem Thema ist, dass viele SSDs sich selbst als 512-Bit-Geräte ausgeben, obwohl sie tatsächlich 4K oder sogar 8K Sektorgrößen haben. Deshalb ist es vor allem ratsam, ashift explizit beim Erstellen eines Pools anzugeben, wenn bekannt ist was für Platten (auch in Zukunft) verwendet werden!


## Zusammenfassung

Es gibt also keinen großen Nachteil darin, ashift zu hochzusetzen, wenn man die verwendeten Festplatten kennt und sie heutigen Standards entsprechen. Ein zu niedriger Wert kann dagegen aber zu Leistungsproblemen führen! Die Verwendung von manuellen ashift-Optionen bietet bei modernen Festplatten fast immer einen optimalen Kompromiss zwischen Leistungsoptimierung und Kapazitätsnutzung. Es ist vorteilhaft für SSD-basierte Systeme und ermöglicht eine flexible Anpassung an verschiedene Speichermedien.