# Andmete Eksportimine ja Turvaline Edastamine

Juhend MySQL/MariaDB andmebaasi dump-faili loomiseks, turvaliseks edastamiseks ja importimiseks Alpine Linux süsteemides.

## 📋 Ülesande Kirjeldus

Selles projektis tuleb:
1. Luua dump-fail andmebaasist, mis sisaldab vähemalt 2M rida
2. Edastada fail turvaliselt teise Alpine Linux serverisse
3. Importida dump-fail sihtsüsteemis
4. Dokumenteerida kogu protsess ekraanipiltidega

**Oluline:** Dump-fail ei tohi olla avalikult kättesaadav mitte ühegi sekundi jooksul!

---

## 🎯 Võimalused NAT-i Taga Olevate Masinate Jaoks

### ✅ Võimalus 1: Tailscale + SCP (SOOVITATUD)

**Eelised:**
- NAT-i läbimine automaatne
- Krüpteeritud ühendus vaikimisi
- Lihtne seadistada
- Töötab ka virtuaalmasinate vahel

#### Samm 1: Tailscale'i Seadistamine

Installi ja seadista Tailscale mõlemas Alpine Linux masinas:

```bash
# Installi Tailscale
apk add tailscale

# Käivita Tailscale daemon
rc-service tailscale start
rc-update add tailscale

# Autenteeri Tailscale'iga (avaneb brauser)
tailscale up

# Kontrolli ühendust ja saa oma IP
tailscale status
tailscale ip -4
```

#### Samm 2: Dump-faili Loomine (Saatja Masinas)

**Docker konteineris töötava andmebaasi puhul:**

```bash
# Loo kataloog dump-failidele
mkdir -p /root/db_dumps
cd /root/db_dumps

# VARIANT 1: Kui MySQL port on forward'itud hostile (nt -p 3306:3306)
mysqldump -h 127.0.0.1 -P 3306 -u root -p DB_NIMI > dump_$(date +%Y%m%d_%H%M%S).sql

# VARIANT 2: Kasutades docker exec
# Kontrolli töötavaid Docker konteinereid
docker ps

# Loo andmebaasi dump (asenda KONTEINER_NIMI ja DB_NIMI)
docker exec KONTEINER_NIMI mysqldump -u root -p DB_NIMI > dump_$(date +%Y%m%d_%H%M%S).sql

# VÕI kui parool on keskkonnamuutujas:
docker exec KONTEINER_NIMI mysqldump -u root -pPAROOL DB_NIMI > dump_$(date +%Y%m%d_%H%M%S).sql

# VÕI kui tahad kõik andmebaasid dumpida:
docker exec KONTEINER_NIMI mysqldump -u root -p --all-databases > dump_all_$(date +%Y%m%d_%H%M%S).sql

# Kompresseeri dump
gzip dump_*.sql

# VALIKULINE: Lisaturvalisus GPG krüpteerimisega
gpg -c dump_*.sql.gz
# Märkus: SCP kasutab juba SSH krüpteerimist, seega see on valikuline
```

**Tavapärase (mitte-Docker) andmebaasi puhul:**

```bash
# Loo andmebaasi dump (asenda DB_NIMI)
mysqldump -u root -p tahvel > dump_$(date +%Y%m%d_%H%M%S).sql

# Kompresseeri dump
gzip dump_*.sql
```

#### Samm 3: Faili Edastamine (Saatja Masinas)

```bash
# Küsi teise õppija Tailscale IP-aadress
# Teine õppija käivitab: tailscale ip -4

# Edasta fail SCP kaudu (asenda 100.x.y.z tegeliku IP-ga)
scp dump_*.sql.gz root@100.x.y.z:/root/

# Kui krüpteerisid GPG-ga:
scp dump_*.sql.gz.gpg root@100.x.y.z:/root/
```

#### Samm 4: Faili Importimine (Vastuvõtja Masinas)

```bash
# Kontrolli faili
ls -lh /root/dump_*

# Dekompresseeri
gunzip dump_*.sql.gz

# Kui kasutasid GPG krüpteerimist:
gpg -d dump_*.sql.gz.gpg > dump_*.sql.gz
gunzip dump_*.sql.gz

# Loo uus andmebaas
mysql -u root -p -e "CREATE DATABASE uus_andmebaas;"

# Impordi dump
mysql -u root -p uus_andmebaas < dump_*.sql

# Kontrolli importi
mysql -u root -p uus_andmebaas -e "SHOW TABLES;"
mysql -u root -p uus_andmebaas -e "SELECT COUNT(*) FROM tabeli_nimi;"

# Kustuta ajutised failid
rm -f /root/dump_*
```

---

### ✅ Võimalus 2: SSH Tunnel + Netcat

**Eelised:**
- Ei vaja lisatarkvara
- Kiire otseühendus
- Töötab standardsete tööriistadega

