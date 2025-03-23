---
title: Server Management mit Ansible - Teil I
date: 2024-11-17
description: Server Management mit Ansible - Teil I
---

# Was ist Ansible

Ansible ist eine leistungsstarke Open-Source-Plattform für die automatisierte Serveradministration. Mit der in den Konfigurationsdateien verwendeten, einfachen YAML-Syntax und agentlosen Architektur macht Ansible es möglich, komplexe IT-Aufgaben effizient zu automatisieren. Neben Ansible gibt es zwei weitere bekannte Tools für die Konfigurationsverwaltung: Puppet und Salt. Jedes dieser Tools hat seinen eigenen Ansatz zur Automatisierung.

[https://github.com/ansible/ansible](https://github.com/ansible/ansible) 


Ein Vorteil von Ansible ist das Prinzip der Idempotenz, welches durch den deklarativen Ansatz von Ansible umgesetzt wird. Das bedeutet, dass Tasks nur dann Änderungen vornehmen, wenn sie tatsächlich nötig sind. Die in YAML definierten tasks beschreiben, so wie zum Beispiel ein Docker Compose File, den gewollten Zustand einer Aktion. Ist dieser Vorhanden werden Aktionen automatisch übersprungen. Dies ermöglicht es, Playbooks mehrfach auszuführen, ohne dass unnötige Änderungen vorgenommen werden. Letztendlich lassen sich so manche Ansibletasks ohne Bedenken regelmäßig per Cronjob ausführen. 

# Updates, Upgrades und Auto-Reboot per Ansible

In diesem Ansible Task lasse ich in meinem Homelab automatisch Updates durchführen und führe anstehende Upgrades durch. Anschließend wird eine Variable gefüllt und somit geprüft, ob im Fall eines Kernelupdates oder veralteter Bibliotheken ein reboot nötig ist. Je ob es sich um einen Hypervisor oder die darauf laufenden Gastmaschinen handelt, wird der Neustart auf eine andere Zeit verlegt. 

### Mein bisheriger Stand sieht so aus: 

---

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
---
Ein weiteres und deutlicheres Beispiel für die Grundidee von Ansible ist der folgende task mit dem wir Konfigurationsabweichungen in der SSHD Config in regelmäßigen Abständen überschreiben können. Es mag sein, dass ich einmal auf eine Passwortanmeldung ausweichen muss und dafür die config ändere oder ein künftiges Upgrade der SSHD Config die bisherigen Anpassungen verwirft. Es mag sogar sein, dass ich vergesse die Änderung rückgängig zu machen. Wioe auch immer!
<br><br>
Im Grunde stellt Ansible dann aber den durch mich eigentlich gewollten Zustand (zb. nur ssh key, keine root anmeldung etc) in regelmäßigen Zeitabständen und durch das Einfügen eines Blocks am Anfang der Config ("first come first serve" Prinzip!) wieder her. 
<br><br>
---

# Sichere SSHD Config erzwingen 

---
```
---
- hosts: all
  tasks:

  - name: sshd configuration file update
    blockinfile:
      path: /etc/ssh/sshd_config
      insertbefore: BOF # Beginning of the file
      marker: "# {mark} ANSIBLE MANAGED BLOCK BY LINUX-ADMIN"
      block: |
        PermitRootLogin no
        PubkeyAuthentication yes
        AuthorizedKeysFile .ssh/authorized_keys
        PasswordAuthentication no
      backup: yes
      validate: /usr/sbin/sshd -T -f %s

  - name: Restart SSHD
    service:
      name: sshd
      state: restarted


```
---
Das sind also zwei erste Beispiele für eine auf Ansible basierende und grundlegende Serveradministration. Durch die Verwendung von Playbooks, Roles und anderen fortgeschrittenen Funktionen lassen sich sebstverständlich auch noch komplexe Aufgaben automatisieren und wiederholen. Dazu in einem späteren Beitrag mehr.
---