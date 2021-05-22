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
5. Potrebno je namestiti i opciju **ethernet.virtualdev**. Ovu opciju nije moguće pronaći u GUI-u, te je potrebno ručno modifikovati konfiguracioni fajl od napravljene virtuelne mašine. Locirajte gde se nalazi mašina na vašem računaru (obično je u: **Documents\Virtual Machines\Ubuntu 64-bit**). Konfiguracioni fajl ima ekstenziju .vmx. Otvorite ga u tekstualnom editoru i promenite opciju: **ethernet0.virtualDev = "vmxnet3"**.

## Instalacija dodatnih alata
Potrebno je instalirati tri alata:
- Telegraf - prikupljanje telemetrijskih podataka.
- InfluxDB - čuvanje prikupljenih podataka.
- Grafana - iscrtavanje prikupljenih podataka.

# Konfiguracija virtuelnog rutera

