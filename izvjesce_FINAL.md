# FINALNO IZVJEŠĆE ZA LIJEPLJENJE U WORD (poglavlja 1–9)
> Kopiraj poglavlja redom u službeni `.docx`. Naredbe/ispise lijepi monospace fontom (Consolas/Courier New).
> Ovo NIJE dio izvješća — obriši ovaj okvir. Jedino što trebaš još: (1) prilagoditi sate/postotke u pogl. 6,
> (2) zalijepiti NSG ispis na označeno mjesto u 3.1.

**Autori i odgovornost u projektu**
- Filip Kušer — Azure okruženje, mreža, LUKS konfiguracija
- Patrik Ostrunić — skripte/ispitivanje, validacija, dokumentacija

---

## 1. Sažetak i cilj rješenja

U ovom projektu obradili smo zaštitu povjerljivosti **podataka u mirovanju** (engl. *data at rest*) na dodatnom
disku virtualnog stroja u oblaku. Tehnički cilj bio je izraditi šifrirani volumen čiji korisnički podaci bez
ispravnog ključa nisu čitljivi, učiniti ga upotrebljivim (otključavanje, datotečni sustav, montiranje) te trajnim
kroz ponovna pokretanja sustava. Rješenje smo izveli na jednom **Rocky Linux 9.6** VM-u u **Microsoft Azureu**
(Switzerland North) s dodatnim 4 GiB diskom, koristeći **LUKS2** (`cryptsetup`/`dm-crypt`) i **XFS**, uz automatsko
otključavanje preko datoteke-ključa i `/etc/crypttab`. Sami smo isplanirali arhitekturu, kreirali Azure resurse,
postavili i konfigurirali LUKS te osmislili i proveli plan ispitivanja (pet provjera). Rezultat smo dokazali
kontrolnim sumama (SHA-256) koje su **nakon ponovnog pokretanja ostale nepromijenjene**, te dokazom da se
jedinstveni tekstualni marker nalazi u dešifriranoj datoteci, ali **ne** i na sirovom uređaju (tip `crypto_LUKS`).
Glavni je zaključak da LUKS pouzdano štiti podatke u mirovanju i predstavlja upravo tehnologiju koju Azure Disk
Encryption koristi na Linuxu. Najvažnije ograničenje je što datoteka-ključ za automatsko otključavanje leži na
(platformski šifriranom) OS disku, pa zaštita ovisi i o sigurnosti tog diska — u produkciji bi se ključ čuvao u
Azure Key Vaultu ili TPM-u.

## 2. Arhitektura sustava i mrežna topologija

Postavili smo **jedan virtualni stroj** s operacijskim sustavom **Rocky Linux 9.6** u Microsoft Azureu te mu
pridružili **jedan dodatni podatkovni disk** koji je predmet zaštite. Cilj arhitekture je zaštititi podatke u
mirovanju na tom disku šifriranjem, neovisno o platformi.

| Element | Vrijednost |
|---|---|
| Broj virtualnih strojeva | 1 |
| Operacijski sustav | Rocky Linux 9.6 (Gen2, Trusted Launch) |
| Glavni servisi / alati | OpenSSH (pristup), cryptsetup / LUKS2, dm-crypt, XFS |
| Otvoreni portovi | 22 (SSH) |
| Način pristupa | SSH (autentikacija ključem) |
| Kratki opis arhitekture | Jedan Rocky Linux 9.6 VM u Azureu s dodatnim diskom od 4 GiB. Pristup je isključivo SSH-om (port 22), a NSG propušta samo taj promet. Dodatni disk šifriran je LUKS2 volumenom (dm-crypt) i montiran na /mnt/secure, dok je OS disk zaštićen Azure SSE-om. |

**Infrastruktura u oblaku (Azure)**

