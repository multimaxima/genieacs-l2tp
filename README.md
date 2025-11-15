# Genie ACS

**UNTUK DI LOKAL TIDAK PERLU INSTALL L2TP, BISA LANGSUNG KE PROSES INSTALL GENIEACS**

INSTALL L2TP untuk TUNNEL GENIEACS
```
sudo apt update && apt upgrade -y
apt install curl
uname -r
```
cek pastikan TIDAK ada mengandung kata cloud

```
wget https://raw.githubusercontent.com/multimaxima/genieacs-l2tp/refs/heads/main/vpnsetup.sh
chmod +x vpnsetup.sh
./vpnsetup.sh
```

```
wget -O add_vpn_user.sh https://raw.githubusercontent.com/hwdsl2/setup-ipsec-vpn/master/extras/add_vpn_user.sh
bash add_vpn_user.sh 'username' 'password'
```
Tambahkan L2TP Client di Mikrotik anda.

Selanjutnya cek di VPS ppp berapa ada terhubung
```
ip route list
```
misal responsenya seperti ini
```
192.168.42.10 dev ppp0 proto kernel scope link src 192.168.42.1
```
Artinya Mikrotik anda terhubung dengan ppp0. Maka tambahkan route di VPS
```
ip route add 10.0.0.0/24 dev ppp0
```
**10.0.0.0/24** adalah ip lokal modem/onu.


=========================================================================

**INSTALL GENIEACS**
```
wget https://raw.githubusercontent.com/multimaxima/genieacs-l2tp/refs/heads/main/genie.sh
chmod +x genie.sh
./genie.sh
```

Selanjutnya silahkan buka URL genie acs IP:3000.
Lanjutkan dengan update Config, Provisioning dan Virtual Parameter

```
mkdir /root/db
cd /root/db
wget https://github.com/multimaxima/genieacs-l2tp/raw/refs/heads/main/config.bson
wget https://github.com/multimaxima/genieacs-l2tp/raw/refs/heads/main/config.metadata.json
wget https://github.com/multimaxima/genieacs-l2tp/raw/refs/heads/main/presets.bson
wget https://github.com/multimaxima/genieacs-l2tp/raw/refs/heads/main/presets.metadata.json
wget https://github.com/multimaxima/genieacs-l2tp/raw/refs/heads/main/provisions.bson
wget https://github.com/multimaxima/genieacs-l2tp/raw/refs/heads/main/provisions.metadata.json
wget https://github.com/multimaxima/genieacs-l2tp/raw/refs/heads/main/virtualParameters.bson
wget https://github.com/multimaxima/genieacs-l2tp/raw/refs/heads/main/virtualParameters.metadata.json
mongorestore --db genieacs --drop /root/db
systemctl start genieacs-{cwmp,ui,nbi}
```

UPDATE
add ip route otomatis ketika l2tp terhubung, caranya buat ip static pada tiap user

edit
```
sudo nano /etc/ppp/chap-secrets
```
```
"bery1" l2tpd "12345678" "192.168.42.10"
"bery2" l2tpd "12345678" "192.168.42.11"
"bery3" l2tpd "12345678" "192.168.42.12"
```

formatnya
```
"username" l2tpd "password" "ip local static yang diberikan ke user"
```

tambahkan /etc/ppp/ip-up.d/add-routes
```
sudo nano /etc/ppp/ip-up.d/add-routes
```
masukkan script dibawah ini, rubah ip tr069 modem kalian.
```
#!/bin/bash
# Auto add route berdasarkan IP remote user L2TP
# Environment: $IPREMOTE, $IFNAME, $PEERNAME
# Letakkan di /etc/ppp/ip-up.d/ misalnya

declare -A ROUTES

# Mapping: [IPREMOTE]="net1 net2 ..."
ROUTES["192.168.42.10"]="192.168.120.0/22 10.1.0.0/22"        # bery1
ROUTES["192.168.42.11"]="172.16.4.0/22"                       # bery2
ROUTES["192.168.42.12"]="172.16.12.0/22 172.16.20.0/24"       # bery3

# Tambahkan route jika ada mapping
if [[ -n "${ROUTES[$IPREMOTE]}" ]]; then
  for NET in ${ROUTES[$IPREMOTE]}; do
    logger -t l2tp-route "Adding route $NET via $IFNAME for $IPREMOTE"
    ip route replace "$NET" dev "$IFNAME"
  done
fi

exit 0

```
kasih akses
```
sudo chmod +x /etc/ppp/ip-up.d/add-routes
service xl2tpd restart
```

melihat ipsec ada disini
```
cat /etc/ipsec.secrets
```
