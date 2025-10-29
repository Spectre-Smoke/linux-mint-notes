# Linux Mint Notes

Kompakte, praxisnahe Notizen für **Linux Mint (Ubuntu-Basis)**: Setup, Systeminfo, Paketverwaltung (APT/Snap/Flatpak), PPAs/Keys, Dienste & Logs, Netzwerk, Firewall (UFW), Benutzer & Rechte, Storage & fstab, Backups (Timeshift & rsync+Cron), Grafiktreiber, Pflege & nützliche Einzeiler.

**Inhalt:**  
[Quickstart](#quickstart) · [Systeminfo & Tools](#systeminfo) · [Pakete: APT / Snap / Flatpak](#packages) · [PPAs & Repo-Keys](#ppa) · [Dienste & Logs](#services) · [Netzwerk](#network) · [Firewall (UFW)](#ufw) · [Benutzer & Rechte](#users) · [Storage & fstab](#storage) · [Backups: Timeshift](#timeshift) · [rsync + Cron](#backup-cron) · [Grafiktreiber](#graphics) · [Systempflege / Updates](#maintenance) · [Einzeiler](#oneliners) · [Troubleshooting](#troubleshooting)

---

<a id="quickstart"></a>
## 🚀 Quickstart

```bash
# System aktualisieren
sudo apt update && sudo apt full-upgrade -y

# Nützliche Tools
sudo apt install -y curl git htop neofetch inxi ufw

# Systemzustand / Fehler
lsb_release -a
journalctl -p err -n 50 --no-pager
```

---

<a id="systeminfo"></a>
## 🧰 Systeminfo & Tools

```bash
inxi -Fazy             # volle Hardware/Driver-Übersicht
neofetch               # hübsche Kurzübersicht
uname -a               # Kernel
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT,LABEL
lsusb; lspci -nn
```

---

<a id="packages"></a>
## 📦 Pakete: APT / Snap / Flatpak

```bash
# APT
apt search <paket>
apt show <paket>
sudo apt install <paket>
sudo apt remove <paket> && sudo apt autoremove --purge -y
apt-cache policy <paket>         # Version/Quelle
sudo apt-mark hold <paket>       # Paket pinnen (kein Update)
sudo apt-mark unhold <paket>

# Snap (optional)
snap find <app>
sudo snap install <app>          # ggf. --classic
snap list

# Flatpak (optional, inkl. Flathub)
sudo apt install -y flatpak
sudo flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
flatpak search <app>
sudo flatpak install flathub <app-id>
```

---

<a id="ppa"></a>
## 🧩 PPAs & Repo-Keys

> PPAs sparsam verwenden (Stabilität!). Für moderne Repos: `signed-by` nutzen.

```bash
# PPA hinzufügen / entfernen
sudo add-apt-repository ppa:<owner>/<ppa> -y && sudo apt update
sudo apt install -y ppa-purge && sudo ppa-purge ppa:<owner>/<ppa>

# Externes Repo (Beispiel, signed-by)
curl -fsSL https://example.com/repo.gpg | sudo gpg --dearmor -o /usr/share/keyrings/example.gpg
echo "deb [signed-by=/usr/share/keyrings/example.gpg] https://example.com/ubuntu noble main" |
  sudo tee /etc/apt/sources.list.d/example.list
sudo apt update
```

---

<a id="services"></a>
## 🧭 Dienste & Logs (systemd/journal)

```bash
# Dienste
systemctl status <dienst>
sudo systemctl enable --now <dienst>
sudo systemctl restart <dienst>

# Logs
journalctl -u <dienst> -e -f
journalctl -p warning -xb
```

---

<a id="network"></a>
## 🌐 Netzwerk

```bash
# Überblick & aktive Verbindungen
ip -br a
ss -tulpn | grep LISTEN

# WLAN (NetworkManager)
nmcli dev status
nmcli dev wifi list
nmcli dev wifi connect "SSID" password "PASS"
```

---

<a id="ufw"></a>
## 🔒 Firewall (UFW)

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status verbose
```

---

<a id="users"></a>
## 👤 Benutzer & Rechte

```bash
# User + sudo
sudo adduser <name>
sudo usermod -aG sudo <name>
groups <name>

# Dateirechte
sudo chown -R <user>:<group> /pfad
chmod 640 file
chmod -R 755 /opt/app
```

---

<a id="storage"></a>
## 💾 Storage & fstab

```bash
lsblk -f
df -h
sudo blkid

# fstab (Beispiel)
# /etc/fstab
UUID=xxxx-xxxx  /data  ext4  defaults,noatime  0 2
sudo mkdir -p /data && sudo mount -a

# Große Ordner finden (Top 20)
sudo du -xh / | sort -h | tail -n 20
```

---

<a id="timeshift"></a>
## 🕒 Backups: Timeshift (System-Snapshots)

```bash
# Timeshift (GUI/CLI) – Systemzustände sichern & zurückrollen
sudo apt install -y timeshift
sudo timeshift --check                   # Status
sudo timeshift --create --comments "pre-update"
sudo timeshift --list
```

**Tipp:** Timeshift fokussiert Systemdateien; Nutzerdaten separat sichern (z. B. rsync).

---

<a id="backup-cron"></a>
## 🔄 rsync + Cron (Daten-Backup mit Rotation)

**Skript** ` /usr/local/bin/backup_data.sh`:

```bash
sudo tee /usr/local/bin/backup_data.sh >/dev/null <<'SH'
#!/usr/bin/env bash
set -euo pipefail

SRC="$HOME/data"              # <- anpassen
DST="/srv/backup"             # Zielbasis
STAMP="$(date +%F_%H-%M-%S)"
DEST="${DST}/${STAMP}"

mkdir -p "$DEST"
rsync -aHAX --delete --info=stats2 "${SRC}/" "${DEST}/data/"

# Aufräumen: alles älter 7 Tage
find "$DST" -maxdepth 1 -type d -name '20*' -mtime +7 -print -exec rm -rf {} \;
SH
sudo chmod +x /usr/local/bin/backup_data.sh
```

**Cron (täglich 03:30, mit Lock):**
```cron
0 3 * * * /usr/bin/flock -n /var/lock/backup_data.lock /usr/local/bin/backup_data.sh >>/var/log/backup_data.log 2>&1
```

Logs:
```bash
tail -n 100 /var/log/backup_data.log
```

---

<a id="graphics"></a>
## 🖥️ Grafiktreiber (NVIDIA & Co.)

```bash
# Empfohlene Treiber anzeigen/automatisch installieren
ubuntu-drivers list
sudo ubuntu-drivers autoinstall

# Aktive GPU
lspci -nn | grep -i 'vga\|3d'
```

> Mint hat zusätzlich GUI-Tools (Treiber-Manager). Für stabilen Betrieb besser die empfohlenen Treiber nutzen.

---

<a id="maintenance"></a>
## 🛠️ Systempflege / Updates

```bash
# Regelmäßig
sudo apt update && sudo apt full-upgrade -y
sudo apt autoremove --purge -y
sudo apt clean

# Boot-Logs/Fehler
journalctl -b -p err
```

---

<a id="oneliners"></a>
## 🧰 Einzeiler

```bash
# Welche Unit nutzt Port 80?
sudo ss -tulpn | awk '/:80 /{print}'

# Letzte 100 Fehler
journalctl -p err -n 100 --no-pager

# Snap-Revisionen aufräumen (nur neueste behalten)
snap list --all | awk '/disabled/{print $1, $3}' | while read n r; do sudo snap remove "$n" --revision="$r"; done

# Ausgehende Verbindung testen (443/tcp)
nc -vz 1.1.1.1 443
```

---

<a id="troubleshooting"></a>
## 🧯 Troubleshooting

- **Paket hängt/fehlerhaft:** `sudo dpkg --configure -a && sudo apt -f install`.  
- **Paket „gehalten“:** `apt-mark showhold` prüfen; ggf. `sudo apt-mark unhold <paket>`.  
- **Netzwerk down nach Änderungen:** `nmcli` prüfen / ggf. `sudo systemctl restart NetworkManager`.  
- **Kein Platz:** Große Ordner prüfen (`du -xh`), APT-Cache/Snap-Revisionen aufräumen.  
- **Nach Update Probleme:** Timeshift-Snapshot zurückrollen; Kernel-Auswahl im Grub testen.