| Resurs | Naziv | Namjena |
|---|---|---|
| Grupa resursa | NUX26-16-rg | Objedinjuje sve resurse teme (Switzerland North) |
| Virtualni stroj VM1 | NUX26-16-vm1 | Standard_B2ats_v2 (2 vCPU, 1 GiB), Rocky 9.6 |
| Dodatni disk | NUX26-16-datadisk | 4 GiB, Standard_LRS, LUN 0 — šifrirani volumen |
| Javna IP adresa | NUX26-16-pip | SSH pristup administratora |
| Mrežna grupa (NSG) | NUX26-16-nsg | Dopušta samo SSH (22) |
| Virtualna mreža / podmreža | NUX26-16-vnet / -subnet | 10.0.0.0/16 |

| VM / resurs | Uloga | Privatni IP | Javni IP | Servisi | Portovi |
|---|---|---|---|---|---|
| NUX26-16-vm1 | Šifrirana pohrana podataka | 10.0.0.4 | 51.107.91.110 | SSH, LUKS/dm-crypt | 22 |

**Tekstualni prikaz arhitekture**
```
Administrator (lokalni terminal)
   |
   |  SSH (port 22, autentikacija ključem)
   v
Azure VNet 10.0.0.0/16  ──  NSG (NUX26-16-nsg): dopušten samo 22/SSH
   |
   v
VM1: NUX26-16-vm1 (Rocky Linux 9.6, 10.0.0.4 / javni 51.107.91.110)
   |
   |  OS disk: /dev/sdX (XFS)                    << šifriran Azure SSE-om (platformski)
   +→ Dodatni disk: /dev/disk/azure/scsi1/lun0 (4 GiB)  << naš LUKS šifrirani volumen
```

## 3. Implementacija i konfiguracija sustava

Postupak izgradnje prikazujemo kronološki, korak po korak; uz svaki korak naveden je sirovi tekstualni ispis koji
ujedno služi kao revizijski dokaz.

### Korak 1 — Pozadina i odluka o pristupu
- **Opis i cilj:** Obrazložiti pristup — podatke na dodatnom disku štitimo **ručnim LUKS-om** (umjesto gotovog ADE-a) radi demonstracije mehanizma i pune kontrole nad ključem.

Azure nudi nekoliko razina zaštite diskova:
- **Server-Side Encryption (SSE) — šifriranje u mirovanju.** Svaki Azure *managed* disk automatski je šifriran na razini platforme (AES-256); zadano platformskim ključem (PMK), uz mogućnost vlastitog ključa (CMK) u Key Vaultu. Uvijek uključeno, ali ključem upravlja platforma.
- **Azure Disk Encryption (ADE) — enkripcija na razini gosta.** Koristi mehanizme OS-a: na Linuxu **dm-crypt / LUKS**, na Windowsu BitLocker; integrira se s Azure Key Vaultom.
- **Encryption at host.** Šifriranje na poslužitelju koji izvodi VM (uključuje i privremene diskove i cache).

Mi **ručno implementiramo LUKS** — istu `dm-crypt`/LUKS tehnologiju koju ADE koristi „u pozadini" na Linuxu — čime demonstriramo mehanizam, zadržavamo punu kontrolu nad ključem i izbjegavamo ADE-ova ograničenja (podržane slike/veličine, ovisnost o Key Vaultu). OS disk je usto zaštićen Azure SSE-om, pa imamo dvije neovisne razine zaštite.

### Korak 2 — Kreiranje Azure resursa
- **Opis i cilj:** Kreirati Azure okruženje (resursna grupa, VM Rocky 9.6, dodatni 4 GiB disk, NSG samo SSH) uz imenovanje `NUX26-16-*`, najmanju free-eligible veličinu radi kontrole troškova i fiksiranu verziju slike (`9-base:9.6.20250531`, Gen2) radi ponovljivosti.
- **Izvršene naredbe:**
```
$ az group create -n NUX26-16-rg -l switzerlandnorth -o table
Location          Name
----------------  -----------
switzerlandnorth  NUX26-16-rg

$ az vm image show --urn resf:rockylinux-x86_64:9-base:9.6.20250531 --query hyperVGeneration -o tsv
V2

$ az vm create -g NUX26-16-rg -n NUX26-16-vm1 \
    --image resf:rockylinux-x86_64:9-base:9.6.20250531 \
    --size Standard_B2ats_v2 --security-type TrustedLaunch \
    --admin-username azureuser --generate-ssh-keys \
    --public-ip-sku Standard --public-ip-address NUX26-16-pip \
    --nsg NUX26-16-nsg --nsg-rule SSH \
    --vnet-name NUX26-16-vnet --subnet NUX26-16-subnet \
    --storage-sku Standard_LRS -o table
ResourceGroup  PowerState  PublicIpAddress  PrivateIpAddress  Location
-------------  ----------  ---------------  ----------------  ----------------
NUX26-16-rg    VM running  51.107.91.110    10.0.0.4          switzerlandnorth

$ az vm disk attach -g NUX26-16-rg --vm-name NUX26-16-vm1 \
    --name NUX26-16-datadisk --new --size-gb 4 --sku Standard_LRS --lun 0 -o table
(uspjeh — prazan ispis)
```

