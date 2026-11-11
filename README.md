# FIXNYA
Nah ini penyakitnya SlowDNS bos, udah ketebak.

SlowDNS itu maksa pake *port 53*. Sedangkan di Ubuntu, port 53 itu udah dipake duluan sama `systemd-resolved` buat DNS-nya VPS.

Jadi pas lu install SlowDNS, dia rebutan port 53, `systemd-resolved` nya mati, DNS-nya ilang, IP-nya jadi gak ke-detect, pas reboot tambah parah, `resolv.conf` ke-reset lagi.

Sama satu lagi, kalau di AWS lu *stop/start* VPS, IP Public-nya emang ganti terus, kecuali lu pake Elastic IP.

*Biar IP gak ilang pas reboot, lakuin 3 ini sekaligus:*

### 1. Matikan yang rebutan Port 53
````
systemctl stop systemd-resolved
systemctl disable systemd-resolved
systemctl stop systemd-resolved.service
````
Terus bikin config biar dia gak nyala lagi:
````
mkdir -p /etc/systemd/resolved.conf.d
cat > /etc/systemd/resolved.conf.d/no-stub.conf <<EOF
[Resolve]
DNSStubListener=no
EOF
````
### 2. Bikin DNS-nya permanen, gak ke-reset pas reboot
````
rm -f /etc/resolv.conf
printf "nameserver 8.8.8.8\nnameserver 1.1.1.1\nnameserver 169.254.169.253\n" > /etc/resolv.conf
chattr +i /etc/resolv.conf
cat /etc/resolv.conf
````
`chattr +i` itu kunci biar file-nya gak bisa ditimpa lagi sama sistem pas reboot.

### 3. Bikin IP di `/root/ip` auto-isi pas reboot
Karena menu lu baca dari `/root/ip`, bikin biar tiap reboot auto ke-isi:
````
cat > /root/fix-ip.sh <<'EOF'
#!/bin/bash
IP=$(hostname -I | awk '{print $1}')
[ -z "$IP" ] && IP=$(curl -s --max-time 5 169.254.169.254/latest/meta-data/public-ipv4)
echo "$IP" > /root/ip
EOF
chmod +x /root/fix-ip.sh
crontab -l | grep -v fix-ip; (crontab -l 2>/dev/null; echo "@reboot /root/fix-ip.sh") | crontab -
/root/fix-ip.sh
cat /root/ip
````
### 4. Biar IP Public AWS gak ganti-ganti (ini paling penting)
Buka AWS Console:
`EC2 > Elastic IPs > Allocate Elastic IP address > Allocate`
Terus `Actions > Associate Elastic IP > pilih Instance lu > Associate`

Kalau udah pake Elastic IP, mau reboot 100x pun IP lu tetep sama, gak ilang.

*Terakhir, reboot buat ngetes:*
````
reboot
````
Tunggu 1 menit, login lagi, terus cek:
````
cat /etc/resolv.conf
cat /root/ip
ping -c 2 raw.githubusercontent.com
ss -tulpn | grep 53
````
Kalau `ping github` udah bisa dan `cat /root/ip` ada isinya, berarti udah sembuh permanen.

Lu udah pake Elastic IP belum bos? Kalau belum, itu yang bikin IP lu ilang tiap reboot, bukan cuma DNS-nya.

Nah internetnya nyambung bos, ping-nya bagus `0% packet loss`.

Yang error itu `curl` ke AWS-nya yang lagi ngambek, tadi bisa, sekarang kosong. Itu penyakitnya AWS, kadang minta token.

*Coba cara yang lain, jangan pake yang AWS dulu, pake yang dari internet:*

Ketik ini:
````
curl -s https://ifconfig.me
````
Harusnya muncul `3.106.213.154` langsung, tanpa `> /root/ip` dulu.

