---
title: Server Management mit Ansible - Teil I
date: 2024-11-17
description: Server Management mit Ansible - Teil I
---

Ansible ist eine leistungsstarke Open-Source-Plattform für die automatisierte Serveradministration. Mit der in den Konfigurationsdateien verwendeten, einfachen YAML-Syntax und agentlosen Architektur macht Ansible es möglich, komplexe IT-Aufgaben effizient zu automatisieren. Neben Ansible gibt es zwei weitere bekannte Tools für die Konfigurationsverwaltung: Puppet und Salt. Jedes dieser Tools hat seinen eigenen Ansatz zur Automatisierung.

Ein Vorteil von Ansible ist das Prinzip der Idempotenz, welches durch den deklarativen Ansatz von Ansible umgesetzt wird. Das bedeutet, dass Tasks nur dann Änderungen vornehmen, wenn sie tatsächlich nötig sind. Die in YAML definierten tasks beschreiben, so wie zum Beispiel ein Docker Compose File, den gewollten Zustand einer Aktion. Ist dieser Vorhanden werden Aktionen automatisch übersprungen. Dies ermöglicht es, Playbooks mehrfach auszuführen, ohne dass unnötige Änderungen vorgenommen werden. Letztendlich lassen sich so manche Ansibletasks ohne Bedenken regelmäßig per Cronjob ausführen. 

In diesem Ansible Task lasse ich in meinem Homelab automatisch Updates durchführen und prüfe, ob im Fall eines Kernelupdates oder veralteter Bibliotheken ein reboot nötig ist. Je ob es sich um einen Hypervisor oder die darauf laufenden Gastmaschinen handelt, wird der Neustart auf eine andere Zeit verlegt. 

Das sieht so aus: 

```
---
- name: Update and Upgrade Debian Servers
  hosts: all
  gather_facts: yes
  become: yes

  tasks:

    - name: Update apt cache
      apt:
        update_cache: yes
        cache_valid_time: 86400

    - name: Upgrade packages
      apt:
        upgrade: yes

    - name: Remove dependencies that are no longer required.
      ansible.builtin.apt:
        autoremove: yes

    - name: Check if a reboot is required.
      ansible.builtin.stat:
        path: /var/run/reboot-required
        get_checksum: no
      register: reboot_required_file

    - name: Schedule reboot at 02:00 if not a host machine
      ansible.builtin.command: sudo shutdown -r 02:00
      when:
        (reboot_required_file.stat.exists) and
        (inventory_hostname not in groups[pve_hosts])

    - name: Schedule reboot at 04:00 if not a guest machine
      ansible.builtin.command: sudo shutdown -r 04:00
      when:
        (reboot_required_file.stat.exists) and
        (inventory_hostname in groups[pve_hosts])


```