### Korak 3 — Priprema VM-a i identifikacija diska
- **Opis i cilj:** Pripremiti VM (instalirati `cryptsetup`) i sigurno identificirati prazan dodatni disk preko stabilne putanje `/dev/disk/azure/scsi1/lun0` (ime `sdX` može varirati). Napomena: firewalld nije na ovom minimalnom imageu, pa mrežnu zaštitu radi Azure NSG (samo 22).
- **Izvršene naredbe:**
```
$ cat /etc/os-release
NAME="Rocky Linux"
VERSION="9.6 (Blue Onyx)"
ID="rocky"
PLATFORM_ID="platform:el9"
PRETTY_NAME="Rocky Linux 9.6 (Blue Onyx)"

$ rpm -q cryptsetup xfsprogs
package cryptsetup is not installed
xfsprogs-6.4.0-5.el9.x86_64
$ sudo dnf install -y cryptsetup xfsprogs
... Installed: cryptsetup-2.8.1-3.el9.x86_64 ; Upgraded: xfsprogs-6.4.0-7.el9 ...
$ rpm -q cryptsetup xfsprogs
cryptsetup-2.8.1-3.el9.x86_64
xfsprogs-6.4.0-7.el9.x86_64
$ cryptsetup --version
cryptsetup 2.8.1 flags: UDEV BLKID KEYRING FIPS KERNEL_CAPI PWQUALITY

$ sudo firewall-cmd --list-all
sudo: firewall-cmd: command not found        # firewalld nije instaliran (zaštitu radi Azure NSG)

$ lsblk -o NAME,HCTL,SIZE,TYPE,FSTYPE,MOUNTPOINTS
NAME   HCTL        SIZE TYPE FSTYPE MOUNTPOINTS
sda    0:0:0:0      10G disk
├─sda2             100M part vfat   /boot/efi
├─sda3            1000M part xfs    /boot
└─sda4             8.9G part xfs    /
sdb    1:0:0:0       4G disk
sr0    0:0:0:2     628K rom
$ ls -l /dev/disk/azure/scsi1/
lrwxrwxrwx. 1 root root 12 ... lun0 -> ../../../sdb        # naš prazan 4 GiB disk
```

### Korak 4 — Izrada LUKS volumena
- **Opis i cilj:** Na dodatnom disku izraditi LUKS2 šifrirani volumen (`aes-xts-plain64`, `argon2id`), uz `--pbkdf-memory 262144` (256 MiB) zbog 1 GiB RAM-a da otključavanje radi i pri pokretanju. (`pwquality` odbija preslabe zaporke; `luksDump` potvrđuje LUKS2/aes-xts-plain64/argon2id.)
- **Izvršene naredbe:**
```
$ DISK=/dev/disk/azure/scsi1/lun0
$ sudo cryptsetup luksFormat --pbkdf-memory 262144 "$DISK"
WARNING! This will overwrite data on /dev/disk/azure/scsi1/lun0 irrevocably.
Are you sure? (Type 'yes' in capital letters): YES
Enter passphrase ...:            (zaporka se ne ispisuje)
Verify passphrase:
(volumen uspješno kreiran)

$ sudo cryptsetup luksDump "$DISK"
LUKS header information
Version:        2
UUID:           4fce498f-3efc-4208-9d06-928183be8177
Data segments:
  0: crypt   offset: 16777216    cipher: aes-xts-plain64   sector: 4096 [bytes]
Keyslots:
  0: luks2   Key: 512 bits   Cipher: aes-xts-plain64   PBKDF: argon2id
             Time cost: 11   Memory: 262144   Threads: 2
Digests:
  0: pbkdf2  Hash: sha256   Iterations: 212090
```

