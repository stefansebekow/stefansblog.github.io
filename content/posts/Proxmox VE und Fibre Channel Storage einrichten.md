---
title: "Fibre Channel SAN-Storage und Proxmox-Cluster per Multipath integrieren"
date: 2026-03-01
tags: ["Proxmox", "Fibre Channel", "Multipath", "SAN", "Cluster", "LVM"] 
categories: ["Virtualisierung", "Storage", "Linux"]
---

 # Hochverfügbares SAN-Storage im Proxmox-Cluster sauber integrieren

In diesem Artikel zeige ich praxisnah, wie man in einem 3‑Node‑Proxmox‑Cluster mit Fibre‑Channel‑Anbindung ein Shared Storage sauber und hochverfügbar einbindet. Ziel ist es, zwei vom SAN bereitgestellte LUNs über "DM‑Multipath" zu bündeln und anschließend als Shared LVM Storage im Cluster bereitzustellen.

--- 

## Ausgangsszenario 

Wir betreiben einen vollständig eingerichteten Drei‑Node‑Cluster auf Basis von Proxmox VE. Der Cluster ist stabil über Corosync verbunden.

Jeder Cluster‑Node verfügt über zwei Fibre‑Channel‑HBAs mit jeweils zwei Ports. Auf Storage‑Seite stehen zwei redundant am SAN‑Switch angebundene Controller (Controller A und B) zur Verfügung, die jeweils zwei Ports besitzen. Der Einfachheit halber kommt in diesem Beispielszenario ein einzelner Fibre‑Channel‑Switch zum Einsatz, über den sämtliche Proxmox‑Nodes mit dem SAN verbunden sind.

Die Zonierung des SANs ist nach dem Prinzip des Single‑Initiator‑Zoning aufgebaut. Das bedeutet: Jeder HBA‑Port eines einzelnen Servernodes befindet sich zu den anderen Ports abgetrennt in einer eigenen Zone und kommuniziert damit zur selben Zeit ausschließlich mit einem dedizierten Port des SAN‑Controllers.

Auf dem SAN‑Storage wurden bei der Ersteinrichtung bereits zwei LUNs bereitgestellt, die allen drei Nodes präsentiert werden. Pro Server ergeben sich ohne Multipathing damit insgesamt 16 sichtbare Blockgeräte/Pfade:

```
2 HBA-Ports pro Server × 2 SAN-Controller × 2 Ports pro Controller × 2 LUNs = 16 Blockdevices
```

## Warum Multipath zwingend notwendig ist

Die Einrichtung von Multipath‑Connectivity in einem Proxmox‑Cluster ist eine Voraussetzung für den stabilen und hochverfügbaren HA Betrieb von VMS, die von einem SAN bereitgestellt werden. Multipathing sorgt dafür, dass auf ein Storage‑Gerät redundant zugegriffen werden kann. Fällt ein Pfad oder eine Komponente aus, bleibt der Zugriff auf das Gerät erhalten und greift auf die sekundären Pfade zurück. Das sogenannte *Device Mapper Multipathing (DM‑Multipathing)* sorgt dafür, dass mehrere physische Pfade zu einer LUN als ein logisches Gerät zusammengeführt werden können. Fällt ein FC‑Pfad, ein HBA oder ein Controller aus, bleibt der Zugriff bestehen.

Wichtig ist dabei: Alle Cluster‑Knoten müssen auf dasselbe Multipath‑Gerät zugreifen, damit LVM‑basierte VMs korrekt funktionieren. Nur wenn jedes Node exakt dieselbe LUN über denselben WWID‑Multipath‑Device sieht, ist ein stabiler und hochverfügbarer Betrieb über die gemeinsam geteilten Metadaten möglich.


---

## Schritt 1: Multipath-Tools installieren und Dienst aktivieren

Auf allen Proxmox‑Nodes:

```
apt update
apt install -y multipath-tools

systemctl enable multipathd
systemctl start multipathd
```

## Schritt 2: FC-Bus neu scannen und Geräte identifizieren

```
for host in /sys/class/scsi_host/host*; do echo "- - -" > "$host/scan" done
```

## Blockgeräte anzeigen und WWID der Geräte ermitteln

```
lsblk
#oder#
for disk in sda sdb sdc sdd sde sdf sdg sdh; do echo $disk /lib/udev/scsi_id -g -u -d /dev/$disk done
```
## Schritt 3: Multipath prüfen und WWIDs hinzufügen

