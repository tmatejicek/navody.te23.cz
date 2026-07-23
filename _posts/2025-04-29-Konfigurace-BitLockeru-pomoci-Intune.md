---
title: "Konfigurace BitLockeru pomocí Intune"
date: 2025-04-29
tags: [Microsoft Intune, BitLocker, Security]
layout: post
---

## Konfigurace BitLockeru pomocí Intune

Tento postup nastaví tiché zapnutí BitLockeru na spravovaných počítačích s Windows a uloží obnovovací klíče do Microsoft Entra ID. Názvy položek odpovídají profilu **BitLocker** v Endpoint security.

⚠️ **Windows 10 dosáhl konce běžné podpory 14. října 2025.** Nové nasazení plánujeme na podporovaných verzích Windows 11; Windows 10 používáme jen v rámci odpovídajícího programu ESU.

---

### 1. Předpoklady

Před nasazením ověříme:

- zařízení je **Microsoft Entra joined** nebo **Microsoft Entra hybrid joined** a je spravováno v Intune,
- edice Windows podporuje BitLocker,
- počítač používá UEFI se zapnutým Secure Boot, má funkční TPM 1.2 nebo novější a zapnuté Windows Recovery Environment,
- v prostředí není jiný produkt pro šifrování disku,
- helpdesk má oprávnění zobrazit obnovovací klíč a existuje ověřený postup obnovy.

Na testovacím zařízení spustíme jako správce:

```powershell
Get-Tpm
Confirm-SecureBootUEFI
reagentc /info
dsregcmd /status
```

U hybridně připojeného zařízení musí `dsregcmd /status` uvádět `AzureAdJoined : YES` a `DomainJoined : YES`. Pokud není Secure Boot zapnutý, zařízení nesplňuje podmínky pro tiché zapnutí BitLockeru pomocí Intune. Nejprve opravíme konfiguraci firmware a způsob spouštění systému, případně použijeme řízené nasazení, které vyžaduje interakci uživatele.

---

### 2. Vytvoření politiky