### Korak 5 — Otključavanje, formatiranje i montiranje
- **Opis i cilj:** Otključati volumen (`luksOpen securedata`), postaviti XFS i montirati ga na `/mnt/secure`; `lsblk -f` pokazuje dvije razine (sirovi `crypto_LUKS` → mapper `xfs`).
- **Izvršene naredbe:**
```
$ sudo cryptsetup luksOpen "$DISK" securedata
Enter passphrase ...:            (zaporka se ne ispisuje)
$ sudo mkfs.xfs /dev/mapper/securedata
meta-data=/dev/mapper/securedata isize=512  agcount=4, agsize=261120 blks
data     = bsize=4096   blocks=1044480
$ sudo mkdir -p /mnt/secure && sudo mount /dev/mapper/securedata /mnt/secure
$ df -hT /mnt/secure
Filesystem             Type  Size  Used Avail Use% Mounted on
/dev/mapper/securedata xfs   4.0G   61M  3.9G   2% /mnt/secure
$ lsblk -f
sdb          crypto_LUKS 2  4fce498f-3efc-4208-9d06-928183be8177
└─securedata xfs              8ce7c164-2c0f-4b6d-8362-43e660db009d   /mnt/secure
```

### Korak 6 — Test pohrane podataka
- **Opis i cilj:** Zapisati testne podatke (jedinstveni marker + 10 MB nasumičnih bajtova) i spremiti njihove SHA-256 sume u `/root/sums.before` (izvan šifriranog volumena) radi kasnije provjere integriteta.
- **Izvršene naredbe:**
```
$ echo "NUX26-16-PLAINTEXT-MARKER test pohrane $(date)" | sudo tee /mnt/secure/test.txt
NUX26-16-PLAINTEXT-MARKER test pohrane Sat Jun  6 13:58:18 UTC 2026
$ sudo dd if=/dev/urandom of=/mnt/secure/podaci.bin bs=1M count=10
10485760 bytes (10 MB, 10 MiB) copied, 0.0376343 s, 279 MB/s
$ sudo sha256sum /mnt/secure/test.txt /mnt/secure/podaci.bin | sudo tee /root/sums.before
5bf66638534faf2602356b7fc85fc619ddce98967d137f367af534c44b4a152f  /mnt/secure/test.txt
31f5468f06f7cdac297430d0d20f160c75e87d9c65f9e1763b3d092c9cce2463  /mnt/secure/podaci.bin
```

### Korak 7 — Trajno (automatsko) otključavanje
- **Opis i cilj:** Omogućiti automatsko otključavanje i montiranje pri pokretanju (datoteka-ključ + `/etc/crypttab` preko **LUKS UUID-a** + `/etc/fstab`, opcija `nofail`), bez ručnog upisa zaporke na headless VM-u; konfiguraciju validirati prije reboota.
- **Izvršene naredbe:**
```
$ sudo mkdir -p /etc/luks
$ sudo dd if=/dev/urandom of=/etc/luks/securedata.key bs=512 count=8
4096 bytes (4.1 kB, 4.0 KiB) copied
$ sudo chmod 0400 /etc/luks/securedata.key ; sudo chown root:root /etc/luks/securedata.key
$ sudo ls -l /etc/luks/securedata.key
-r--------. 1 root root 4096 ... /etc/luks/securedata.key
$ sudo cryptsetup luksAddKey "$DISK" /etc/luks/securedata.key
Enter any existing passphrase:   (zaporka se ne ispisuje)
$ sudo cryptsetup luksDump "$DISK" | grep -E "Keyslots|luks2"
Keyslots:
  0: luks2
  1: luks2

$ cat /etc/crypttab
securedata UUID=4fce498f-3efc-4208-9d06-928183be8177 /etc/luks/securedata.key luks,nofail
$ cat /etc/fstab
UUID=cd23ec61-... / xfs defaults 0 1
UUID=c6a3f4bf-... /boot xfs defaults 0 0
UUID=4C62-2DDB /boot/efi vfat defaults,umask=0077,shortname=winnt 0 0
/dev/mapper/securedata /mnt/secure xfs defaults,nofail 0 0

# Validacija prije reboota:
$ sudo umount /mnt/secure ; sudo cryptsetup close securedata ; sudo systemctl daemon-reload
$ sudo systemctl start systemd-cryptsetup@securedata.service ; sudo systemctl start mnt-secure.mount
$ systemctl is-active systemd-cryptsetup@securedata.service mnt-secure.mount
active
active
```

