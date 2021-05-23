# Uvod

Ovaj fajl predstavlja uputstvo kako namestiti virtuelnu mrežu u cilju testiranja Streaming Telemety na Cisco CSR1000V ruteru. Mreža je mala, ali sasvim dovoljna za testiranja telemetrije. Sastoji se od: 
- Virtuelnog računara (Linux Kubuntu 18.04 LTS) koji služi za prihvatanje telemetrijskih podataka sa rutera i njihovo iscrtavanje, kao i za generisanje saobraćaja kako bi podaci dobijeni telemetrijom bili smisleniji.
- Virtuelnog rutera (Cisco CSR1000V na kome je Cisco IOS XE 16.12.03 operativni sistem) koji služi da omogući virtuelnom računaru pristup internetu, kao i da šalje telemetrijske podatke računaru.

Primer mreže se može videti na slici ispod.

![mreza-interfejsi](https://user-images.githubusercontent.com/18577840/119224063-ff23e700-bafc-11eb-889c-e7d162bb8f57.png)

Za kreiranje i rad sa virtuelnim mašinama koristi se [VMWare Workstation Player](https://www.vmware.com/latam/products/workstation-player/workstation-player-evaluation.html).  
:warning: **Image za Cisco CSR1000V nije podržan u ostalim alatima za rad sa virtuelnim mašinama (kao što su VirtualBox, KVM, GNS).**
Minimalni hardverski zahtevi su: 
- 8GB RAM.
- 4 logička procesora.
- 30GB slobodnog mesta.

# Konfiguracija virtuelnog računara
U ovom poglavlju je dato detaljno uputstvo kako spremiti virtuelni računar. 
- Prvo poglavlje opisuje kako kreirati virtuelni mašinu i kako instalirati operativni sistem.
- Drugo poglavlje opisuje kako namestiti mrežni interfejs i koliko je potrebno resursa dodeliti virtuelnoj mašini.
- Treće poglavlje opisuje kako instalirati alate (Telegraf, InfluxDB, Grafana) koji su potrebni za prikupljanje telemetrijskih podataka.
 
## Kreiranje virtuelne mašine i instalacija operativnog sistema
Korišćeni operativni sistem je [Linux Kubuntu 18.04.5](https://kubuntu.org/alternative-downloads).
U nastavku su dati koraci kako kreirati virtuelnu mašinu:
1. Na datom linku potrebno je skinuti operativni sistem. Odabrana verzija treba da bude: **kubuntu-18.04.5-desktop-amd64.iso**
2. Otvorite VMWare Worksation Player i odaberite opciju **Create a New Virtual Machine**
![Ubuntu1](https://user-images.githubusercontent.com/18577840/119225015-9854fc80-bb01-11eb-97b6-f17a7e6ada4a.PNG)
3. Potrebno je odabrati opciju **Installer disc image file (iso)** i odabrati putanju do prethodno preuzetog operativnog sistema.
![Ubuntu2](https://user-images.githubusercontent.com/18577840/119225077-fbdf2a00-bb01-11eb-9eee-8d5a48d26f04.PNG)
4. Klikom na dugme **next** pojavljuje se novi prozor gde je potrebno uneti ima računara kao i login kredencijale. Radi jednostavnosti tokom celog uputstva ćemo koristiti reč **cisco** i kao username i kao password.
![Ubuntu3](https://user-images.githubusercontent.com/18577840/119225128-42cd1f80-bb02-11eb-8cf1-9806f8346bf9.PNG)
5. Na nekoliko narednih dijaloga nije potrebno ništa menjati, te izberite opciju **next** i na kraju **finsih**.
6. Sada kreće instalacija Kubuntu operativnog sistema. Ovaj deo neće biti detaljno opisan, pošto je dovoljno proći kroz instalaciju koristeći samo podrazumevana podešavanja.
7. Isključite virtuelnu mašinu.

## Konfiguracija virtuelne mašine
Potrebno je prilagoditi mrežni interfejs, kao i količinu RAM memorije i broja logičkih procesora datih virtuelnoj mašini.
1. Odaberite opciju **Edit virtual machine settings**.
![Ubuntu4](https://user-images.githubusercontent.com/18577840/119225605-e9b2bb00-bb04-11eb-9c0d-8e4dcecb29b5.PNG)
2. Otvoriće se novi prozor, gde u meniju sa strane možete odabrati **Processors** i **Memory** kako biste promenili broj logičkih procesora, kao i količinu RAM memorije u zavisnosti od hardvera na kome radite. Minimum je potrebno postaviti 3GB RAM i 2 CPU.
3. Zatim odaberite opciju ** Network Adapter** (takođe se nalazi u meniju sa strane) i izaberite oopciju **Custom: Specific Virtual Network** i iz padajuće liste odaberite **VMNet0**.
![Ubuntu5](https://user-images.githubusercontent.com/18577840/119225775-3945b680-bb06-11eb-9498-8d4990579476.PNG)
4. Potrebno je namestiti i opciju **ethernet.virtualdev**. Ovu opciju nije moguće pronaći u GUI-u, te je potrebno ručno modifikovati konfiguracioni fajl od napravljene virtuelne mašine. Locirajte gde se nalazi mašina na vašem računaru (obično je u: **Documents\Virtual Machines\Ubuntu 64-bit**). Konfiguracioni fajl ima ekstenziju .vmx. Otvorite ga u tekstualnom editoru i promenite opciju: **ethernet0.virtualDev = "vmxnet3"**.
5. U ovom koraku želimo da namestimo IP adresu računara, gateway, kao i DNS. Potrebno je uključiti virtuelnu mašinu.
   - Treba da pronađete ime mrežnog interfejsa. U ovom slučaju je ens160 (ali možda kod vas bude drugačije). Ovo možete postići tako što ćete pročitati izlaz komande: 
     ```ifconfig```
   - Kada je pronađeno ime interfejsa, potrebno je promeniti **netplan** konfiguracioni fajl. Putanja do fajla je: ```/etc/netplan/01-network-manager-all.yaml```.
     Potrebno je promeniti sadržaj fajla tako da izgleda ovako:
     ```
     network:
     ethernets:
     ens160:
         addresses: [10.0.0.2/24]
         gateway4: 10.0.0.1
         dhcp4: false
         optional: false
         nameservers:
         addresses: [8.8.8.8,8.8.4.4]
     version: 2
     renderer: NetworkManager

     ```
  - Potom izvršite komandu: ```sudo netplan apply```

## Instalacija dodatnih alata
Potrebno je instalirati tri alata:
- Telegraf - prikupljanje telemetrijskih podataka.
- InfluxDB - čuvanje prikupljenih podataka.
- Grafana - iscrtavanje prikupljenih podataka.

1. Uključite virtuelnu mašinu i otvorite konzolu. Svi alati su instalirani preko komandne linije.
2. Instalacija InfluxDB-a. 
```
# Trust the Influx GPG key
wget -qO- https://repos.influxdata.com/influxdb.key | sudo apt-key add -
# Add the Influx repositories to apt
source /etc/lsb-release
echo "deb https://repos.influxdata.com/${DISTRIB_ID,,} ${DISTRIB_CODENAME} stable" | sudo tee /etc/apt/sources.list.d/influxdb.list

# Update the repositories, and install influx
sudo apt-get update && sudo apt-get install influxdb

# Enable influx, and start it
sudo systemctl unmask influxdb.service
sudo systemctl start influxdb
```
Možete proveriti da li InfluxDB radi tako što ćete u konzoli ukucati **influx**.

2. Instalacija Telegrafa.
```
# Update the repositories, and install telegraf
sudo apt-get update && sudo apt-get install telegraf

# Generate the telegraf configuration with input from Cisco
# devices, and output to Influxdb
sudo telegraf --output-filter influxdb --input-filter cisco_telemetry_mdt config | sudo tee /etc/telegraf/telegraf.conf

# Enable and start the service
sudo systemctl enable telegraf
sudo systemctl start telegraf
```
Možete proveriti da li Telegraf radi tako što ćete proveriti da li otvoren port 57000 na virtuelnoj mašini. 
To možete uraditi komandom: ```sudo ss -plant```.

3. Instalacija Grafane.
```
sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
sudo apt-get update
sudo apt-get install grafana

sudo systemctl daemon-reload
sudo systemctl start grafana-server
```
Možete proveriti da li Grafana radi tako što ćete iz internet pretraživača otvoriti URL: **http://localhost:3000/** i uneti kredencijale **admin/admin**.

:warning: **Ako restartujete virtuelnu mašinu potrebno je ponovo pokrenuti sva tri programa/servisa. To možete uraditi ponovnim unošenjem 'systemctl' komandi.**

4. (Opciono) Instalacija Cisco YANG Suite-a
Ovaj alat sluzi za istrazivanje i proveru podataka koji se mogu primiti sa Cisco uređaja. Preporuka je da namestite ovaj alata, ali on nije neophodan. Uputstvo kako namestiti ovaj alat se nalazi na [GitHub repou](https://github.com/CiscoDevNet/yangsuite/).

# Konfiguracija virtuelnog rutera
U ovo poglavlju je dato detaljno uputstvo kako namestiti virutelni ruter.
Za dobijanje image potrebno je obratiti se profesoru.

## Priprema image-a
Image dolazi u qcow2 formatu (prilagođen za QEMU), koji nije pogodan za učitavanje u VMware Workstation Player.
Image je potrebno konvertovati u prikladan (vmdk) format. Za konverziju je korišćen [**qemu-img.exe**](https://cloudbase.it/qemu-img-windows/).
Konverziju je moguće uraditi putem pokretanja komande: 
```
qemu-img convert -f qcow2 -O vmdk -o subformat=streamOptimized source_qcow_image_path destination_path_to_vmdk
```

## Kreiranje virtuelne mašine
Sledeći korak je da napravimo virtuelnu mašinu koristeći VMware Workstation Player. Fajl koji smo dobili konverzijom predstavlja samo virtuelni disk, te je potrebno napraviti novu virtuelnu mašinu, a kovertovani fajl je potrebno postaviti kao disk od virtuelne mašine. Koraci:
1. Pokrenuti VMware Workstation Player i odabrati opciju **Create a New Virtual Machine**.
2. U novom prozoru odabrati opciju **I will install operating system later**.
![Ruter1](https://user-images.githubusercontent.com/18577840/119234986-aa01c880-bb30-11eb-8050-a5a1aca3272b.PNG)
3. U novom prozoru za verziju i tip operativnog sistema je potrebno izabrati **Other**.
![Ruter2](https://user-images.githubusercontent.com/18577840/119235044-d584b300-bb30-11eb-805c-6347c08e375d.PNG)
4. U novom prozoru unesite željeno ime virtuelne mašine (npr. csr1000v).
5. Kroz ostatak kreiranja virtuelne mašine samo koristite opciju **next** i **finish**.
6. Podesite RAM memoriju na 3GB, a za broj virtuelnih procesore staviti 3 (možete probati i sa manjim brojem). Ako ne znate kako ovo da uradite, pogledajte pogavlja o pripremi virtuelnog računara.
7. Trenutno smo kreirali novu virtuelnu mašinu, ali tek je potrebno postaviti njen virtuelni disk. Idite u opcije i u meniju sa strane odaberite opciju **Hard Disk**, a zatim kliknuti **Remove**.
![Ruter3](https://user-images.githubusercontent.com/18577840/119235221-b33f6500-bb31-11eb-93bc-c1de4312f32d.PNG)
8. Sada je potrebno postaviti novi disk. Idite na opciju **Add** i odaberite opciju **Hard Disk**.
9. U narednom prozoru obeležite tip **IDE**.
10. U narednom prozoru odaberite opciju **Use an existing virtual disk**.
![Ruter4](https://user-images.githubusercontent.com/18577840/119235285-06b1b300-bb32-11eb-97a3-fba9de91ce0d.PNG)
11. U narednom prozoru uneti putanju do fajla koji ste dobili konverzijom.
12. Virtuelnoj mašini je potrebno dodeliti i 4 mrežna adaptera.
    - Mrežni adapter **Network Adapter** postaviti da koristi NAT i obeležiti opciju **Connect at power on**.
      ![Ruter5](https://user-images.githubusercontent.com/18577840/119235731-42e61300-bb34-11eb-83af-ff561528cf04.PNG)

    - Mrežni adapter **Network Adapter 4** postaviti da koristi VMnet0 i obeležiti opciju **Connect at power on**.
      ![Ruter6](https://user-images.githubusercontent.com/18577840/119235733-45486d00-bb34-11eb-9649-d054ada719c8.PNG)
13. Postaviti **vmxnet3** za sve mrežne adaptere. Ovaj korak je već prethodno opisan u konfiguraciji virtuelnog računara.
14. Pokrenite virtuelnu mašinu kako biste se uverili da ona radi.
15. Kada pokrenete virtuelnu mašinu odaberite opciju **Golden image**. 
![Ruter7](https://user-images.githubusercontent.com/18577840/119235841-a708d700-bb34-11eb-8e55-4deee7f2aadd.PNG)

## Konfiguracija mrežnih interfejsa rutera
U prethodnom koraku smo definisali četiri mrežna adaptera, ali ćemo koristiti samo dva. Mrežni adapter 4 ćemo koristiti za povezivanje sa virtuelnim račuarom, a mrežni adapter 1 ćemo koristiti za povezivanje sa NAT-om. Komanda koja će nam biti korisna za proveru statusa mrežnih interfejsa je:
```
show ip interface brief
```

### Konfiguracija mrežnog adaptera 4.
```
enable
configure terminal
interface GigabitEthernet 4
no shutdown
ip address 10.0.0.1 255.255.255.0
exit
copy running-config startup-config
```

Sada bi trebalo da smo u mogućnosti da pingujemo virtuelni računar. Proveriti da li sledeća komanda radi:
```
ping 10.0.0.2
```

### Konfiguracija mrežnog adaptera 1.
1. Defisanje DHCP-a na interfejsu:
```
enable
configure terminal
interface GigabitEthernet 1
no shutdown
ip address dhcp
exit
copy running-config startup-config
```

2. Definisanje NAT-a na interfejsu:
```
enable
configure terminal
interface GigabitEthernet 1
ip nat outside
end
configure terminal
interface GigabitEthernet 4
ip nat inside
end
configure terminal
ip nat inside source list 1 interface GigabitEthernet 1 overload
access-list 1 permit host 10.0.0.2
end
copy running-config startup-config
```

## Konfiguracija streaming telemetry
U ovom poglavlju biće date komande kako konfigurisati ruter da šalje telemetrijske podatke. 
Ruter može da šalje razne podatke (YANG Suite name može pomoći da istražimo koje sve podatke možemo da šaljemo), ali mi ćemo se fokusirati na tri tipa podataka: 
iskorišćenje procesora, iskorišćenje memorije i mrežni saobraćaj. 
1. Prvi korak je da se konfiguriše NETCONF-YANG:
```
enable
configure terminal
username cisco privilege 15 password cisco
aaa new-model
aaa authentication login default local
aaa authorization exec default local
netconf-yang
netconf-yang feature candidate-datastore
exit
copy running-config startup-config
```

2. Proveriti output sledeće komande: 
```
show platform software yang-management process
```
Svi procesi moraju da imaju status **Running**. Ukoliko to nije slučaj, slanje telemetrijskih podataka neće raditi.

3. Sledeći korak je da definišemo koje podatke želimo da ruter šalje:
   - Slanje telemetrije o iskorišćenju CPU-a:
     ```
     enable
     configure terminal
     telemetry ietf subscription 1
     encoding encode-kvgpb
     source-address 10.0.0.1
     stream yang-push
     update-policy periodic 500
     filter xpath /process-cpu-ios-xe-oper:cpu-usage/cpu-utilization/five-seconds
     receiver ip address 10.0.0.2 57000 protocol grpc-tcp
     exit
     copy running-config startup-config
     ```
   - Slanje telemetrije o iskorišćenju memorije:
     ```
     enable
     configure terminal
     telemetry ietf subscription 2
     encoding encode-kvgpb
     source-address 10.0.0.1
     stream yang-push
     update-policy periodic 500
     filter xpath /process-cpu-ios-xe-oper:cpu-usage/cpu-utilization/five-seconds
     receiver ip address 10.0.0.2 57000 protocol grpc-tcp
     exit
     copy running-config startup-config
     ```
     
   - Slanje telemetrije o saobraćaju na mrežnim interfejsima:
     ```
     enable
     configure terminal
     telemetry ietf subscription 3
     encoding encode-kvgpb
     source-address 10.0.0.1
     stream yang-push
     update-policy periodic 500
     filter xpath /interfaces-ios-xe-oper:interfaces/interface/statistics
     receiver ip address 10.0.0.2 57000 protocol grpc-tcp
     exit
     copy running-config startup-config
     ```

# Prikazivanje podataka dobijenih od rutera.




