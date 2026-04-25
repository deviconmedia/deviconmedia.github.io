---
title: Tutorial Backup & Sync Database antar Server
parent: TUTORIALS
nav_order: 1
layout: home
---

# Backup & Sync Database Antar VPS

![](https://t9018288956.p.clickup-attachments.com/t9018288956/041fbecd-8d70-4dc0-8b53-7a671a1f9720/Gemini_Generated_Image_teg2htteg2htteg2.png)
_Gambar oleh: Gemini AI_

Tutorial kali ini akan menjelaskan mengenai cara melakukan backup basis data otomatis dan terjadwal serta melakukan sync antar server. DBMS yang akan saya gunakan kali ini adalah **_MariaDB_** yang terinstal pada **_Ubuntu server 22.04 LTS_**. Sebelum lanjut, pastikan Anda sudah memiliki 2 server (VPS) katakan **VPS Master A (192.168.1.109)** dan **VPS Backup B (192.168.1.110)**. Jika sudah, mari kita lanjut ke langkah-langkahnya.

## 1\. Buatkan User Backup pada Server B
Untuk melakukan backup, kita siapkan sebuah user khusus pada server tujuan agar tidak perlu menggunakan root sebagai user backup. Pada tutorial ini saya membuat user baru dengan nama **_backupuser_** yang akan saya tambahkan pada server B. Jalakan perintah berikut ini.

```plain
sudo adduser --disabled-password --gecos "" backupuser
```

Perintah di atas akan membuat user baru dengan nama **_backupuser_** tanpa password. Kenapa tanpa password? Ya, karena kita tidak ingin proses backup menggunakan password, kita cukup menggunakan **_ssh key_** saja. (Akan dijelaskan di bagian selanjutnya).

## 2\. Atur SSH Config pada Server B
Untuk memastikan proses backup nantinya tidak perlu menggunakan password, maka kita akan mengatur beberapa konfigurasi pada ssh config di server B. Silakan jalankan perintah berikut ini.

```plain
sudo nano /etc/ssh/sshd_config
```

Cari baris berikut atau tambahkan jika belum ada.

```yaml
PasswordAuthentication no
ChallengeResponseAuthentication no
UsePAM yes


```

Kalau Anda ingin membatasi hanya user tertentu saja yang bisa login menggunakan ssh (misalnya root dan backupuser), tambahkan di file yang sama pengaturan berikut ini.

```plain
AllowUsers root backupuser
```

Silakan simpan pengaturan Anda lalu restart ssh.

```plain
sudo systemctl restart ssh
```

atau

```plain
sudo systemctl restart sshd
```

## 3\. Siapkan Folder Backup pada Server B
Tambahkan folder khusus di server B untuk menyimpan hasil backup yang dikirim dari server master. Jalankan perintah berikut ini.

```perl
sudo mkdir -p /home/backupuser/mariadb-backups
sudo chown backupuser:backupuser /home/backupuser/mariadb-backups 
```

Perintah di atas akan membuat sebuah folder baru dan memberikan hak akses dan kepemilikan untuk **_userbackup_**.

## 4\. Siapkan SSH Key di VPS A
Seperti yang sudah dijelaskan sebelumnya, kita tidak ingin menggunakan password saat melakukan sync database ke server B, maka kita akan membuat SSH Key terlebih dahulu sebagai pengganti password untuk user **_backupuser_** yang sudah ditambahkan di server B. Jalankan perintah berikut ini.

```bash
ssh-keygen -t ed25519 -C "backup-mariadb"
```

Silakan tekan ENTER agar tidak menggunakan passphrase.
Jika SSH Key sudah berhasil dibuat, selanjutnya kita salin SSH Key tersebut ke server B.

```sql
ssh-copy-id backupuser@192.168.1.110
```

Perhatikan bahwa IP yang digunakan pada perintah di atas adalah IP server B.
Selanjutnya, silakan tes koneksi dengan cara melakukan login ke server B melalui ssh.

```css
ssh backupuser@192.168.1.110
```

Jika berhasil masuk, maka pengaturan Anda sejauh ini sudah benar.

## 5\. Siapkan Script Backup pada Server A
Pada tahap ini, kita siapkan script backup database yang nantinya akan kita jalankan secara otomatis melalui cronjob.

Buatkan sebuah file dengan nama [mariadb-backup-mydb.sh](http://mariadb-backup-mydb.sh) pada folder /usr/local/bin.

Jalankan perintah berikut.

```bash
nano /usr/local/bin/mariadb-backup-mydb.sh
```

lalu salin script di bawah ini.

```bash
#!/bin/bash

# Konfigurasi database
USER="root"                  
PASSWORD="passwordmu"        
DBNAME="mydb"                
OUTPUT="/var/backups/mariadb"
DATE=$(date +'%Y-%m-%d_%H-%M-%S')

# VPS tujuan (VPS B)
REMOTE_USER="backupuser"
REMOTE_HOST="192.168.1.110"
REMOTE_DIR="/home/backupuser/mariadb-backups"

# 1. Backup database
mysqldump -u $USER -p$PASSWORD $DBNAME > $OUTPUT/$DBNAME-$DATE.sql

# 2. Kompres hasil backup
gzip $OUTPUT/$DBNAME-$DATE.sql

# 3. Hapus backup lama di VPS A (lebih dari 7 hari)
find $OUTPUT -type f -name "$DBNAME-*.sql.gz" -mtime +7 -delete

# 4. Sync ke VPS B
rsync -avz $OUTPUT/$DBNAME-$DATE.sql.gz $REMOTE_USER@$REMOTE_HOST:$REMOTE_DIR/


```

Perhatikan bahwa, script di atas akan melakukan backup satu database saja dengan nama mydb (sesuaikan dengan nama database Anda). Jika Anda ingin melakukan backup untuk semua database pada server A, maka ganti **_$DBNAME_** menjadi **_\--all-databases_**.
Beri izin supaya script ini bisa dieksekusi.

```perl
sudo chmod +x /usr/local/bin/mariadb-backup-mydb.sh
```

## 6\. Jalankan Script (Testing)
Jika Anda sudah berhasil membuat script untuk backup dan memberikan izin untuk dieksekusi maka, tahap ini kita akan coba menjalankan script tersebut melalui _command line_ pada server A. Jalankan perintah berikut ini.

```bash
/usr/local/bin/mariadb-backup-mydb.sh
```

Cek hasil pada server B di folder **/home/backupuser/mariadb-backups**.

```powershell
ls -lh /home/backupuser/mariadb-backups
```

Jika berhasil, maka terdapat sebuah file **.sql** (misalnya mydb-2025-08-28\_02-30-00.sql.gz) pada server B.

## 7\. Atur Cronjob
Kita tidak ingin menjalankan script ini secara manual, kita akan memanfaatkan cronjob untuk mengeksekusi script ini secara otomatis dan sesuai jam yang kita inginkan. Silakan Anda membuka pengaturan crontab pada ubuntu.

```plain
crontab -e
```

lalu tambahkan baris berikut ini.

```elixir
0 2 * * * /usr/local/bin/mariadb-backup-mydb.sh
```

**Tujuannya:** Agar backup berjalan otomatis setiap hari di jam 2 pagi tanpa perlu dijalankan manual. Anda dapat mengecek hasil backup dan _sync_ otomatis ini secara berkala.

Demikian tutorial kali ini, selamat belajar, selamat mencoba semoga bermanfaat. Amin.