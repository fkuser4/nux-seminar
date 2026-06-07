# PREZENTACIJA — tekst za lijepljenje u službeni .pptx predložak
> Najviše 4 slajda · 5 min izlaganja + 1–2 min pitanja · prikazati barem jedan ključni dokaz.

---

## SLAJD 1 — Naslov, cilj, ključni rezultat

**Naslov:** Enkripcija diska — Azure disk enkripcija i LUKS zaštita podataka
**Projektna tema 16** · Napredna UNIX rješenja · 2025./2026.

**Autori i odgovornosti**
- Filip Kušer — Azure okruženje + LUKS konfiguracija
- Patrik Ostrunić — ispitivanje + dokumentacija

**Cilj (1–2 rečenice):**
Zaštititi podatke u mirovanju na dodatnom Azure disku izradom LUKS šifriranog volumena čiji podaci bez ispravnog
ključa nisu čitljivi, te ga učiniti upotrebljivim i trajnim kroz ponovna pokretanja.

**Jedan ključni rezultat:**
Nakon ponovnog pokretanja volumen se automatski otključa i `sha256sum -c` vraća **OK** (podaci nepromijenjeni);
na sirovom disku (tip `crypto_LUKS`) jedinstveni marker se **ne može pronaći**.

---

## SLAJD 2 — Arhitektura sustava i mrežna topologija

**Postavka:** 1 VM (Rocky Linux 9.6, Azure, Switzerland North, B2ats_v2, Trusted Launch) + 1 dodatni disk 4 GiB.

**Dijagram (tekstualni):**
```
Administrator (SSH, port 22)
        |
   Azure VNet 10.0.0.0/16  +  NSG: dopušten samo 22
        |
   VM1: NUX26-16-vm1  (10.0.0.4 / javni 51.107.91.110)
        |-- OS disk  /dev/sdX  (XFS)              -> Azure SSE (platformski)
        +-- Dodatni disk /dev/disk/azure/scsi1/lun0 (4 GiB) -> LUKS šifrirani volumen
```

**Mrežni podaci**
| VM / resurs | Uloga | Privatni IP | Portovi |
|---|---|---|---|
| NUX26-16-vm1 | šifrirana pohrana podataka | 10.0.0.4 | 22 (SSH) |
| NUX26-16-nsg | mrežna zaštita | — | dopušta samo 22 |

**Obvezno objasniti:** jedan čvor, samo SSH promet (22), dodatni disk je šifrirani LUKS volumen; dokaz da radi =
`sha256sum -c` OK nakon reboota.

---

## SLAJD 3 — Implementacija i ključna konfiguracija

**4 ključna koraka:**
1. **Priprema** — SSH, `cryptsetup`, identifikacija diska preko stabilne putanje `/dev/disk/azure/scsi1/lun0`.
2. **LUKS2 izrada** — `cryptsetup luksFormat` (aes-xts-plain64, argon2id, `--pbkdf-memory 262144` zbog 1 GiB RAM-a).
3. **Otključavanje + FS** — `luksOpen securedata` → `mkfs.xfs` → `mount /mnt/secure`.
4. **Trajnost** — keyfile + `/etc/crypttab` (preko **LUKS UUID-a**) + `/etc/fstab`, opcija `nofail`.

**Ključni ispis (revizijski trag):**
```
$ lsblk -f
sdX          crypto_LUKS 2   4fce498f-...
└─securedata xfs               8ce7c164-...   /mnt/secure
$ cryptsetup luksDump  →  LUKS2, aes-xts-plain64, 512-bit, argon2id, 2 keyslota
```

---

## SLAJD 4 — Ispitivanje, dokaz i zaključak

**Plan ispitivanja (svih 5 provjera = OK):**
| I | Provjera | Status |
|---|---|---|
| I1 | LUKS2 header valjan | OK |
| I2 | tip: crypto_LUKS / xfs | OK |
| I3 | otključavanje + mount + upis | OK |
| I4 | reboot + integritet (`sha256sum -c`) | OK |
| I5 | enkripcija (marker nije na sirovom) | OK |

**Ključni dokaz:** `sha256sum -c /root/sums.before` → oba **OK** nakon reboota; marker **PASS** (nije na sirovom uređaju).

**Poteškoća → rješenje:** automatsko otključavanje na headless VM-u + zamjena imena diska (`sda↔sdb`) nakon reboota
→ riješeno **keyfileom**, **`nofail`** i korištenjem **LUKS UUID-a** (ne imena uređaja).

**Zaključak:** ručni LUKS = ista `dm-crypt`/LUKS tehnologija koju Azure Disk Encryption koristi na Linuxu; pouzdana
zaštita podataka u mirovanju. Ograničenje: keyfile na OS disku (u produkciji → Key Vault / TPM).

**Azure troškovi:** VM dealociran nakon rada; resursi se brišu nakon obrane.

**Doprinos:** Filip — Azure + LUKS; Patrik — ispitivanje + dokumentacija. Oba člana odgovaraju na pitanja.