#### Dump Loomine (Saatja)

**Docker konteineris:**

```bash
mkdir -p /root/db_dumps
cd /root/db_dumps

# Loo dump otse kompresseeritult
docker exec KONTEINER_NIMI mysqldump -u root -p DB_NIMI | gzip > dump_$(date +%Y%m%d_%H%M%S).sql.gz
```

**Tavapärane:**

```bash
mkdir -p /root/db_dumps
cd /root/db_dumps
mysqldump -u root -p DB_NIMI | gzip > dump_$(date +%Y%m%d_%H%M%S).sql.gz
```

#### Faili Edastamine

```bash
# VASTUVÕTJA masinas - alusta kuulamist pordil 9999
nc -l -p 9999 > received_dump.sql.gz

# SAATJA masinas - saada fail (asenda VASTUVÕTJA_IP)
cat dump_*.sql.gz | nc VASTUVÕTJA_TAILSCALE_IP 9999
```

#### Import (Vastuvõtja)

```bash
gunzip received_dump.sql.gz
mysql -u root -p -e "CREATE DATABASE uus_andmebaas;"
mysql -u root -p uus_andmebaas < received_dump.sql
rm -f received_dump.sql
```

---

### ✅ Võimalus 3: SFTP + Krüpteeritud Fail

**Eelised:**
- Võimaldab faili ajutist säilitamist
- Võimalik kasutada GUI kliente (FileZilla)
- Täielik kontroll faili üle

#### Dump + Krüpteerimine (Saatja)

**Docker konteineris:**

```bash
# Loo dump
docker exec KONTEINER_NIMI mysqldump -u root -p DB_NIMI | gzip > dump.sql.gz

# Krüpteeri GPG-ga
gpg -c dump.sql.gz  # Sisesta tugev parool!

# Kustuta krüpteerimata fail
rm dump.sql.gz
```

**Tavapärane:**

```bash
# Loo dump
mysqldump -u root -p DB_NIMI | gzip > dump.sql.gz

# Krüpteeri GPG-ga
gpg -c dump.sql.gz  # Sisesta tugev parool!

# Kustuta krüpteerimata fail
rm dump.sql.gz
```

#### SFTP Server Seadistamine (Vastuvõtja)

```bash
# Veendu, et SSH server töötab
rc-service sshd status

# Kui ei tööta, käivita:
rc-service sshd start
rc-update add sshd
```

#### Faili Üleslaadimine (Saatja)

```bash
# Kasuta SFTP-d
sftp root@VASTUVÕTJA_TAILSCALE_IP

# SFTP sessionis:
put dump.sql.gz.gpg
exit
```

#### Dekrüpteerimine + Import (Vastuvõtja)

```bash
# Dekrüpteeri
gpg -d dump.sql.gz.gpg > dump.sql.gz

# Dekompresseeri
gunzip dump.sql.gz

# Impordi
mysql -u root -p -e "CREATE DATABASE uus_andmebaas;"
mysql -u root -p uus_andmebaas < dump.sql

# Puhasta
rm dump.sql.gz.gpg dump.sql
```

---

## 📸 Dokumenteerimine

### Nõutavad Ekraanipildid

#### 1. Saatja Masinas

```bash
# Näita hostname'i
cat /etc/hostname  # Peab olema: eesnimi-perenimi-alpine

# Näita Docker konteinereid
docker ps

# Dump'i loomine (Dockerist)
docker exec KONTEINER_NIMI mysqldump -u root -p DB_NIMI > dump_*.sql

# Faili suurus
ls -lh dump_*
```

#### 2. Edastuse Käigus

```bash
# Näita edastamise käsku
scp dump_*.sql.gz root@100.x.y.z:/root/
# VÕI
nc -l -p 9999 > received_dump.sql.gz
```

#### 3. Vastuvõtja Masinas

```bash
# Näita hostname'i
cat /etc/hostname  # Peab olema: eesnimi-perenimi-alpine

# Saabunud fail
ls -lh dump_*

# Importimise käsk
mysql -u root -p uus_andmebaas < dump_*.sql

# Tõendus - näita andmeid
mysql -u root -p uus_andmebaas -e "SELECT COUNT(*) FROM tabeli_nimi;"
mysql -u root -p uus_andmebaas -e "SHOW TABLES;"
```

---

## 🔒 Turvalisuse Kontroll-loend

- [ ] Dump-fail on krüpteeritud või liigub krüpteeritud kanaliga
- [ ] Fail ei ole avalikus asukohas
- [ ] Kasutatakse turvalist ühendusmeetodit (Tailscale, SSH, SFTP)
- [ ] Ajutised failid kustutatakse pärast importi
- [ ] Mõlemas masinas on õige hostname (eesnimi-perenimi-alpine)
- [ ] Ekraanipiltidel on nähtav kogu protsess
- [ ] Import on kontrollitud (päringud töötavad)

---