1. Otevřeme [Microsoft Intune admin center](https://intune.microsoft.com/).
2. Přejdeme na **Endpoint security > Disk encryption**.
3. Zvolíme **Create Policy**.
4. Nastavíme **Platform: Windows** a **Profile: BitLocker**.
5. Profil pojmenujeme například `Windows - BitLocker - Silent`.

📌 Pro spolehlivé tiché zapnutí používáme profil **BitLocker** v Endpoint security, nikoli obecný Settings catalog. Settings catalog neobsahuje všechny ovládací prvky TPM potřebné pro tento scénář.

---

### 3. Nastavení tichého šifrování

V části **Configuration settings** nastavíme:

| Nastavení | Hodnota |
|---|---|
| **Require Device Encryption** | `Enabled` |
| **Allow Warning For Other Disk Encryption** | `Disabled` |
| **Allow Standard User Encryption** | `Enabled` |

`Allow Standard User Encryption` se zobrazí po vypnutí varování před jiným šifrováním. Je nutné, pokud je přihlášený uživatel standardní uživatel.

⚠️ **Vypnutí Allow Warning For Other Disk Encryption potlačí kontrolní dialog.** Před nasazením proto inventarizujeme a odinstalujeme všechny šifrovací produkty třetích stran; jinak hrozí poškození systému nebo ztráta dat.

#### 3.1 Ověření pomocí TPM bez interakce uživatele

V části **Operating System Drives** nastavíme:

| Nastavení | Hodnota |
|---|---|
| **Require additional authentication at startup** | `Enabled` |
| **Configure TPM startup** | `Allow TPM` nebo `Require TPM` |
| **Configure TPM startup PIN** | `Do not allow startup PIN with TPM` |
| **Configure TPM startup key** | `Do not allow startup key with TPM` |
| **Configure TPM startup key and PIN** | `Do not allow startup key and PIN with TPM` |

📌 PIN nebo USB startup key vyžaduje zásah uživatele a tiché zapnutí zablokuje. Stejná nastavení zkontrolujeme i v bezpečnostních baseline a v doménových GPO, aby se politiky nepřepisovaly.

#### 3.2 Metoda šifrování

Pro nové šifrování nastavíme:

| Nastavení | Hodnota |
|---|---|
| **Encryption Method For Operating System Drives** | `XTS-AES 256-bit` |
| **Encryption Method For Fixed Data Drives** | `XTS-AES 256-bit` |

XTS je režim určený pro šifrování disků a AES-256 odpovídá konzervativnímu bezpečnostnímu profilu. Aktuální doporučení NÚKIB připouští AES-128, AES-192 i AES-256 a pro disky doporučuje režim XTS nebo EME2; pro nová nasazení preferujeme 256 bitů.

📌 Změna metody se použije až při novém zašifrování. Již zašifrovaný disk kvůli samotné změně algoritmu bez důvodu nedešifrujeme a znovu nešifrujeme.

---

### 4. Obnovovací klíče

V nastavení obnovy pro systémový disk vyžadujeme 48místné recovery password a zálohu do Microsoft Entra ID:

| Nastavení | Hodnota |
|---|---|
| **Save BitLocker recovery information to Microsoft Entra ID** | `Enabled` |
| **Store recovery information in Microsoft Entra ID before enabling BitLocker** | `Required` |
| **Client-driven recovery password rotation** | `Enable rotation on Microsoft Entra ID and hybrid joined devices` |

📌 Pokud profil používá starší názvy voleb, nastavíme jejich významový ekvivalent. Zásadní je, aby šifrování nezačalo před úspěšným uložením recovery password do Entra ID.

Po použití obnovovacího klíče jej otočíme v Intune: **Devices > All devices > zařízení > BitLocker key rotation**. Klíče ověříme přes **Devices > All devices > zařízení > Recovery keys**. Zobrazení klíče se zapisuje do auditního logu.

---

### 5. Přiřazení a pilot

1. Politiku nejprve přiřadíme malé **skupině zařízení**.
2. Ověříme různé modely zařízení, BIOS/UEFI a scénář hybrid join.
3. Zkontrolujeme, že každé zašifrované zařízení má v Entra ID obnovovací klíč.
4. Teprve poté rozšíříme přiřazení na produkční skupiny.

⚠️ **Nevytváříme současně více profilů BitLocker se stejnými nastaveními.** Konflikt s GPO, security baseline nebo jinou MDM politikou patří mezi nejčastější příčiny selhání tichého šifrování.

---

### 6. Kontrola nasazení

Centrální stav otevřeme v **Devices > Monitor > Encryption report**. Na konkrétním počítači ověříme:

```powershell
Get-BitLockerVolume
manage-bde -status C:
manage-bde -protectors -get C:
```

Kontrolujeme zejména:

- `Protection Status: Protection On`,
- `Conversion Status: Fully Encrypted` nebo záměrně `Used Space Only Encrypted`,
- použitou metodu XTS-AES,
- přítomnost `Numerical Password` protectoru,
- shodné ID recovery protectoru v zařízení a v Entra ID.

Chyby klienta hledáme v **Event Viewer > Applications and Services Logs > Microsoft > Windows > BitLocker-API > Management** a ve stavu profilu daného zařízení v Intune.

---

### 7. Compliance a Conditional Access

⚠️ Compliance policy s požadavkem na šifrované úložiště nasadíme až po dokončeném pilotu. Pokud compliance ihned propojíme s Conditional Access, může chybná nebo konfliktní politika BitLocker zablokovat přístup uživatelů ještě před dokončením šifrování.

Nejdříve používáme report-only nebo dostatečně dlouhou lhůtu pro nápravu, ověříme recovery klíče a až potom přístup skutečně vynucujeme.

---

### Shrnutí

✅ Používáme profil **Endpoint security > Disk encryption > BitLocker**.  
✅ Před tichým zapnutím ověříme UEFI, Secure Boot, TPM, WinRE a stav Entra join.  
✅ Obnovovací heslo uložíme do Microsoft Entra ID ještě před zahájením šifrování.  
✅ Nasazení nejprve ověříme na malé skupině různých zařízení.  
⚠️ Compliance a Conditional Access vynucujeme až po úspěšném pilotu a ověření recovery klíčů.

### Zdroje

- [Encrypt Windows devices with BitLocker using Intune](https://learn.microsoft.com/en-us/intune/device-configuration/endpoint-security/encrypt-bitlocker-windows)
- [Disk encryption policy for endpoint security in Intune](https://learn.microsoft.com/en-us/intune/device-configuration/endpoint-security/disk-encryption)
- [Troubleshooting BitLocker policies from the client side](https://learn.microsoft.com/en-us/troubleshoot/mem/intune/device-protection/troubleshoot-bitlocker-policies)
- [Minimální požadavky na kryptografické algoritmy a jejich parametry, NÚKIB](https://portal.nukib.gov.cz/informacni-servis/podpurne-materialy/69a82264c32c1c948f0686a4)
