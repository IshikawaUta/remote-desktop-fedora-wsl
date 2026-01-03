![Tampilan Desktop OpenSESU Leap](https://res.cloudinary.com/dzsqaauqn/image/upload/v1767425701/Screenshot_2026-01-03_141947_pefxdv.jpg)
---

# Panduan Lengkap: Web-Based Remote Desktop Gateway Fedora WSL (noVNC & Cloudflare Tunnel)

Panduan ini mencakup instalasi GUI XFCE di Fedora WSL, konfigurasi server VNC, dan penggunaan Docker untuk membuat gateway remote desktop berbasis web yang bisa diakses secara global tanpa port forwarding.

---

## Bagian 1: Konfigurasi Fedora WSL

### 1. Update dan Instalasi Desktop

Pertama, pastikan sistem Fedora Anda diperbarui dan paket desktop environment terpasang.

* **Update sistem:** `sudo dnf update`
* **Instal XFCE4 Desktop:** `sudo dnf install @xfce-desktop-environment`
* **Instal TigerVNC Server:** `sudo dnf install tigervnc-server`
* **Instal modul pendukung:** `sudo dnf install -y dbus-x11 xset rootlesskit`

### 2. Pengaturan Keamanan dan Startup

* **Buat password akses VNC:** `vncpasswd` (Pilih **n** untuk opsi *view-only*).
* **Buat folder konfigurasi:** `mkdir -p ~/.vnc`
* **Buat file startup:** `nano ~/.vnc/xstartup`

**Isi file `xstartup`:**

```bash
#!/bin/sh
unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS

if [ -z "$DBUS_SESSION_BUS_ADDRESS" ]; then
    eval $(dbus-launch --sh-syntax --exit-with-session)
fi

export XDG_CURRENT_DESKTOP="XFCE"
export XDG_SESSION_TYPE=x11
export GDK_BACKEND=x11
export QT_QPA_PLATFORM=xcb
exec startxfce4

```

*Simpan dengan menekan `CTRL+X`, lalu `Y`, dan `Enter`.* Berikan izin eksekusi: `chmod +x ~/.vnc/xstartup`.

### 3. Menjalankan VNC Server

Bersihkan sisa sesi lama dan jalankan server pada display `:5` (Port 5905):

```bash
vncserver -kill :* 2>/dev/null
sudo rm -f /tmp/.X*-lock && sudo rm -f /tmp/.X11-unix/X*
vncserver :5 -geometry 1366x768 -depth 24

```

---

## Bagian 2: Web Based Gateway (Docker)

Gunakan konfigurasi `docker-compose.yml` berikut. Pastikan Anda berada di folder yang sama dengan file `.env`.

**docker-compose.yml:**

```yaml
services:
  novnc:
    image: geek1011/easy-novnc:latest
    container_name: novnc
    ports:
      - "6080:8080"
    extra_hosts:
      - "host.docker.internal:host-gateway"
    command: ["--addr", "0.0.0.0:8080", "--host", "host.docker.internal", "--port", "5905", "--no-url-password"]
    restart: unless-stopped

  cloudflare-tunnel:
    image: cloudflare/cloudflared:latest
    container_name: cloudflare-tunnel
    command: tunnel --no-autoupdate run --token ${CLOUDFLARE_TOKEN}
    restart: unless-stopped

```

---

## Bagian 3: Konfigurasi Cloudflare Tunnel

Untuk mendapatkan **CLOUDFLARE_TOKEN**, ikuti langkah berikut:

1. **Buka Dashboard:** Masuk ke [Cloudflare Zero Trust](https://one.dash.cloudflare.com/).
2. **Buat Tunnel:** Pergi ke **Networks** > **Tunnels** > **Create a Tunnel**.
3. **Pilih Cloudflared:** Beri nama tunnel (misal: `fedora-wsl`), lalu klik **Save**.
4. **Salin Token:** Di bagian "Install and run a connector", pilih environment Docker. Salin kode token panjang yang muncul (setelah teks `--token`).
5. **Set Up Hostname:** * Klik tab **Public Hostname** pada tunnel yang baru dibuat.
* Klik **Add a public hostname**.
* Isi **Subdomain** (misal: `desktop`) dan pilih **Domain** Anda.
* Pilih **Service Type:** `HTTP` dan **URL:** `novnc:8080`.
* Klik **Save**.



---

## Bagian 4: Cara Menggunakan

1. **Siapkan Environment:**
Buat file `.env` di folder yang sama dengan `docker-compose.yml`:


```bash
CLOUDFLARE_TOKEN=ISI_DENGAN_TOKEN_DARI_CLOUDFLARE_ANDA

```


2. **Jalankan Docker Stack:**


```bash
docker-compose up -d

```


3. **Akses Desktop:**
* Buka browser dan akses URL yang Anda buat (misal: `https://desktop.domainanda.com`).
* Klik **Connect** pada antarmuka noVNC.
* Masukkan password VNC yang Anda buat di langkah `vncpasswd`.
* **Selesai!** Desktop XFCE4 Fedora Anda kini bisa diakses dari mana saja.
