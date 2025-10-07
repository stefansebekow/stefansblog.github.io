---
title: "Hyperkonvergente Infrastruktur mit Proxmox und Ceph.md"
date: 2025-10-07
description:  ""

---
Viele Unternehmen stehen aktuell vor der Herausforderung, dass VMware durch die Änderungen der Lizenz- und Preisgestaltungkleinere kleinere Vorhaben stark belastet. Dauerlizenzen verschwinden, die Preisaufschläge sind massiv und Supportkosten steigen. Deshalb wird Proxmox als Alternative immer interessanter. Ich setze privat und auch im Berufsfeld schon seit längerer Zeit Proxmox für interne Dienste ein. Mit den Änderungen um Proxmox 9 wird dieser Hypervisor jetzt sowohl für alte Infrastruktur, als auch neue ein guter Nachfolger. [ProxmoxVE]([https://www.proxmox.com]) ist eine Open-Source-Plattform für Virtualisierung, die Container, VMs, Netzwerk und Storage unter einer Oberfläche vereint. Im Zusammenspiel mit [CEPH](https://ceph.io/en/discover/) entsteht daraus eine hyperkonvergente Infrastruktur. Das bedeutet: Rechenleistung und Speicher werden nicht getrennt organisiert, sondern auf denselben Knoten bereitgestellt. Jeder Host ist gleichzeitig Compute-Node und Storage-Node. 

# Proxmox VE 9 hat jetzt ISCSI+FC Snapshot-Support für klassische Shared-Storage-Systeme

Fangen wir aber erst einmal bei der alten Welt an. Mit Proxmox  9 gibt es neue Möglichkeiten, [Snapshots auch auf gemeinsam genutztem SAN Storage](https://forum.proxmox.com/threads/inside-proxmox-ve-9-san-snapshot-support.169675/) der alten Welt zu nutzen, z. B. bei LUNs, die über iSCSI oder Fibre Channel bereitgestellt werden. Speicher, der bislang nicht zuverlässig Snapshots unterstützte, kann jetzt mit sogenannten „thick‑provisioned LVM Storage mit Snapshot als Volume Chains“ konfiguriert werden. Das heißt: Ein Disk‑Snapshot persistiert den aktuellen Zustand und der neue Datenstrom wird auf einem neuen Volume geführt. Das macht MSA‑ oder SAN‑LUN‑Speicher deutlich flexibler nutzbar. Snapshots waren früher oft ein Grund, bei bekannten Hypervisoren zu bleiben. 

# Proxmox VE und Ceph: Virtualisierung und Storage in einem?

[Ceph](https://ceph.io/en/discover/) ist ein verteiltes, selbstheilendes und skalierbares Speichersystem. Es speichert Daten als Objekte über mehrere physische Datenträger hinweg und kann bei Ausfällen einzelner Komponenten selbstständig wieder vollständige Redundanz herstellen. Ceph wird direkt über die Proxmox-Oberfläche konfiguriert und verwaltet, ohne zusätzliche Software oder Gateways. 

In einem typischen Proxmox-Ceph-Setup besteht der Cluster aus mindestens drei physischen Hosts. Jeder Host bringt eigene SSDs oder HDDs mit, aus denen Ceph sogenannte *OSDs* macht. Diese OSDs werden zu Storage-Pools zusammengefasst. Proxmox nutzt diese Pools als Speicherziel für VM-Disks. Die Daten einer virtuellen Maschine liegen also nie nur auf einem Host, sondern werden redundant über alle Nodes verteilt gespeichert. Fällt ein Host aus, bleiben die Daten erhalten, und die VM kann auf einem anderen Node weiterlaufen. 

Das Ergebnis ist ein Cluster, bei dem keine externe Storage-Appliance notwendig ist. Statt eine teure und oft proprietäre Storage-Lösung in die Umgebung einzuhängen, bringt jeder Server seine eigene Storage-Kapazität mit. Ceph sorgt im Hintergrund für Konsistenz und Redundanz.

# Vorteile gegenüber klassischen Ansätzen

In traditionellen Virtualisierungs-Setups wird Rechenleistung zentral organisiert und auf Hosts verteilt. Der Storage hingegen liegt auf separaten Appliances. Diese sind oft teuer, wartungsintensiv und bilden einen zentralen Fehlerpunkt. Hyperkonvergentes Ceph löst diesen Bruch auf. Es braucht keine externen Controller, keine RAID-Hardware, kein dediziertes Storagemanagement Die gesamte Umgebung wird über das Proxmox-Frontend administriert, inklusive Storage. Updates, Monitoring und Konfiguration laufen einheitlich. Der Verzicht auf Hardware-RAID ist Absicht. Ceph funktioniert besser mit direktem Zugriff auf die Rohplatten. Redundanz wird durch Replikation auf Objektebene erreicht, nicht über klassische RAID-Mechanismen. Das vereinfacht das Setup, reduziert die Fehleranfälligkeit und sorgt für klarere Zuständigkeiten.

Ein Vorteil gegenüber vielen klassischen Systemen ist auch die Möglichkeit, verschiedene Festplattentypen zu kombinieren. HDDs für Kapazität, SSDs für reine Hochleistungspools. Alles lässt sich mischen, ohne dass das System instabil werden muss. Auch inhomogene Knoten mit unterschiedlicher Ausstattung sind technisch möglich.

# Zu schön um wahr zu sein? Vorsicht bei Skalierung und Betrieb!

Gleichzeitig ist Ceph kein einfaches Out of the Box SAN-Replacement. Wer die Komplexität falsch einschätzt, bekommt keinen stabilen Cluster. Die Konfiguration hat viele Fallstricke. Netzwerkdesign, OSD-Verwaltung, CRUSH-Map, Quorum und Monitoring brauchen Erfahrung. Bei Setups über mehrere Standorte hinweg stößt das System schnell an Grenzen! Ceph ist nicht für hohe Latenzen oder instabile Links ausgelegt. Kommunikation zwischen den Nodes muss dauerhaft stabil und performant sein, sonst ist der Betrieb beeinträchtigt oder gar nicht mehr möglich.

Ein Cluster lässt sich horizontal erweitern. Neue Hosts werden hinzugefügt, OSDs werden aufgenommen, und Ceph verteilt die Daten automatisch neu (Rebalancing). Dabei bleibt die Plattform live. Weder VM-Betrieb noch Datenverfügbarkeit müssen unterbrochen werden. Die Grundlage für Betrieb und Verfügbarkeit ist das *Quorum*. Ceph basiert wie Proxmox auf einem klassischen Scale-out-Ansatz mit verteiltem Konsens. Damit ein Cluster korrekt arbeiten kann, muss mehr als die Hälfte der Monitore aktiv und erreichbar sein. Die Faustregel lautet: *Bei "n" Knoten benötigen Sie mehr als "n/2" Knoten*.

Bei ungeraden Zahlen wir dementsprechend aufgerundet. Das ist die minimale Zahl an Monitoren (MONs), die für das CEPH Quorum notwendig sind. Bei drei Nodes braucht es also immer mindestens zwei aktive MONs.
Für den produktiven Betrieb ist auch ein dediziertes Netzwerk für den Ceph-Traffic notwendig. Bei höherer Last oder größeren Clustern empfiehlt sich der Einsatz von zwei getrennten Netzen, eines für Public Traffic, eines für den internen Cluster-Traffic. Der Datendurchsatz ist entscheidend: Ceph profitiert stark von hoher Bandbreite: 25GbE oder 40GbE sollten bei größeren Setups Standard sein.

Hardwareseitig rechnet man mit ca. 1 GB RAM pro 1 TB brutto Storage, abhängig vom Pool-Typ, Replikationsgrad und eingesetzter Hardware. Ceph erlaubt das Mischen verschiedener Datenträgertypen – SATA-HDDs für Kapazität, NVMe-SSDs für Journal oder schnelle Pools. Auch inhomogene Knoten sind möglich.

Trotz aller Flexibilität ist der Einstieg nicht trivial. Die Fehlertoleranz von Ceph schützt nicht vor Fehlkonfiguration. Know-how in Bezug auf OSDs, MONs, sowie den sogenannten CRUSH-Maps und Netzwerkverhalten ist notwendig. Fehler in der Architektur oder beim Netzdesign wirken sich sofort auf die Stabilität des Clusters aus!

# Hardwareempfehlungen für Ceph und Proxmox VE

Für einen stabilen, HA Ceph-Cluster auf Basis von Proxmox VE mit mindestens 3 Nodes konnte ich bisher folgende Hardwareanforderungen sammeln:

- Ceph OSDs und CephFS MDS benötigen hohe CPU-Leistung (viele Kerne + hoher Basistakt)
- Mindestens 8 CPU-Kerne (besser 16+) pro Node, je nach Anzahl der OSDs
- Moderne CPUs mit hohem Basistakt (≥ 2.8 GHz) und vielen Threads (z. B. AMD EPYC, Intel Xeon Silver/Gold)
- Ceph Faustregel: 1 GiB RAM pro 1 TiB Rohdaten
- Zusätzlich 3–5 GiB RAM pro OSD
- Wichtig: Genug Headroom einplanen bei VM-/Container-RAM + Ceph-RAM 
- 128 GiB ECC-RAM pro Node (für kleinere Cluster)
- Mindestens 10 Gbps (besser 25-40Gbps)

- Empfehlung: 1 Disk = 1 OSD
- SSD bevorzugt (bessere Performance und Recovery-Zeit)
- NVMe SSDs (U.2, M.2, PCIe Gen4/Gen5) für High-IO-Anwendungen
- Enterprise SATA SSDs bei Budgetgrenzen
- Keine RAID-Controller! Stattdessen: Host Bus Adapter (HBA) verwenden

