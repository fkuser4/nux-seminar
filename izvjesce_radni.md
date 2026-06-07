# RADNA VERZIJA IZVJEŠĆA — Tema 16 (Enkripcija diska: Azure + LUKS)
> Ovo je radni dokument iz kojeg se tekst lijepi u službeni `.docx` predložak.
> Sirovi ispisi su stvarni (s našeg VM-a). Poglavlja se popunjavaju usporedno s izvedbom.

---

## PRIKUPLJENI SIROVI ISPISI (kronološki — izvor za revizijski trag)

### [F1] Azure — kreiranje resursne grupe
```
$ az group create -n NUX26-16-rg -l switzerlandnorth -o table
Location          Name
----------------  -----------
switzerlandnorth  NUX26-16-rg
```

### [F1] Azure — odabir Rocky Linux 9 slike (Gen2)
```
$ az vm image list --all --publisher resf --offer rockylinux-x86_64 -o table
... (skraćeno) ...
x64  rockylinux-x86_64  resf  9-base  resf:rockylinux-x86_64:9-base:9.6.20250531  9.6.20250531
$ az vm image show --urn resf:rockylinux-x86_64:9-base:9.6.20250531 --query hyperVGeneration -o tsv
V2
$ az vm image terms accept --urn resf:rockylinux-x86_64:9-base:9.6.20250531
Accepted: True   (uvjeti marketplace slike prihvaćeni)
```

### [F1] Azure — kreiranje VM-a
```
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
```

### [F1] Azure — dodavanje dodatnog diska (4 GiB, LUN 0)
```
$ az vm disk attach -g NUX26-16-rg --vm-name NUX26-16-vm1 \
    --name NUX26-16-datadisk --new --size-gb 4 --sku Standard_LRS --lun 0 -o table
(uspjeh — prazan ispis)
```

### [F2] VM — verzija OS-a
```
$ cat /etc/os-release
PRETTY_NAME="Rocky Linux 9.6 (Blue Onyx)"
VERSION_ID="9.6"
... (Rocky 9.6, platform:el9) ...
```

### [F2] VM — instalacija/provjera paketa
```
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
```

### [F2] VM — identifikacija diska
```
$ lsblk -o NAME,HCTL,SIZE,TYPE,FSTYPE,MOUNTPOINTS
NAME   HCTL        SIZE TYPE FSTYPE MOUNTPOINTS
sda    0:0:0:0      10G disk
├─sda1               2M part
├─sda2             100M part vfat   /boot/efi
├─sda3            1000M part xfs    /boot
└─sda4             8.9G part xfs    /
sdb    1:0:0:0       4G disk
sr0    0:0:0:2     628K rom
$ ls -l /dev/disk/azure/scsi1/
lrwxrwxrwx. 1 root root 12 ... lun0 -> ../../../sdb
```
> Napomena: B2ats_v2 nema privremeni disk → `sdb` je naš podatkovni disk; koristimo stabilnu putanju
> `/dev/disk/azure/scsi1/lun0`. firewalld nije instaliran na imageu (mrežnu zaštitu radi Azure NSG, port 22).

### [F3] VM — izrada LUKS2 volumena (pwquality je odbio slabe zaporke)
```
$ DISK=/dev/disk/azure/scsi1/lun0
$ sudo cryptsetup luksFormat --pbkdf-memory 262144 "$DISK"
WARNING! This will overwrite data on /dev/disk/azure/scsi1/lun0 irrevocably.
Are you sure? (Type 'yes' in capital letters): YES
Enter passphrase ...:        << zaporka — REDIGIRANO (ne ispisuje se)
Verify passphrase:
(napomena: cryptsetup je prethodno odbio preslabe zaporke — "shorter than 8", "too simplistic")
(volumen uspješno kreiran)
```