Kalau masih kosong, coba yang ini:
````
curl -s https://api.ipify.org
````
Kalau masih kosong juga, coba yang ini:
````
wget -qO- ifconfig.me
````
Salah satu dari 3 itu pasti muncul IP-nya.

*Kalau udah muncul, baru isi file-nya pake yang berhasil itu.* Misal yang berhasil `api.ipify.org`, ketik:
````
curl -s https://api.ipify.org > /root/ip
cat /root/ip
````
Coba yang pertama dulu bos:
````
curl -s https://ifconfig.me
````
*BERES BOS!* 🎉

Itu udah bener `3.106.213.154`, cuma nempel aja sama `root@ubuntu:~#` nya karena gak ada enter di akhir. Gak masalah.

*Biar rapi, ketik ini biar ada enter-nya:*
````
echo "" >> /root/ip
cat /root/ip
````
Harusnya sekarang jadi rapi:
3.106.213.154
root@ubuntu:~#
*Terakhir, pasang biar otomatis bener lagi pas reboot:*
````
(crontab -l 2>/dev/null; echo "@reboot /root/fix-ip.sh") | crontab -
crontab -l
````
Harusnya muncul `@reboot /root/fix-ip.sh` nya.

Kalau udah, *VPS lu udah AMAN BUAT REBOOT* sekarang bos, gak bakal balik ke `172.26` lagi.

Mau lanjut reboot sekarang atau mau cek SSL-nya dulu?

Nah itu yang gua bilang tadi bos, itu tandanya *IP VPS lu udah ganti pas reboot, tapi file `/root/ip` nya masih IP lama.*

AWS itu kalau gak pake Elastic IP, tiap reboot IP Public-nya ganti. Jadi `3.106.213.154` yang tadi itu udah bukan IP lu lagi sekarang.

*Cara ngeceknya, ketik ini:*
````
echo "FILE:"; cat /root/ip; echo ""
echo "ASLI:"; curl -s https://ifconfig.me; echo ""
````
Kalau hasilnya beda, misal:
FILE: 3.106.213.154
ASLI: 13.210.xxx.xxx
Berarti bener, IP lu udah ganti.

### Cara atasinya - 2 menit beres:

*1. Update file-nya ke IP baru:*
````
curl -s https://ifconfig.me > /root/ip
cat /root/ip
````
*2. Karena lu pake SlowDNS, WAJIB update juga config SlowDNS-nya, kalau enggak SlowDNS-nya mati:*

Cek dulu file SlowDNS lu:
````
cat slowdns.env
````
Kalau di dalemnya masih ada IP lama `3.106.213.154`, ganti ke IP baru:
````
IPBARU=$(cat /root/ip)
sed -i "s/3.106.213.154/$IPBARU/g" slowdns.env
cat slowdns.env
````
*3. Restart SlowDNS-nya:*
````
systemctl restart slowdns.service
systemctl status slowdns.service
ss -ulnp | grep :53
````
Harusnya sekarang udah jalan lagi di IP baru.

### Biar gak keulang terus tiap reboot:

Ini cuma obat sementara bos, besok lu reboot lagi, IP ganti lagi, ngulang lagi.

*Solusi permanennya cuma 1: Pasang Elastic IP di AWS.*

1. AWS Console > EC2 > Elastic IPs > Allocate Elastic IP
2. Associate ke Instance VPS lu
3. Abis itu IP lu dikunci, gak akan ganti lagi walau reboot 100x

Kalau udah pasang Elastic IP, IP di `cat /root/ip` sama di `curl -s https://ifconfig.me` bakal sama terus.

Coba kirim ke gua bos hasil ini:
````
cat /root/ip
curl -s https://ifconfig.me
````

````
cat /root/ip
curl -s https://ifconfig.me; echo ""
ip -4 addr show | grep inet
````
Bedanya jauh gak IP lama sama IP baru lu?

Oke, uninstall total sampe bersih bos. Jangan cuma di-stop, nanti rule `NOTRACK` nya masih nyangkut di iptables, port 53 masih ke-kunci.

