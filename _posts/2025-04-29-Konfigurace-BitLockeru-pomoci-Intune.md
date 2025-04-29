---
title: "Konfigurace BitLockeru pomocí Intune"
date: 2025-04-29
tags: ["Microsoft Intune", "Encryption", "Security"]
layout: post
---

## Konfigurace šifrování pevných disků pomocí Intune

Tento návod popisuje, jak nastavit a spravovat šifrování pevných disků na zařízeních Windows 10/11 pomocí Microsoft Intune. Cílem je zajistit ochranu dat na discích automatizovanou a centralizovanou cestou.

---

### 1. Příprava prostředí

Než začnete, ověřte následující:

- ✅ Zařízení je registrováno v Microsoft Intune
- ✅ Windows verze podporuje BitLocker (Pro, Enterprise, Education)
- ✅ Uživatelé mají příslušné licence (např. Microsoft 365 E3/E5)

### 2. Vytvoření konfiguračního profilu přes Šifrování disku

#### 2.1 Vytvoření nové politiky

- Přihlaste se do [Microsoft Intune Admin Center](https://intune.microsoft.com)
- Přejděte na **Endpoint security > Disk encryption > Create Policy**
- Nastavte:
  - Platforma: **Windows 10 and later**
  - Profil: **BitLocker**

#### 2.2 Konfigurace hlavních nastavení

V sekci nastavení politiky nakonfigurujte:

- **Encrypt devices**: `Require`
- **BitLocker base settings**:
  - **Require device encryption for operating system drives**: `Yes`
  - **Require device encryption for fixed drives**: `Yes`

**Použité šifrovací algoritmy:**

- Ve výchozím nastavení Intune se pro šifrování pevných disků používá algoritmus **AES 128bit XTS**.
- Národní údrad pro kybernetickou a informační bezpečnost (NUKIB) v materiálu [Minimální požadavky na kryptografické prostředky v4](https://nukib.gov.cz/download/publikace/podpurne_materialy/Minimalni_pozadavky_v4_FINAL.pdf) (kapitola 2.f)) doporučuje pro šifrování disků algoritmy **XTS** nebo **EME2**.

```powershell
# Ukázka Powershellu pro ruční kontrolu stavů BitLockeru
Get-BitLockerVolume
```

### 3. Přiřazení politiky uživatelům

#### 3.1 Definování skupin

- V sekci **Assignments** vyberte skupiny zařízení nebo uživatelů
- 📌 Doporučené: Použít dynamické skupiny pro automatizaci

#### 3.2 Testovací nasazení

- ⚠ Nejdříve otestujte na omezené sadě zařízení
- Sledujte nasazení v **Endpoint security > Disk encryption**

### 4. Správa a kontrola stavů

#### 4.1 Monitorování BitLockeru

- Použijte sestavy v Intune: **Reports > Endpoint security > Disk encryption**
- Pravidelně sledujte:
  - Šifrované vs. nešifrované disky
  - Chybové hlášení při nasazování

#### 4.2 Vymáhání compliance

- Vytvořte compliance policy s kontrolou šifrování
- Nastavte akce při nevyhovění, např. zablokování přístupu k datům

```powershell
# Ukázka kontroly compliance stavu
Get-IntuneCompliancePolicyDeviceSettingState
```

---

### 5. Doporučené postupy a časté chyby

#### 5.1 Doporučené postupy

- 📌 Vždy ukládejte obnovovací klíče do Azure AD (Entra ID)
- 📌 Před šifrováním ověřte, že TPM funguje a je povolen v BIOS/UEFI
- 📌 Používejte "silent enablement", pokud je možné

#### 5.2 Časté chyby

- ⚠ Nepovolený TPM či chybně nakonfigurovaná BIOS/UEFI nastavení
- ⚠ Neuložené obnovovací klíče mohou znemožnit obnovení dat
- ⚠ Nedostatečné testování politiky před masovým nasazením

---

## Shrnutí

✅ Příprava zařízení pro BitLocker šifrování
✅ Vytvoření politiky šifrování v Endpoint Security
✅ Ukládání obnovovacích klíčů do Azure AD (Entra ID)
✅ Správné přiřazení a testovací nasazení
✅ Kontrola stavů a enforcement compliance politik
✅ Dodržování doporučených postupů a vyvarování se častým chybám