### [F3] VM — LUKS header (luksDump — siguran, bez tajnog materijala)
```
$ sudo cryptsetup luksDump "$DISK"
LUKS header information
Version:        2
UUID:           4fce498f-3efc-4208-9d06-928183be8177
Data segments:
  0: crypt
        offset: 16777216 [bytes]    length: (whole device)
        cipher: aes-xts-plain64     sector: 4096 [bytes]
Keyslots:
  0: luks2
        Key:        512 bits
        Cipher:     aes-xts-plain64    Cipher key: 512 bits
        PBKDF:      argon2id
        Time cost:  11    Memory: 262144    Threads: 2
        AF stripes: 4000  AF hash: sha256
Digests:
  0: pbkdf2   Hash: sha256   Iterations: 212090
```

### [F3] VM — otključavanje + mapper uređaj
```
$ sudo cryptsetup luksOpen "$DISK" securedata
Enter passphrase ...:        << REDIGIRANO
$ ls -l /dev/mapper/securedata
lrwxrwxrwx. 1 root root 7 ... /dev/mapper/securedata -> ../dm-0
```

### [F3] VM — formatiranje (XFS) + montiranje
```
$ sudo mkfs.xfs /dev/mapper/securedata
meta-data=/dev/mapper/securedata isize=512  agcount=4, agsize=261120 blks
data     = bsize=4096   blocks=1044480, imaxpct=25
naming   =version 2  bsize=4096  ascii-ci=0, ftype=1
log      =internal log  bsize=4096  blocks=16384
$ sudo mount /dev/mapper/securedata /mnt/secure
$ df -hT /mnt/secure
Filesystem             Type  Size  Used Avail Use% Mounted on
/dev/mapper/securedata xfs   4.0G   61M  3.9G   2% /mnt/secure
$ lsblk -f
NAME         FSTYPE      FSVER LABEL UUID                                 FSUSE% MOUNTPOINTS
sda
├─sda2       vfat        FAT16 EFI   4C62-2DDB                               10%  /boot/efi
├─sda3       xfs               BOOT  c6a3f4bf-...                            28%  /boot
└─sda4       xfs               rocky cd23ec61-...                            12%  /
sdb          crypto_LUKS 2           4fce498f-3efc-4208-9d06-928183be8177
└─securedata xfs                     8ce7c164-2c0f-4b6d-8362-43e660db009d     2%  /mnt/secure
sr0
```

### [F4] VM — status volumena + test pohrane podataka
```
$ sudo cryptsetup status securedata
/dev/mapper/securedata is active and is in use.
  type:    LUKS2
  cipher:  aes-xts-plain64     keysize: 512 [bits]
  device:  /dev/sdb            sector size: 4096 [bytes]
  size:    8355840 [512-byte units] (4278190080 [bytes])
  mode:    read/write

$ echo "NUX26-16-PLAINTEXT-MARKER test pohrane $(date)" | sudo tee /mnt/secure/test.txt
NUX26-16-PLAINTEXT-MARKER test pohrane Sat Jun  6 01:58:18 PM UTC 2026
$ sudo dd if=/dev/urandom of=/mnt/secure/podaci.bin bs=1M count=10 status=progress
10485760 bytes (10 MB, 10 MiB) copied, 0.0376343 s, 279 MB/s
$ sudo sha256sum /mnt/secure/test.txt /mnt/secure/podaci.bin | sudo tee /root/sums.before
5bf66638534faf2602356b7fc85fc619ddce98967d137f367af534c44b4a152f  /mnt/secure/test.txt
31f5468f06f7cdac297430d0d20f160c75e87d9c65f9e1763b3d092c9cce2463  /mnt/secure/podaci.bin
```

### [F5] VM — keyfile + drugi keyslot
```
$ sudo dd if=/dev/urandom of=/etc/luks/securedata.key bs=512 count=8
4096 bytes (4.1 kB, 4.0 KiB) copied, 0.000164239 s, 24.9 MB/s
$ sudo chmod 0400 /etc/luks/securedata.key ; sudo chown root:root /etc/luks/securedata.key
$ sudo ls -l /etc/luks/securedata.key
-r--------. 1 root root 4096 Jun  6 14:01 /etc/luks/securedata.key
$ sudo cryptsetup luksAddKey "$DISK" /etc/luks/securedata.key
Enter any existing passphrase:          << REDIGIRANO
$ sudo cryptsetup luksDump "$DISK" | grep -E "Keyslots|luks2"
Keyslots:
  0: luks2
  1: luks2
```

