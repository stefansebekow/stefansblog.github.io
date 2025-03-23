---
title: "Cage-Kiosk: Ein Wayland-Kiosk auf Basis von NixOs, Cage und Chromium"
date: 2025-03-23
description:  ""

---
Ich haben im Rahmen unserer Aufgaben mit einem Kollegen ein Kiosk-Setup entwerfen dürfen. Ziel war, dass User unbeaufsichtigt an einem Rechner sitzen sollten und drei verschiedene Seiten über einen Browser aufrufen können sollten ohne Änderungen am System vornehmen zu können. Dabei sollte kein Zugriff auf das Dateisystem möglich sein. Auch nicht-PC-versierte Nutzerinnen sollen damit zurecht kommen und die Konfiguration sollte unveränderbar sein. Damit schieden sehr simple Lösungen mit Windows-Bordmitteln oder einem Mozilla Kiosk aus.

Bei der folgenden Lösung handelt es sich um NixOS 24.11, das nach dem Start "cage" ausführt.  Cage ist ein Wayland-Compositor für Kiosk-Anwendungen der ursprünglich ein Projekt von Jente Hidskes ist. In dieser Sandbox wird dann wiederum ein Chromium im Kiosk-Modus gestartet. 
Aller Dank gebührt: 
>
(https://github.com/cage-kiosk/cage) 
>

In dem Chromium läuft ein Add-On aus dem Chrome-Store "Kiosk-Extension", das eine Whitelist für die erlaubten Websites verwaltet und zudem als Overlay drei Buttons zur Verfügung stellt, mit denen die Nutzer bequem zwischen den 3 Seiten wechseln können. Tab- und Adressleiste sind im Kiosk-Modus aber ausgeschaltet.
>
https://chromewebstore.google.com/detail/kiosk-extension/hbpkaaahpgfafhefiacnndahmanhjagi
>
Die Iso für NixOS kann auf der Website geladen werden. Die Installation benötigt Internet. NixOs kann mit folgenden Kommandos von der Live-ISO installiert werden:

# Installationsskript


```
#!/bin/sh
# Skript installiert NixOS mit Chrome im Kiosk-Modus in einer Cage-Umgebung
echo "NixOS-Installationsskript"
echo "Mit sudo ausführen"

# Löschen von /dev/sda und erstellen von 2 neuen Partitionen (2GB EFI, Rest root)
sgdisk -og /dev/sda
sgdisk -n 1:2048:4196351 -t 1:ef00 -c 1:NIXBOOT /dev/sda
sgdisk -n 2:4196352:0 -t 2:8300 -c 2:NIXROOT /dev/sda

sgdisk -p /dev/sda

mkfs.fat -F 32 /dev/sda1
fatlabel /dev/sda1 NIXBOOT
mkfs.ext4 /dev/sda2 -L NIXROOT

mount /dev/disk/by-label/NIXROOT /mnt
mkdir -p /mnt/boot
mount /dev/disk/by-label/NIXBOOT /mnt/boot

# Configs erstellen und vorbereitete Config kopieren
nixos-generate-config root /mnt
cp /iso/configuration.nix /mnt/etc/nixos
cd /mnt

#
proxy="http://FooBar/:Port"
export http_proxy="$proxy"
export https_proxy="$proxy"
export HTTP_PROXY="$proxy"
export HTTPS_PROXY="$proxy"

nixos-install

```

Die configuration.nix muss vorher auf einem LiveUSB-Stick abgelegt werden. Nach dem Reboot startet das System in den Kiosk-Modus. Hier installiert man die Erweiterung aus dem Chrome Web Store, macht ggf. noch weitere Anpassungen an der Config des Browser und schließt diesen dann. Mit [STRG]+[ALT]+[F2] wechselt man in tty2, meldet sich mit den Zugangsdaten an und kann die Konfigurationsdatei so bearbeiten, dass das System beim nächsten mal direkt fertig konfiguriert ist. Anschließend startet man einmal, installiert das KIOSK-AddOn im Chromium und lädt die Konfigurationsdatei. Eine Beispiel JSON findet sich ganz unten! 

Mit 
`nixos-rebuild switch` 
kann man anschließend die Änderung aktivieren. Das gerade genannte Vorgehen ist auch die einzige Möglichkeit, um die Rechner zu warten.


# Die configuration.nix

```
# Edit this configuration file to define what should be installed on
# your system.  Help is available in the configuration.nix(5) man page
# and in the NixOS manual (accessible by running ‘nixos-help’).

{ config, pkgs, ... }:

{
  imports =
    [ # Include the results of the hardware scan.
      ./hardware-configuration.nix
    ];

  # Bootloader.
  boot.loader.systemd-boot.enable = true;

  networking.hostName = "Kiosk-YOURPREFEREDNAME"; # Define your hostname.
  # networking.wireless.enable = false;  # Enables wireless support via wpa_supplicant.

  # Configure network proxy if necessary
  networking.proxy.default = "http://FooBar:Proxy";
  networking.proxy.noProxy = "127.0.0.1,localhost";

  # Enable networking
  networking.networkmanager.enable = true;

  # Set your time zone!
  time.timeZone = "Europe/Berlin";

  # Select internationalisation properties!
  i18n.defaultLocale = "de_DE.UTF-8";

  i18n.extraLocaleSettings = {
    LC_ADDRESS = "de_DE.UTF-8";
    LC_IDENTIFICATION = "de_DE.UTF-8";
    LC_MEASUREMENT = "de_DE.UTF-8";
    LC_MONETARY = "de_DE.UTF-8";
    LC_NAME = "de_DE.UTF-8";
    LC_NUMERIC = "de_DE.UTF-8";
    LC_PAPER = "de_DE.UTF-8";
    LC_TELEPHONE = "de_DE.UTF-8";
    LC_TIME = "de_DE.UTF-8";
  };

  console = {
    earlySetup = true;
    useXkbConfig = true;
  };
  # Configure keymap in X11
  services.xserver = {
    xkb.layout = "de";
    xkb.variant = "nodeadkeys";
  };

  systemd.services."cage-tty1".after = [
    "network-online.target"
    "systemd-resolved.service"
  ];

  programs.chromium = {
    enable = true;
	package = pkgs.chromium;
	
	extraOpts = {
	   "BrowserSignin" = 0;
       "SyncDisabled" = true;
       "PasswordManagerEnabled" = false;
       "SpellcheckEnabled" = false;
	};
  };

  systemd.services.cage-tty1.environment.XKB_DEFAULT_LAYOUT = "de";

  services.cage = {
    enable = true;
    user = "kiosk";
#    program = "${pkgs.chromium}/bin/chromium https://FOOBAR.COM/ --kiosk --lang=de --enable-extensions --noerrdialogs --no-first-run";
    program = "${pkgs.chromium}/bin/chromium --lang=de --noerrdialogs --no-first-run";
  };

  systemd.services.cage-tty1.environment

  # Define a user account. Don't forget to set a password with ‘passwd’.
  users.users.kiosk = {
    isNormalUser = true;
    description = "FOOBAR";
    extraGroups = [ "networkmanager" "wheel"];
    packages = with pkgs; [];
  };

  # Allow unfree packages
  nixpkgs.config.allowUnfree = true;

  environment.systemPackages = with pkgs; [
    vim
    pkgs.chromium
  ];

  # Some programs need SUID wrappers, can be configured further or are
  # started in user sessions.
  # programs.mtr.enable = true;
  # programs.gnupg.agent = {
  #   enable = true;
  #   enableSSHSupport = true;
  # };

  # List services that you want to enable:

  # Enable the OpenSSH daemon.
  #services.openssh.enable = true;

  # Open ports in the firewall.
  # networking.firewall.allowedTCPPorts = [ 80,443 ];
  # networking.firewall.allowedUDPPorts = [ ... ];
  # Or disable the firewall altogether.
  networking.firewall.enable = true;

  # This value determines the NixOS release from which the default
  # settings for stateful data, like file locations and database versions
  # on your system were taken. It‘s perfectly fine and recommended to leave
  # this value at the release version of the first install of this system.
  # Before changing this value read the documentation for this option
  # (e.g. man configuration.nix or on https://nixos.org/nixos/options.html).
  system.stateVersion = "22.11"; # Did you read the comment?

  environment.sessionVariables = {
    MOZ_ENABLE_WAYLAND = "1";
	NIXOS_OZONE_WL = "1";
  };

}


```

# CONFIG.JSON für Chromium

```
{
  "created": "2025-01-10T14:22:28.426Z",
  "version": "0.0.0.36",
  "options": {
    "admin_pass": "PASSWORD",
    "allowsaml": true,
    "arrayEB": [
      {
        "bgcolor": "rgba(47, 47, 47, 0.36)",
        "fontcolor": "rgb(255, 255, 255)",
        "fontsize": 20,
        "id": 1,
        "img": "",
        "imgrz": "20%",
        "imgsize": 20,
        "rx": 321.421875,
        "ry": 3,
        "tab": false,
        "text": "SITENAME",
        "url": "https://FOOBAR.COM",
        "x": 1585,
        "y": 1053
      },
      {
        "bgcolor": "rgba(47, 47, 47, 0.36)",
        "fontcolor": "rgb(255, 255, 255)",
        "fontsize": 20,
        "id": 2,
        "img": "",
        "imgrz": "20%",
        "imgsize": 20,
        "rx": 97.109375,
        "ry": 3,
        "tab": false,
        "text": "SITENAME2",
        "url": "https://FOOBAR2.COM",
        "x": 1833,
        "y": 1053
      },
      {
        "bgcolor": "rgba(47, 47, 47, 0.36)",
        "fontcolor": "rgb(255, 255, 255)",
        "fontsize": 20,
        "id": 3,
        "img": "",
        "imgrz": "20%",
        "imgsize": 20,
        "rx": 9.015625,
        "ry": 4,
        "tab": false,
        "text": "SITENAME3",
        "url": "https://FOOBAR3.COM",
        "x": 2059,
        "y": 1053
      }
    ],
    "backbutton": false,
    "backnavbutton": true,
    "bblist": "exclude",
    "blockkeys": true,
    "darkmode": "false",
    "dateformat": "iso",
    "disablemailto": true,
    "disabletel": true,
    "disableurlchange": false,
    "ebbuttons": true,
    "eblist": "exclude",
    "enaaltbbfunction": false,
    "enaclock": false,
    "enaclockdate": false,
    "enaclockdaterow": false,
    "enaclockfixedpos": false,
    "enaclockseconds": false,
    "enaclocktime": true,
    "enaclockweek": false,
    "enanohistory": false,
    "enatimer": false,
    "enazoomreset": false,
    "forwardnavbutton": true,
    "homenavbutton": false,
    "homepage": "https://FOOBAR.COM",
    "idlerefresh": false,
    "idletimeout": "600",
    "langformat": "de-DE",
    "listfunction": "allowlist",
    "navbcol": false,
    "navbuttons": true,
    "nohprefresh": true,
    "nomediarefresh": true,
    "openurl": true,
    "opt_backbutton": {
      "bgcolor": "rgba(47, 47, 47, 0.36)",
      "buttonfunction": "back",
      "fontcolor": "rgb(255, 255, 255)",
      "fontsize": 20,
      "rx": 2024.109375,
      "ry": 7,
      "text": "Zur ck",
      "url": "",
      "x": 13,
      "y": 1228
    },
    "opt_navbutton": {
      "fontcolor": "rgb(0, 0, 0)",
      "fontsize": 31,
      "rx": 1984.21875,
      "ry": 2,
      "x": 26,
      "y": 1057
    },
    "opt_qrbutton": {
      "bgcolor": "rgba(47, 47, 47, 0.36)",
      "fontcolor": "rgb(255, 255, 255)",
      "fontsize": 14,
      "text": "Show QR Code of this Page",
      "x": 50,
      "y": 150
    },
    "opt_secretbutton": {
      "bgcolor": "rgba(47, 47, 47, 0.36)",
      "size": 20,
      "x": 25,
      "y": 25
    },
    "opt_timedateoverlay": {
      "bgcolor": "rgba(47, 47, 47, 0.36)",
      "dateFormat": "iso",
      "fixedPos": false,
      "fontcolor": "rgb(255, 255, 255)",
      "fontsize": 20,
      "showDate": false,
      "showSeconds": false,
      "showTime": true,
      "showWeek": false,
      "timeDaterow": false,
      "timedatealign": "center",
      "timeformat": "24h",
      "x": 25,
      "y": 25
    },
    "optclearcacheidle": true,
    "optusealtidlepage": false,
    "qrbuttonlocation": "above",
    "qrcodebutton": false,
    "qrcodebuttonfavicon": false,
    "qrcodebuttonlink": false,
    "qrcodebuttonlogo": true,
    "refreshnavbutton": false,
    "secretbutton": false,
    "tdlist": "exclude",
    "timedatealign": "center",
    "timeformat": "24h",
    "upnavbutton": false,
    "urllist_l01": [],
    "urllist_l02": [
      "FOOBAR/*",
      "FOOBAR2.COM/*",
      "FOOBAR3.COM/*"
    ],
    "urllist_l03": [],
    "urllist_l04": [],
    "urllist_l05": [],
    "usecssfile": false,
    "zoomlevel": "100"
  }
}
```


## Vielen Dank an Peter fürs Vervollständigen und an SergeantBiggs für den initiallen Hinweis auf Cage und den NixOs Crashkurs :)  