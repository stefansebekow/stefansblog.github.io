---
title: "Wake on LAN Troubleshooting: Dell Deep Sleep Control als Ursache"
date: 2026-04-26
draft: false
---

Letzte Woche habe ich einen Kollegen bei einer Fehlersuche unterstützt. Wir rätselten beide über ein eigentlich banales Problem. Zahlreiche der neu gelieferten Dell Rechner ließen sich plötzlich nicht mehr nach dem Standardsetup und unserem üblichen Dell/Uefi Preset per Wake on LAN starten. 

## Wake on LAN und das Magic Packet

Bei Wake on Lan ist das entscheidende Signal das sogenannte **Magic Packet**. Dabei handelt es sich um ein speziell aufgebautes Datenpaket auf Byte Ebene, das per Broadcast verschickt wird. Das Magic Packet beginnt mit sechs Bytes **FF FF FF FF FF FF** als Broadcast MAC Adresse. Das führt dazu, dass das Paket im lokalen Netzwerk an alle MAC Adressen und damit Geräte im selben Segment verteilt wird. Dadurch erreicht es auch Systeme, die sich im ausgeschalteten Zustand befinden, solange deren Netzwerkkarte noch mit Strom versorgt ist.

Direkt danach folgt die MAC Adresse der Zielnetzwerkkarte, die sechzehnmal hintereinander wiederholt im Datenbereich enthalten ist. Die Netzwerkkarte prüft diesen Datenstrom und erkennt so eindeutig, ob das Paket für sie bestimmt ist. Wird die eigene MAC Adresse in dieser Struktur gefunden, löst sie das **Wake Signal** aus und der Rechner startet. 

Das Paket selbst wird in ein normales Ethernet Frame eingebettet. Im IP Header steht als Zieladresse die Broadcast Adresse, im Ethernet Header wird ebenfalls die Broadcast MAC genutzt. Dadurch nimmt jeder Switch im lokalen Netzwerk das Paket entgegen und leitet es an alle Ports im selben VLAN oder Segment weiter.

## Fehlerbild

Das Tückische in unserem Fall war, dass die Netzwerkkarte augenscheinlich aktiv war. Sie wurde weiterhin mit Strom versorgt und die LEDs am Anschluss haben geblinkt. Es fand jedoch keine Reaktion auf die Broadcastkommunikation seitens des Interfaces statt. 

Ohne viel Hoffnung auf Erfolg bin ich noch einmal die Power Einstellungen im Bereich der Energieeinstellungen durchgegangen und habe dabei die entscheidende Option gefunden.

Diese nennt sich bei Dell **Deep Sleep Control** und sorgt dafür, dass bestimmte Energiesparvorgaben eingehalten werden, unter anderem ein sehr niedriger Verbrauch im Zustand S5, also wenn das System ausgeschaltet ist.

Diese Einstellung steuert, wie stark das System im ausgeschalteten Zustand Energie spart. Konkret geht es um die Zustände **S4 und S5**, also Ruhezustand und Herunterfahren. Wird bei Dell Firmware Deep Sleep Control aktiviert, versetzt sich das System in einen besonders sparsamen Modus (<1W Verbrauch). Dabei werden fast alle stromverbrauchenden Komponenten abgeschaltet, um extrem niedrige Verbrauchswerte zu erreichen. Genau hier liegt das Problem.

Für Wake on LAN muss die Netzwerkkarte weiterhin minimal mit Strom versorgt werden und auf eingehende Signale hören können. Das Magic Packet wird vom Netzwerk empfangen und löst dann das Einschalten des Systems aus. Im Deep Sleep Modus wird jedoch genau diese Funktion unterbunden. Die Stromversorgung für Teile der Netzwerkschnittstelle wird reduziert oder ganz deaktiviert und auch interne Signale wie **Power Management Events** werden abgeschaltet.

Sobald Deep Sleep Control deaktiviert wurde, blieb die Netzwerkkarte im ausgeschalteten Zustand wieder ausreichend aktiv. Sie konnte das Magic Packet empfangen und den Startvorgang wie vorgesehen auslösen. 

Die Erkenntnis daraus ist einfach: Wenn Wake on LAN nicht funktioniert, obwohl die Konfiguration korrekt erscheint, lohnt sich ein Blick in die Energieeinstellungen auf UEFI oder Firmware Ebene. Stromsparfunktionen können Netzwerkfunktionen vollständig unterbinden, ohne dass es auf den ersten Blick sichtbar ist.

## Weitere Infos

Weitere Informationen zu DeepSleepControl und allen Uefi-Firmware Settings unter modernen Dell Desktopsystemen finden sich im [**Dell Support Manual**](https://www.dell.com/support/manuals/de-de/dell-cmnd-config-v3.1/dcc_cli/-deepsleepctrl)  