### [F5] VM — /etc/crypttab i /etc/fstab
```
$ sudo cryptsetup luksUUID "$DISK"
4fce498f-3efc-4208-9d06-928183be8177
$ cat /etc/crypttab
securedata UUID=4fce498f-3efc-4208-9d06-928183be8177 /etc/luks/securedata.key luks,nofail
$ cat /etc/fstab
UUID=cd23ec61-b371-4676-9286-5ff096d92a9e / xfs defaults 0 1
UUID=c6a3f4bf-fb25-4906-807b-ade33eb1e71f /boot xfs defaults 0 0
UUID=4C62-2DDB /boot/efi vfat defaults,umask=0077,shortname=winnt 0 0
/dev/mapper/securedata /mnt/secure xfs defaults,nofail 0 0
```

### [F5] VM — validacija prije reboota (automatsko otključavanje + montiranje)
```
$ sudo umount /mnt/secure ; sudo cryptsetup close securedata
$ sudo systemctl daemon-reload
$ sudo systemctl start systemd-cryptsetup@securedata.service
$ sudo systemctl start mnt-secure.mount
$ findmnt /mnt/secure
TARGET      SOURCE                 FSTYPE OPTIONS
/mnt/secure /dev/mapper/securedata xfs    rw,relatime,seclabel,...

$ systemctl status systemd-cryptsetup@securedata.service mnt-secure.mount --no-pager
● systemd-cryptsetup@securedata.service - Cryptography Setup for securedata
     Loaded: loaded (/etc/crypttab; generated)
     Active: active (exited) since Sat 2026-06-06 14:03:45 UTC
    Process: ExecStart=/usr/lib/systemd/systemd-cryptsetup attach securedata
             /dev/disk/by-uuid/4fce498f-... /etc/luks/securedata.key luks,nofail (status=0/SUCCESS)
● mnt-secure.mount - /mnt/secure
     Loaded: loaded (/etc/fstab; generated)
     Active: active (mounted) since Sat 2026-06-06 14:03:47 UTC
      Where: /mnt/secure     What: /dev/mapper/securedata
```

### [F6] VM — nakon reboota: AUTO-otključavanje + integritet (slova diska su se ZAMIJENILA)
```
$ sudo reboot         (zatim ponovno spajanje SSH-om)
$ lsblk -f
NAME         FSTYPE      FSVER LABEL UUID                                 FSUSE% MOUNTPOINTS
sda          crypto_LUKS 2           4fce498f-3efc-4208-9d06-928183be8177          << sada DATA disk = sda
└─securedata xfs               ...   8ce7c164-...                            2%  /mnt/secure
sdb                                                                                << sada OS disk = sdb
├─sdb2       vfat   EFI   4C62-2DDB                                         10%  /boot/efi
├─sdb3       xfs    BOOT  c6a3f4bf-...                                      28%  /boot
└─sdb4       xfs    rocky cd23ec61-...                                      13%  /
$ sudo cryptsetup status securedata
/dev/mapper/securedata is active and is in use.
  type: LUKS2   cipher: aes-xts-plain64   keysize: 512 [bits]   device: /dev/sda
$ df -hT /mnt/secure
/dev/mapper/securedata xfs  4.0G  71M  3.9G  2%  /mnt/secure
$ ls -l /mnt/secure
-rw-r--r--. 1 root root 10485760 Jun  6 13:58 podaci.bin
-rw-r--r--. 1 root root       71 Jun  6 13:58 test.txt
$ sudo ss -tulpen           (izvod — bitno)
tcp LISTEN 0 128 0.0.0.0:22  users:(("sshd",pid=1021))      << SSH sluša na 22
tcp LISTEN 0 128 [::]:22     users:(("sshd",pid=1021))
(lokalno: rpcbind 111, chronyd 323 — izvana nedostupni, NSG dopušta samo 22)
$ sudo sha256sum -c /root/sums.before
/mnt/secure/test.txt: OK
/mnt/secure/podaci.bin: OK
```