### Korak 8 — Kontrola Azure troškova
- **Opis i cilj:** Zaustaviti naplatu računanja (dealokacija VM-a) i isplanirati brisanje resursa nakon ocjene; tijekom rada korištena najmanja free-eligible veličina (`Standard_B2ats_v2`), najjeftiniji disk (`Standard_LRS`) i otvoren samo port 22.
- **Izvršene naredbe:**
```
$ az vm deallocate -g NUX26-16-rg -n NUX26-16-vm1
(VM prebačen u stanje 'Stopped (deallocated)' — prestaje naplata računanja)

# Konačno čišćenje nakon obrane:
$ az group delete -n NUX26-16-rg --yes --no-wait
```

### 3.1 Revizijski trag i tehnički dokazi

| Obvezni tehnički dokaz | Naredba / rezultat | Lokacija (Korak) |
|---|---|---|
| Instalirani paketi | `rpm -q cryptsetup xfsprogs` → cryptsetup-2.8.1, xfsprogs-6.4.0 | Korak 3 |
| Aktivni servis | `systemctl is-active systemd-cryptsetup@securedata` → active; `cryptsetup status securedata` → LUKS2 active | Korak 7, pogl. 4 |
| Otvoreni portovi | `ss -tulpen` → sshd sluša na 22 | pogl. 4 (I dolje) |
| Status vatrozida / mrežna zaštita | `firewall-cmd` → not found (firewalld nije instaliran); Azure NSG dopušta samo 22 (ispis ↓) | Korak 3 + ispod |
| Ključna konfiguracija | `cat /etc/crypttab`, `cat /etc/fstab`, `cryptsetup luksDump` | Korak 4 i 7 |
| Validacija rješenja | reboot + `sha256sum -c` (OK) + dokaz enkripcije | Poglavlje 4 |

```
# Dokaz da NSG dopušta samo SSH (22):
$ az network nsg rule list -g NUX26-16-rg --nsg-name NUX26-16-nsg -o table
Name               Priority  Access  Protocol  Direction  DestinationPortRanges
-----------------  --------  ------  --------  ---------  ---------------------
default-allow-ssh  1000      Allow   Tcp       Inbound    22

# Otvoreni portovi: izvana je (preko Azure NSG-a) dohvatljiv ISKLJUČIVO 22/SSH.
$ sudo ss -tulpen
tcp LISTEN 0 128 0.0.0.0:22  users:(("sshd",pid=1021))     # SSH — jedini izvana dostupan (NSG)
tcp LISTEN 0 128 [::]:22     users:(("sshd",pid=1021))
udp UNCONN 0 0 0.0.0.0:111   users:(("rpcbind",...))       # sluša na svim sučeljima, ali NSG ne propušta izvana
udp UNCONN 0 0 127.0.0.1:323 users:(("chronyd",...))       # samo loopback (lokalno)
```

## 4. Ispitivanje funkcionalnosti i validacija rješenja

Definirali smo pet provjera (tema srednje/napredne težine traži 4–5).