*Ketik urut dari atas ke bawah, jangan diloncat:*

### 1. Stop & Uninstall pake installernya

Di folder tempat lu install tadi, biasanya `/root/`:
````
cd /root
bash ./install.sh uninstall
````
Harusnya muncul:
`Service removed. The binary, configuration, and keys remain...`

Kalau file `install.sh` nya udah kehapus, langsung ke cara manual di bawah.

### 2. Cara Manual (kalau cara 1 gagal
````
systemctl stop slowdns.service
systemctl disable slowdns.service
rm -f /etc/systemd/system/slowdns.service
systemctl daemon-reload
````
### 3. Bersihin Rule NOTRACK yang nyangkut (INI PENTING)

Ini yang bikin IP lu ilang-ilangan tadi:
````
iptables -t raw -S | grep slowdns
iptables -t raw -D PREROUTING -i eth0 -d 3.106.213.154 -p udp --dport 53 -m comment --comment "slowdns-direct" -j CT --notrack 2>/dev/null; true
iptables -t raw -F
ip6tables -t raw -F
iptables -t raw -S
````
Yang terakhir harusnya kosong, gak ada lagi `slowdns-direct`.

### 4. Hapus semua file SlowDNS
````
rm -f /root/slowdns.env /root/server.key /root/server.pub /root/slowdns /root/slowdns-amd64 /root/install.sh
rm -rf /root/.slowdns-install.*
ls -la /root/ | grep slowdns
````
Yang terakhir harusnya kosong juga.

### 5. Balikin DNS normal biar internet jalan lagi

Karena SlowDNS udah dihapus, balikin `systemd-resolved`:
````
systemctl enable systemd-resolved
systemctl start systemd-resolved
ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
cat /etc/resolv.conf
ping -c 2 ifconfig.me
````
### 6. Cek udah bersih total belum
````
systemctl status slowdns.service
ss -ulnp | grep :53
ps aux | grep slowdns
````
- `systemctl status` harusnya: `could not be found`
- `ss -ulnp` harusnya: kosong / cuma ada `systemd-resolved` di `127.0.0.53`
- `ps aux` harusnya: gak ada slowdns yang jalan

Udah beres bos. Mau gua bantu install ulang SlowDNS yang bener pake `auto` + Elastic IP, atau mau pindah ke DNS biasa aja biar gak ribet lagi?
Mantap, service-nya udah kehapus bos. ✅

Tinggal bersihin sisa-sisanya, biar port 53 bener-bener bebas dan IP gak ilang lagi.

*Lanjut ketik ini, urut:*

### 1. Bersihin rule NOTRACK yang masih nyangkut
````
iptables -t raw -F
ip6tables -t raw -F
iptables -t raw -S
````
Harusnya yang terakhir kosong, gak ada `slowdns-direct` lagi.

### 2. Hapus file sisa SlowDNS
````
rm -f /root/slowdns.env /root/server.key /root/server.pub /root/slowdns /root/slowdns-amd64 /root/install.sh
rm -rf /root/.slowdns-install.*
ls /root/
````
### 3. Balikin DNS normal
````
systemctl enable systemd-resolved
systemctl start systemd-resolved
ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
cat /etc/resolv.conf
````
### 4. Cek terakhir, udah bersih belum
```
systemctl status slowdns.service
ss -ulnp | grep :53
cat /root/ip
curl -s https://ifconfig.me; echo ""
````
Harusnya:
- `status` = `could not be found`
- `ss` = cuma ada `127.0.0.53` punya `systemd-resolved`, gak ada lagi yang di port 53 IP public
- `cat /root/ip` = sama kayak `curl ifconfig.me`

Kirim ke gua hasil nomer 4 bos, udah sama belum IP di `/root/ip` sama IP aslinya?

Kalau udah sama, VPS lu udah bersih total dari SlowDNS. Mau install ulang yang bener atau udahan dulu?
