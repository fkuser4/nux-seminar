# PRIPREMA ZA USMENU OBRANU (NE predaje se — samo za vas)
> Oba člana moraju moći objasniti CIJELO rješenje. Filip naglasak na Azure+LUKS, Patrik na ispitivanju, ali znajte oboje.

## Najvjerojatnija pitanja i kratki odgovori

**1. Što je LUKS, a što dm-crypt?**
`dm-crypt` je kernelski mehanizam (device-mapper) koji transparentno šifrira/dešifrira blok uređaj. **LUKS** je
standardni format zaglavlja iznad dm-crypta koji upravlja ključevima (više keyslotova, metapodaci, KDF). `cryptsetup`
je alat kojim to konfiguriramo.

**2. Razlika SSE / ADE / encryption-at-host?**
- **SSE** — platformsko šifriranje svih managed diskova u mirovanju (AES-256), uvijek uključeno; ključ drži platforma (PMK) ili CMK u Key Vaultu.
- **ADE (Azure Disk Encryption)** — enkripcija na razini gosta: `dm-crypt`/LUKS (Linux) ili BitLocker (Windows), integrirano s Key Vaultom.
- **Encryption at host** — šifriranje na samom hostu koji izvodi VM (uklj. privremene diskove/cache).
- Mi smo **ručno** napravili LUKS → ista tehnologija koju ADE koristi na Linuxu.

**3. Zašto `aes-xts-plain64`?**
XTS je mod AES-a dizajniran baš za šifriranje diskova. 512-bitni ključ znači AES-256 (XTS koristi dva pod-ključa).
To je zadana, preporučena LUKS2 postavka.

**4. Što je u `/etc/crypttab` i `/etc/fstab`, zašto `nofail`?**
- `crypttab`: ime mappera (`securedata`), izvor preko **LUKS UUID-a**, datoteka-ključ, opcije `luks,nofail`. Govori systemd-u kako otključati volumen pri bootu.
- `fstab`: što montirati (`/dev/mapper/securedata`), gdje (`/mnt/secure`), tip `xfs`, opcije `nofail`.
- `nofail`: ako otključavanje/montiranje ne uspije, **boot se ne zaustavlja** (ključno na headless VM-u bez konzole).

**5. Zašto keyfile i koji je sigurnosni kompromis?**
Headless VM nema konzolu za upis zaporke pri bootu → koristimo nasumičnu **datoteku-ključ** za automatsko
otključavanje. Kompromis: keyfile leži na OS disku (čita ga root); tko dobije OS disk + keyfile, može otključati
podatkovni disk. Ublažavanje: OS disk je šifriran Azure SSE-om; u produkciji bi ključ išao u **Key Vault / TPM**.

**6. Što se dogodi bez ključa?**
Volumen se ne može otključati; sirovi uređaj je `crypto_LUKS` i ne može se montirati; dokazali smo da se marker
**ne nalazi** na sirovom disku → podaci su nečitljivi.

**7. Zašto UUID umjesto imena diska (`sda`)?**
Imena uređaja nisu stabilna — nakon reboota su nam se `sda`/`sdb` **zamijenili**. **LUKS UUID** je trajan, pa
otključavanje uvijek pogađa pravi uređaj. (To smo i demonstrirali.)

**8. Što dokazuje `sha256sum -c`?**
Da su datoteke nakon reboota bit-identične (ista kontrolna suma) → podaci su prošli kroz šifrirani sloj
**nepromijenjeni** i trajno su dostupni.

**9. Zašto `--pbkdf-memory 262144`?**
LUKS2 `argon2id` po defaultu troši puno RAM-a; VM ima samo **1 GiB**. Ograničili smo na 256 MiB da formatiranje i
otključavanje (i pri bootu) ne ostanu bez memorije. (Isti razlog zašto smo za dokaz koristili `strings`, a ne `grep`.)

**10. Zašto ručni LUKS, a ne gotov ADE?**
Da demonstriramo sam mehanizam (isti koji ADE koristi), zadržimo punu kontrolu nad ključem i izbjegnemo ADE
ograničenja (podržane slike/veličine, ovisnost o Key Vaultu). ADE objašnjavamo teorijski, kako tema traži.

## Demo „na brzinu" (ako traže da pokažete)
```
cryptsetup status securedata          # aktivan LUKS2 volumen
lsblk -f                              # crypto_LUKS -> securedata (xfs)
sudo sha256sum -c /root/sums.before   # OK = integritet
```