## 🖥️ Virtuaalmasinate Erijuhud

### VirtualBox / VMware / Hyper-V

Kui Alpine Linux masinad asuvad virtuaalmasinas:

**Variant A: Tailscale (Soovitatud)**
- Töötab sõltumata VM võrguseadistusest
- Ei vaja NAT-i konfiguratsiooni muutmist
- Töötab isegi kui VM-id on erinevates võrkudes

**Variant B: Bridged Network**
Kui VM-id on sama hosti peal:
1. Muuda VM võrguadapter "Bridged" režiimile
2. Mõlemad VM-id saavad lokaalse võrgu IP-d
3. Võid kasutada otse neid IP-sid ilma Tailscale'ita

**Variant C: Internal Network**
VirtualBoxis:
1. Loo Internal Network (nt "alpine-network")
2. Lisa mõlemad VM-id samasse võrku
3. Seadista staatilised IP-d mõlemas masinas
4. Kasuta neid IP-sid edastamiseks

---

## 📊 Hindamiskriteeriumid

| Kriteerium | Kirjeldus |
|------------|-----------|
| **Turvalisus** | Dump ei olnud avalikus asukohas ja liikus krüpteeritult |
| **Tõendatavus** | Ekraanipiltidelt on nähtav kogu protsess |
| **Hostname** | Igal ekraanipildil on näha masina hostname (eesnimi-perenimi-alpine) |
| **Funktsinaalsus** | Import õnnestus ja andmed on kättesaadavad |
| **Puhastus** | Ajutised failid on kustutatud |

---

## 🛠️ Kasulikud Käsud

### Hostname Kontroll/Muutmine

```bash
# Vaata praegust hostname'i
cat /etc/hostname

# Muuda hostname'i (kui vaja)
echo "eesnimi-perenimi-alpine" > /etc/hostname
hostname -F /etc/hostname
```

### MySQL/MariaDB Abikäsud

**Docker konteineris:**

```bash
# Andmebaaside loetelu
docker exec KONTEINER_NIMI mysql -u root -p -e "SHOW DATABASES;"

# Tabelite arv
docker exec KONTEINER_NIMI mysql -u root -p DB_NIMI -e "SHOW TABLES;" | wc -l

# Ridade arv tabelis
docker exec KONTEINER_NIMI mysql -u root -p DB_NIMI -e "SELECT COUNT(*) FROM tabeli_nimi;"

# Dump-faili suurus kontrollimine
ls -lh dump_*.sql.gz
du -h dump_*.sql.gz

# Sisene Docker konteinerisse interaktiivselt
docker exec -it KONTEINER_NIMI bash
# või
docker exec -it KONTEINER_NIMI mysql -u root -p
```

**Tavapärane:**

```bash
# Andmebaaside loetelu
mysql -u root -p -e "SHOW DATABASES;"

# Tabelite arv
mysql -u root -p DB_NIMI -e "SHOW TABLES;" | wc -l

# Ridade arv tabelis
mysql -u root -p DB_NIMI -e "SELECT COUNT(*) FROM tabeli_nimi;"

# Dump-faili suurus kontrollimine
ls -lh dump_*.sql.gz
du -h dump_*.sql.gz
```

### Võrguühenduse Testimine

```bash
# Ping Tailscale IP-d
ping 100.x.y.z

# Kontrolli SSH ühendust
ssh root@100.x.y.z

# Kontrolli avatud porte
netstat -tuln | grep LISTEN
```

---

## ❓ Tõrkeotsing

### Probleem: Tailscale ei autenteeri

```bash
# Kontrolli Tailscale staatust
tailscale status

# Taaskäivita Tailscale
rc-service tailscale restart

# Proovi uuesti
tailscale up
```

### Probleem: SCP ühendus ebaõnnestub

```bash
# Kontrolli SSH serverit vastuvõtjas
ssh root@VASTUVÕTJA_IP

# Kui ei tööta, käivita SSH server
rc-service sshd start
```

### Probleem: Import annab vea

```bash
# Kontrolli dump-faili terviklikkust
gunzip -t dump_*.sql.gz

# Vaata dump-faili algust
zcat dump_*.sql.gz | head -20

# Impordi verbose režiimis
mysql -u root -p -v uus_andmebaas < dump_*.sql
```

---

## 📚 Viited

- [Tailscale Documentation](https://tailscale.com/kb/)
- [MySQL Dump Documentation](https://dev.mysql.com/doc/refman/8.0/en/mysqldump.html)
- [GPG Encryption Guide](https://www.gnupg.org/gph/en/manual.html)
- [Alpine Linux Wiki](https://wiki.alpinelinux.org/)

---

## 👥 Autorid

- **Projekt:** Andmete Eksportimine (Paaristöö)
- **Kuupäev:** Oktoober 2025

---

## 📝 Litsents

See juhend on loodud hariduslikel eesmärkidel.