| Oznaka | Provjera | Naredba / radnja | Očekivano | Dobiveno | Status |
|---|---|---|---|---|---|
| I1 | Valjanost LUKS headera | `cryptsetup luksDump` | LUKS2, aes-xts-plain64, argon2id | LUKS2, aes-xts-plain64, 512-bit, argon2id, 2 keyslota | OK |
| I2 | Tip uređaja (dvije razine) | `lsblk -f`, `blkid` | sirovi = crypto_LUKS, mapper = xfs | crypto_LUKS + securedata = xfs | OK |
| I3 | Otključavanje + mount + upis | `luksOpen`, `mkfs.xfs`, `mount`, upis, `df` | volumen montiran, upis radi | montiran na /mnt/secure; zapisani test.txt + podaci.bin | OK |
| I4 | Trajnost + integritet (reboot) | reboot + `sha256sum -c /root/sums.before` | oba `OK`, automatsko montiranje | oba `OK`; volumen se sam otključao i montirao | OK |
| I5 | Učinkovitost enkripcije | zatvori → `mount` sirovog (fail) + traženje markera (sirovi vs. dešifrirani) | mount ne uspijeva; marker nije na sirovom uređaju | `unknown filesystem type 'crypto_LUKS'`; marker NIJE na sirovom (PASS), a JEST u dešifriranoj datoteci | OK |

**Ključni ispisi ispitivanja**
```
# I4 — nakon reboota (slova diska su se zamijenila: sada DATA=sda, OS=sdb):
$ sudo cryptsetup status securedata
/dev/mapper/securedata is active and is in use.
  type: LUKS2   cipher: aes-xts-plain64   keysize: 512 [bits]   device: /dev/sda
$ ls -l /mnt/secure
-rw-r--r--. 1 root root 10485760 ... podaci.bin
-rw-r--r--. 1 root root       71 ... test.txt
$ sudo sha256sum -c /root/sums.before
/mnt/secure/test.txt: OK
/mnt/secure/podaci.bin: OK

# I5 — dokaz enkripcije (zatvoreni volumen):
$ sudo cryptsetup close securedata ; sudo blkid "$DISK"
/dev/disk/azure/scsi1/lun0: UUID="4fce498f-..." TYPE="crypto_LUKS"
$ sudo mount "$DISK" /mnt/secure
mount: /mnt/secure: unknown filesystem type 'crypto_LUKS'.
$ sudo strings "$DISK" | grep -qm1 'NUX26-16-PLAINTEXT-MARKER' && echo NADJENO || echo "PASS: marker NIJE na sirovom uredaju"
PASS: marker NIJE na sirovom uredaju
$ sudo grep -c 'NUX26-16-PLAINTEXT-MARKER' /mnt/secure/test.txt   # nakon ponovnog otključavanja
1
```

## 5. Analiza incidenata i rješavanje tehničkih izazova

Najveći tehnički rizik kod ove teme je **automatsko otključavanje pri pokretanju na headless VM-u**: kad bi LUKS
volumen u `/etc/crypttab` tražio zaporku, a VM nema interaktivnu konzolu, pokretanje bi se moglo zaglaviti. Taj smo
rizik **preventivno** uklonili datotekom-ključem i opcijom `nofail`, a konfiguraciju validirali prije reboota.

Tijekom reboot-testa **stvarno smo primijetili** zanimljiv izazov: **slova uređaja su se zamijenila** — prije reboota
je OS disk bio `sda`, a podatkovni `sdb`, dok je nakon reboota OS disk postao `sdb`, a podatkovni `sda`. Da smo se
oslonili na ime uređaja, automatsko otključavanje bi puklo; budući da smo koristili **LUKS UUID**, sve je prošlo
ispravno — što potvrđuje ispravnost te odluke.

| Poteškoća | Kako je riješena | Status |
|---|---|---|
| Moguće zaglavljivanje boota (zaporka na headless VM-u) | keyfile + `nofail` + validacija prije reboota | Riješeno |
| Nestabilnost imena uređaja (sda↔sdb nakon reboota) | korištenje LUKS UUID-a u `crypttab`/`fstab` | Riješeno |
| `cryptsetup` odbija preslabe zaporke (`pwquality`) | postavljena dovoljno jaka zaporka | Riješeno |
| `grep` po cijelom disku iscrpi memoriju (1 GiB RAM) | korišten `strings` (stream) umjesto `grep` nad blok-uređajem | Riješeno |

## 6. Organizacija tima i podjela odgovornosti

> ⚠️ PRILAGODITE postotke i sate stvarnom stanju prije predaje (obrišite ovu napomenu).