### [F7] VM — dokaz enkripcije (zatvoreni uređaj nečitljiv; marker se ne nalazi)
```
$ sudo umount /mnt/secure ; sudo cryptsetup close securedata
$ sudo cryptsetup isLuks "$DISK" && echo "OK: uredaj je LUKS"
OK: uredaj je LUKS
$ lsblk -f
sda  crypto_LUKS 2  4fce498f-3efc-4208-9d06-928183be8177      << zatvoreno: nema mappera, samo crypto_LUKS
$ sudo blkid "$DISK"
/dev/disk/azure/scsi1/lun0: UUID="4fce498f-3efc-4208-9d06-928183be8177" TYPE="crypto_LUKS"
$ sudo mount "$DISK" /mnt/secure
mount: /mnt/secure: unknown filesystem type 'crypto_LUKS'.     << sirovi uređaj se ne može montirati

# Marker na SIROVOM uređaju (koristimo `strings` jer `grep` na 1 GiB RAM-a iscrpi memoriju
# zbog ogromnog niza neispisanih bajtova bez novog reda):
$ sudo strings "$DISK" | grep -qm1 'NUX26-16-PLAINTEXT-MARKER' && echo NADJENO || echo "PASS: marker NIJE na sirovom uredaju"
PASS: marker NIJE na sirovom uredaju (sifrirano)

# Isti marker u DEŠIFRIRANOJ datoteci (nakon ponovnog otključavanja keyfileom):
$ sudo grep -c 'NUX26-16-PLAINTEXT-MARKER' /mnt/secure/test.txt
1
```

---
---

# IZVJEŠĆE (čisti tekst za lijepljenje u predložak)

## 1. Sažetak i cilj rješenja

U ovom projektu obradili smo zaštitu povjerljivosti **podataka u mirovanju** (engl. *data at rest*) na dodatnom
disku virtualnog stroja u oblaku. Tehnički cilj bio je izraditi šifrirani volumen koji je bez ispravnog ključa
potpuno nečitljiv, učiniti ga upotrebljivim (otključavanje, datotečni sustav, montiranje) te trajnim kroz ponovna
pokretanja sustava. Rješenje smo izveli na jednom **Rocky Linux 9.6** VM-u u **Microsoft Azureu** (Switzerland
North) s dodatnim 4 GiB diskom, koristeći **LUKS2** (`cryptsetup`/`dm-crypt`) i **XFS**, uz automatsko otključavanje
preko datoteke-ključa i `/etc/crypttab`. Sami smo isplanirali arhitekturu, kreirali Azure resurse, postavili i
konfigurirali LUKS te osmislili i proveli plan ispitivanja (5 provjera). Rezultat smo dokazali kontrolnim sumama
(SHA-256) koje su **nakon ponovnog pokretanja ostale nepromijenjene**, te dokazom da se jedinstveni tekstualni
marker nalazi u dešifriranoj datoteci, ali **ne** i na sirovom uređaju (tip `crypto_LUKS`). Glavni je zaključak da
LUKS pouzdano štiti podatke u mirovanju i predstavlja upravo tehnologiju koju Azure Disk Encryption koristi na
Linuxu. Najvažnije ograničenje je što datoteka-ključ za automatsko otključavanje leži na (platformski šifriranom)
OS disku, pa zaštita ovisi i o sigurnosti tog diska — u produkciji bi se ključ čuvao u Azure Key Vaultu ili TPM-u.

## 2. Arhitektura sustava i mrežna topologija

U projektu smo postavili **jedan virtualni stroj** s operacijskim sustavom **Rocky Linux 9.6** u Microsoft
Azureu te mu pridružili **jedan dodatni podatkovni disk** koji je predmet zaštite. Cilj arhitekture je
zaštititi podatke u mirovanju (engl. *data at rest*) na tom disku pomoću šifriranja, neovisno o platformi.

**Pregled sustava**

| Element | Vrijednost |
|---|---|
| Broj virtualnih strojeva | 1 |
| Operacijski sustav | Rocky Linux 9.6 (Gen2, Trusted Launch) |
| Glavni servisi / alati | OpenSSH (pristup), cryptsetup / LUKS2, dm-crypt, XFS |
| Otvoreni portovi | 22 (SSH) |
| Način pristupa | SSH (autentikacija ključem) |

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
Administrator (Mac, Ghostty)
   |
   |  SSH (port 22, autentikacija ključem)
   v