```
multipath -ll
multipath -a <WWID> 
multipath -r
# Für alle WWIDs wiederholen!
systemctl restart multipathd
```
## Beispielausgabe der korrekt konfigurierten Multipath-Setups bei zwei Luns über jeweils 8 gebündelte Pfase 

Teile der Ausgabe sind mit […] abgekürzt

Interpretation:

    prio=50 = bevorzugter Pfad (optimierter Controller)

    prio=10 = sekundärer Pfad (nicht optimiert, Failover)

    Alle Pfade sind active ready running → korrekt.

```
root@pve-02:/home/USER# multipath -ll

WWID---- dm-5 […]
size=[…] features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' *prio=50 status=enabled*
| |- 1:0:3:2 sdg 8:96  active ready running
| |- 1:0:1:2 sdc 8:32  active ready running
| |- 2:0:3:2 sdo 8:224 active ready running
| `- 2:0:1:2 sdk 8:160 active ready running
`-+- policy='service-time 0' *prio=10 status=enabled*
  |- 1:0:0:2 sda 8:0   active ready running
  |- 1:0:2:2 sde 8:64  active ready running
  |- 2:0:0:2 sdi 8:128 active ready running
  `- 2:0:2:2 sdm 8:192 active ready running
[…]
size=[…] features='1 queue_if_no_path' hwhandler='1 alua' wp=rw
|-+- policy='service-time 0' *prio=50 status=enabled*
| |1:0:3:2 sdg 8:96  active ready running
| |- 1:0:1:2 sdc 8:32  active ready running
| |- 2:0:3:2 sdo 8:224 active ready running
| `- 2:0:1:2 sdk 8:160 active ready running
`-+- policy='service-time 0' *prio=10 status=enabled*
   |- 1:0:0:2 sda 8:0   active ready running
  |- 1:0:2:2 sde 8:64  active ready running
  |- 2:0:0:2 sdi 8:128 active ready running
  `- 2:0:2:2 sdm 8:192 active ready running
```

## Schritt 4: LVM auf dem Multipath-Device anlegen

Physical Volume erstellen

```
pvcreate /dev/mapper/<WWID>

```

Volume Group erstellen

```
vgcreate <Volumegruppenname> /dev/mapper/<WWID>

```
Falls das neue LVM auf anderen Nodes nicht erkannt wird:
```
pvscan --cache
```
## Schritt 5: Storage als Shared LVM in Proxmox einbinden

In der Proxmox‑Gui:

* Datacenter → Storage → Add → LVM
* Volume Group auswählen (z. B. pve-vm01)
* Haken bei Shared setzen
* Speichern

Damit steht das SAN‑Storage clusterweit zur Verfügung. Je nach SAN Storage wird eine angepasste Multipath.conf benötigt, die zb. verhindert das lokale Geräte beim Boot im Multipathing gemounted werden: 

```
# /etc/multipath.conf
#
# Generische Multipath-Konfiguration für Fibre-Channel SAN-Storage


defaults {
    user_friendly_names     no
    find_multipaths         yes
    polling_interval        5
    max_fds                 8192
    queue_without_daemon    no
    flush_on_last_del       yes
}

# Lokale Platten ausschließen
blacklist {
    devnode "^sd[a-z]$"
}

# SAN-Storage erlauben (Platzhalter für Vendor/Product)
blacklist_exceptions {
    device {
        vendor  "<SAN_VENDOR>"
        product "<SAN_PRODUCT>"
    }
}

devices {
    device {
        vendor                  "<SAN_VENDOR>"
        product                 "<SAN_PRODUCT>"
        path_grouping_policy    group_by_prio
        path_selector           "service-time 0"
#je nach hardwarehandler
        hardware_handler        "1 alua"
        prio                    alua
        failback                immediate
        no_path_retry           30
        rr_weight               uniform
        fast_io_fail_tmo        5
        dev_loss_tmo            30
    }
}

multipaths {
    # Optional: feste Zuordnung für bestimmte LUNs
    # multipath {
    #     wwid    <WWID_LUN_01>
    #     alias   san_lun01
    # }
    #
    # multipath {
    #     wwid    <WWID_LUN_02>
    #     alias   san_lun02
    # }
}

```