| Aktivnost | Filip Kušer | Patrik Ostrunić |
|---|---|---|
| Planiranje | 50% | 50% |
| Azure resursi | 70% | 30% |
| Linux konfiguracija | 70% | 30% |
| Ispitivanje | 40% | 60% |
| Dokumentacija | 40% | 60% |
| Prezentacija | 40% | 60% |

| Student | Glavni doprinos | Konkretno odrađeni zadaci | Sati |
|---|---|---|---|
| Filip Kušer | Azure okruženje + LUKS konfiguracija | kreirao VM/disk/NSG, `luksFormat`/`luksOpen`/`mkfs.xfs`/`mount`, izradio keyfile, postavio `/etc/crypttab` i `/etc/fstab` | ~7 |
| Patrik Ostrunić | Ispitivanje + dokumentacija | proveo provjere I1–I5, reboot/integritet i dokaz enkripcije, vodio izradu izvješća i prezentacije | ~7 |

Oba člana razumiju cjelokupno rješenje i sposobna su samostalno obrazložiti svaki korak.

## 7. Kritički osvrt i naučene lekcije

Najveću praktičnu vrijednost donijelo nam je razumijevanje kako LUKS/`dm-crypt` stvarno radi „ispod haube" — od
formata zaglavlja i keyslotova, preko izvođenja ključa (argon2id) i prilagodbe memorije na slabom VM-u, do
integracije s `crypttab`/`fstab` i automatskog otključavanja. Posebno smo naučili koliko je važno oslanjati se na
stabilne identifikatore (**LUKS UUID**) umjesto na imena uređaja, što nam je reboot-test izravno potvrdio. Da projekt
radimo ispočetka, cijelu bismo izvedbu **automatizirali** (cloud-init ili Ansible), a datoteku-ključ bismo umjesto
pohrane na disku čuvali u **Azure Key Vaultu** radi veće sigurnosti.

## 8. Reference i uporaba AI alata

**Korišteni izvori i dokumentacija:**
- [1] `cryptsetup` / LUKS — priručničke stranice `cryptsetup(8)`, `crypttab(5)` te službeni projekt: https://gitlab.com/cryptsetup/cryptsetup (pristupljeno 6.6.2026.).
- [2] Microsoft Learn: *Overview of managed disk encryption options*: https://learn.microsoft.com/azure/virtual-machines/disk-encryption-overview (pristupljeno 6.6.2026.).
- [3] Microsoft Learn: *Add a data disk to a Linux VM*: https://learn.microsoft.com/azure/virtual-machines/linux/add-disk (pristupljeno 6.6.2026.).
- [4] Rocky Linux / RHEL 9 — dokumentacija o šifriranju blok uređaja (LUKS): https://docs.redhat.com/ (pristupljeno 6.6.2026.).

**Izjava o uporabi AI alata:**
- **Korišteni alati:** AI asistent (Claude, Anthropic) i GPT-5 (OpenAI).
- **Svrha i opseg:** Claude — pomoć pri strukturiranju izvješća i objašnjenju sintakse naredbi (`cryptsetup`, `crypttab`, `fstab`); GPT-5 — završna provjera potpunosti i točnosti izvješća. AI nije zamijenio vlastitu izvedbu.
- **Kako je provjerena točnost:** sve naredbe i konfiguracije izvršili smo i provjerili na vlastitom Azure VM-u; rezultati (sirovi ispisi, kontrolne sume, ponašanje pri ponovnom pokretanju, dokaz enkripcije) potvrđeni su praktično i nalaze se u ovom izvješću.

## 9. Izjava o samostalnosti i izvornosti rada

Ovom predajom potvrđujemo da je opisano rješenje rezultat našeg vlastitog rada te da su svi korišteni izvori, tuđa
pomoć, programski kod, konfiguracije i AI alati navedeni u referencama. Razumijemo da se u slučaju timskog rada
ocjenjuje individualni doprinos svakog člana tima te da svaki član mora samostalno obrazložiti cjelokupno rješenje
tijekom obrane.

Filip Kušer · Patrik Ostrunić