Azure VNet 10.0.0.0/16  ──  NSG (NUX26-16-nsg): dopušten samo 22/SSH
   |
   v
VM1: NUX26-16-vm1 (Rocky Linux 9.6, 10.0.0.4 / javni 51.107.91.110)
   |
   |  OS disk: /dev/sda (10 GiB, XFS)            << šifriran Azure SSE-om (platformski)
   +→ Dodatni disk: /dev/disk/azure/scsi1/lun0 (4 GiB)  << naš LUKS šifrirani volumen
```

**Objašnjenje Azure disk enkripcije (uloga u našem rješenju)**

Azure nudi nekoliko razina zaštite diskova, pa je bitno razlikovati ih:

- **Server-Side Encryption (SSE) — šifriranje u mirovanju.** Svaki Azure *managed* disk je automatski šifriran
  na razini platforme (AES-256). Prema zadanim postavkama koriste se *platformski ključevi* (PMK), a moguće je
  koristiti i *vlastite ključeve* (CMK) pohranjene u Azure Key Vaultu. Ovo je transparentno i uvijek uključeno,
  ali ključ drži/upravlja platforma.
- **Azure Disk Encryption (ADE) — enkripcija na razini gosta.** Koristi mehanizme samog OS-a: na Linuxu
  **dm-crypt / LUKS**, a na Windowsu BitLocker. Integrira se s Azure Key Vaultom za pohranu ključeva i može
  šifrirati OS i/ili podatkovne diskove.
- **Encryption at host.** Šifriranje na samom poslužitelju koji izvodi VM (obuhvaća i privremene diskove i cache).

U ovom radu **ručno implementiramo LUKS** — dakle istu `dm-crypt`/LUKS tehnologiju koju ADE koristi „u pozadini"
na Linuxu. Time (a) jasno demonstriramo sam mehanizam šifriranja, (b) zadržavamo **punu kontrolu nad ključem**
(sami biramo zaporku i datoteku-ključ), i (c) rješenje je neovisno o ograničenjima ADE-a (podržane slike/veličine,
ovisnost o Key Vaultu). Dodatno, OS disk je i dalje zaštićen Azure SSE-om, pa imamo dvije neovisne razine zaštite.

## 3. Implementacija i konfiguracija sustava

Postupak izgradnje sustava prikazujemo kronološki, korak po korak. Uz svaki korak naveden je odgovarajući sirovi
tekstualni ispis (oznake [F1]–[F7] u prilogu „Prikupljeni sirovi ispisi"), koji ujedno služi kao revizijski dokaz.

### Korak 1 — Pozadina i odluka o pristupu
Cilj nam je zaštititi podatke na dodatnom disku tako da budu nečitljivi bez ispravnog ključa. Umjesto da
uključimo gotov ADE, odlučili smo **ručno izraditi LUKS2 volumen** jer tako demonstriramo točan mehanizam koji
ADE inače automatizira i zadržavamo kontrolu nad ključem. (Vidi objašnjenje u poglavlju 2.)

### Korak 2 — Kreiranje Azure resursa
Resurse smo kreirali putem Azure CLI-ja (Cloud Shell), poštujući imenovanje `NUX26-16-*` i koristeći najmanju
free-eligible veličinu VM-a radi kontrole troškova. *(Sirovi ispisi: vidi [F1].)*

### Korak 3 — Priprema VM-a i identifikacija diska
Spojili smo se SSH-om, potvrdili Rocky Linux 9.6, instalirali `cryptsetup` i identificirali prazan dodatni disk
preko stabilne putanje `/dev/disk/azure/scsi1/lun0`. *(Sirovi ispisi: vidi [F2].)*

### Korak 4 — Izrada LUKS volumena
Na praznom dodatnom disku (`/dev/disk/azure/scsi1/lun0`) izradili smo LUKS2 šifrirani volumen naredbom
`cryptsetup luksFormat`. Zbog ograničenih 1 GiB RAM-a eksplicitno smo ograničili memoriju funkcije za izvođenje
ključa na 256 MiB (`--pbkdf-memory 262144`) kako derivacija ne bi premašila dostupnu memoriju — i pri formatiranju
i kasnije pri automatskom otključavanju tijekom pokretanja sustava. Prilikom postavljanja zaporke `cryptsetup` je
kroz ugrađenu `pwquality` provjeru odbio preslabe zaporke, pa smo postavili dovoljno jaku. `luksDump` potvrđuje
LUKS2 format, šifru **aes-xts-plain64** s 512-bitnim ključem i **argon2id** KDF. *(Sirovi ispis: [F3].)*

### Korak 5 — Otključavanje, formatiranje i montiranje
Volumen smo otključali naredbom `cryptsetup luksOpen <disk> securedata`, čime u `/dev/mapper/securedata` nastaje
otključani (dešifrirani) prikaz uređaja. Na njega smo postavili **XFS** datotečni sustav (`mkfs.xfs`) i montirali ga
na `/mnt/secure`. `lsblk -f` jasno prikazuje dvije razine: sirovi disk `sdb` je tipa **`crypto_LUKS`**, a tek
otključani mapper `securedata` nosi **XFS** i dostupan je za upis. *(Sirovi ispis: [F3].)*

### Korak 6 — Test pohrane podataka
Na montirani šifrirani volumen zapisali smo testnu datoteku s jedinstvenim tekstualnim markerom
(`NUX26-16-PLAINTEXT-MARKER`) te 10 MB nasumičnih podataka (`/dev/urandom`). Kontrolne sume (SHA-256) obiju
datoteka spremili smo u `/root/sums.before` — namjerno **izvan** šifriranog volumena (na OS disk) — kako bismo
nakon ponovnog pokretanja sustava mogli dokazati da podaci nisu promijenjeni. *(Sirovi ispis: [F4].)*

### Korak 7 — Trajno (automatsko) otključavanje
Da bi se volumen otključavao i montirao automatski pri svakom pokretanju (bez ručnog upisa zaporke na headless
VM-u koji nema konzolu), generirali smo nasumičnu datoteku-ključ `/etc/luks/securedata.key` (4 KiB, prava `0400`,
vlasnik root) i dodali je kao **drugi LUKS keyslot** (`luksAddKey`) — uz postojeću zaporku. U `/etc/crypttab`
volumen referenciramo preko **LUKS UUID-a** i te datoteke-ključa, s opcijom `nofail`; u `/etc/fstab` dodali smo
montiranje `/mnt/secure` također s `nofail` (XFS, passno 0). Prije ponovnog pokretanja proveli smo validaciju:
zatvorili volumen pa pokrenuli generirane systemd jedinice — `systemd-cryptsetup@securedata.service` otključava
volumen keyfileom, a `mnt-secure.mount` ga montira; obje su `active`, čime smo bez rizika za boot potvrdili
ispravnost konfiguracije. *(Sirovi ispis: [F5].)*

### 3.1 Revizijski trag i tehnički dokazi

| Obvezni tehnički dokaz | Naredba / očekivani rezultat | Lokacija u izvješću |
|---|---|---|
| Instalirani paketi | `rpm -q cryptsetup xfsprogs` → cryptsetup-2.8.1, xfsprogs-6.4.0 | Korak 3 / [F2] |
| Aktivni servis | `systemctl status systemd-cryptsetup@securedata` → active; `cryptsetup status securedata` → LUKS2 active | Korak 7 / [F5], [F4], [F6] |
| Otvoreni portovi | `ss -tulpen` → sshd sluša na 22 | [F6] |
| Mrežna zaštita (vatrozid) | firewalld nije na minimalnom imageu; Azure **NSG** (NUX26-16-nsg) dopušta samo 22 | pogl. 2 / [F6] |
| Ključna konfiguracija | `cat /etc/crypttab`, `cat /etc/fstab`, `cryptsetup luksDump` | Korak 4 i 7 / [F3], [F5] |
| Validacija rješenja | reboot + `sha256sum -c` (OK) + dokaz enkripcije | Poglavlje 4 (I1–I5) |

## 4. Ispitivanje funkcionalnosti i validacija rješenja

Definirali smo pet provjera (tema srednje/napredne težine traži 4–5). Svaka ima naredbu, očekivani i dobiveni rezultat.

| Oznaka | Provjera | Naredba / radnja | Očekivano | Dobiveno | Status |
|---|---|---|---|---|---|
| I1 | Valjanost LUKS headera | `cryptsetup luksDump` | LUKS2, aes-xts-plain64, argon2id | LUKS2, aes-xts-plain64, 512-bit, argon2id, 2 keyslota | OK |
| I2 | Tip uređaja (dvije razine) | `lsblk -f`, `blkid` | sirovi = crypto_LUKS, mapper = xfs | crypto_LUKS + securedata = xfs | OK |
| I3 | Otključavanje + mount + upis | `luksOpen`, `mkfs.xfs`, `mount`, upis, `df` | volumen montiran, upis radi | montiran na /mnt/secure; zapisani test.txt + podaci.bin | OK |
| I4 | Trajnost + integritet (reboot) | reboot + `sha256sum -c /root/sums.before` | oba `OK`, automatsko montiranje | oba `OK`; volumen se sam otključao i montirao | OK |
| I5 | Učinkovitost enkripcije | zatvori → `mount` sirovog (fail) + `strings`+`grep` markera (sirovi vs. dešifrirani) | mount ne uspijeva; marker nije na sirovom uređaju | `unknown filesystem type 'crypto_LUKS'`; marker NIJE na sirovom (PASS), a JEST u dešifriranoj datoteci (`grep -c`=1) | OK |

*(Sirovi ispisi: [F3]–[F6]; I5 → [F7].)*

## 5. Analiza incidenata i rješavanje tehničkih izazova

Najveći tehnički rizik kod ove teme je **automatsko otključavanje pri pokretanju na headless VM-u**: kad bi LUKS
volumen u `/etc/crypttab` tražio zaporku, a VM nema interaktivnu konzolu, pokretanje sustava moglo bi se zaglaviti.
Taj smo rizik **preventivno** uklonili datotekom-ključem i opcijom `nofail`, a konfiguraciju smo validirali prije
samog reboota (Korak 7), pa boot nije bilo potrebno „pogađati".

Tijekom reboot-testa **stvarno smo primijetili** zanimljiv izazov: **slova uređaja su se zamijenila** — prije reboota
je OS disk bio `sda`, a podatkovni `sdb`, dok je nakon reboota OS disk postao `sdb`, a podatkovni `sda`. Da smo se u
`/etc/crypttab`/`/etc/fstab` oslonili na ime uređaja, automatsko otključavanje bi puklo. Budući da smo namjerno
koristili **LUKS UUID** (`UUID=4fce498f-…`), otključavanje i montiranje su svejedno prošli ispravno — čime je
potvrđena ispravnost te odluke.

| Poteškoća | Kako je riješena | Status |
|---|---|---|
| Moguće zaglavljivanje boota (zaporka na headless VM-u) | keyfile + `nofail` + validacija prije reboota | Riješeno |
| Nestabilnost imena uređaja (sda↔sdb nakon reboota) | korištenje LUKS UUID-a u `crypttab`/`fstab` | Riješeno |
| `cryptsetup` odbija preslabe zaporke (`pwquality`) | postavljena dovoljno jaka zaporka | Riješeno |

## 6. Organizacija tima i podjela odgovornosti

> ⚠️ PRILAGODITE postotke i sate stvarnom stanju — služe kao temelj za pitanja na obrani.

| Aktivnost | Filip Kušer | Patrik Ostrunić | Napomena |
|---|---|---|---|
| Planiranje | 50% | 50% | zajednički dogovor arhitekture i pristupa (ručni LUKS) |
| Azure resursi | 70% | 30% | kreiranje RG/VM/diska/NSG-a |
| Linux konfiguracija | 70% | 30% | LUKS, keyfile, crypttab, fstab |
| Ispitivanje | 40% | 60% | QA I1–I5, reboot/integritet, dokaz enkripcije |
| Dokumentacija | 40% | 60% | izvješće, revizijski trag |
| Prezentacija | 40% | 60% | izrada slajdova |

| Student | Glavni doprinos | Konkretno odrađeni zadaci | Utrošeno vrijeme (h) |
|---|---|---|---|
| Filip Kušer | Azure okruženje + LUKS konfiguracija | kreirao VM/disk/NSG, `luksFormat`/`luksOpen`/`mkfs.xfs`/`mount`, izradio keyfile, postavio `/etc/crypttab` i `/etc/fstab` | ~7 |
| Patrik Ostrunić | Ispitivanje + dokumentacija | proveo QA provjere (I1–I5), reboot/integritet i dokaz enkripcije, vodio izradu izvješća i prezentacije | ~7 |

Oba člana razumiju cjelokupno rješenje i sposobna su samostalno obrazložiti svaki korak.

## 7. Kritički osvrt i naučene lekcije

Najveću praktičnu vrijednost donijelo nam je razumijevanje kako LUKS/`dm-crypt` stvarno radi „ispod haube" — od
formata zaglavlja i keyslotova, preko izvođenja ključa (argon2id) i prilagodbe memorije na slabom VM-u, do
integracije s `crypttab`/`fstab` i automatskog otključavanja pri pokretanju. Posebno smo naučili koliko je važno
oslanjati se na stabilne identifikatore (**LUKS UUID**) umjesto na imena uređaja, što nam je reboot-test i izravno
potvrdio (slova `sda`/`sdb` zamijenila su se, ali je otključavanje preko UUID-a svejedno radilo). Da projekt radimo
ispočetka, cijelu bismo izvedbu **automatizirali** (cloud-init ili Ansible) radi ponovljivosti, a datoteku-ključ
bismo umjesto pohrane na disku čuvali u **Azure Key Vaultu** radi veće sigurnosti.

## 8. Reference i uporaba AI alata

**Korišteni izvori i dokumentacija:**
- [1] `cryptsetup` / LUKS — priručničke stranice `cryptsetup(8)` i `crypttab(5)` na sustavu te službeni projekt, dostupno na: https://gitlab.com/cryptsetup/cryptsetup (pristupljeno 6.6.2026.).
- [2] Microsoft Learn: *Overview of managed disk encryption options*, dostupno na: https://learn.microsoft.com/azure/virtual-machines/disk-encryption-overview (pristupljeno 6.6.2026.).
- [3] Microsoft Learn: *Add a data disk to a Linux VM*, dostupno na: https://learn.microsoft.com/azure/virtual-machines/linux/add-disk (pristupljeno 6.6.2026.).
- [4] Rocky Linux / Red Hat Enterprise Linux 9: dokumentacija o šifriranju blok uređaja (LUKS), dostupno na: https://docs.redhat.com/ (pristupljeno 6.6.2026.).

**Izjava o uporabi AI alata:**
- **Korišteni alat:** AI asistent (Claude, Anthropic).
- **Svrha i opseg uporabe:** pomoć pri strukturiranju tehničkog izvješća, objašnjenju sintakse naprednih naredbi (`cryptsetup`, `crypttab`, `fstab`) i provjeri formulacija. AI nije zamijenio vlastitu izvedbu.
- **Kako je provjerena točnost:** sve naredbe i konfiguracije izvršili smo i provjerili na vlastitom Azure VM-u; rezultati (sirovi ispisi, kontrolne sume, ponašanje pri ponovnom pokretanju, dokaz enkripcije) potvrđeni su praktično i nalaze se u ovom izvješću.

## 9. Izjava o samostalnosti i izvornosti rada

Ovom predajom potvrđujemo da je opisano rješenje rezultat našeg vlastitog rada te da su svi korišteni izvori, tuđa
pomoć, programski kod, konfiguracije i AI alati navedeni u referencama. Razumijemo da se u slučaju timskog rada
ocjenjuje individualni doprinos svakog člana tima te da svaki član mora samostalno obrazložiti cjelokupno rješenje
tijekom obrane.

Filip Kušer · Patrik Ostrunić
