---
title: "Jak řešit problém s Azure AD: AzureAdPrt = NO"
date: "2025-03-05"
tags: [Azure AD, Hybrid Join, PRT, Troubleshooting]
layout: post
---
# Jak řešit problém s Azure AD: AzureAdPrt = NO

Při hybridním připojení zařízení k Azure AD se může stát, že některé počítače zůstanou ve stavu **"Čekající" (Pending)** a registrace neproběhne úspěšně. Jedním z klíčových indikátorů tohoto problému je hodnota **AzureAdPrt = NO**, která znamená, že zařízení nemá platný **Primary Refresh Token (PRT)**. Bez něj se uživatelé nemohou správně ověřovat a využívat funkce Azure AD. V tomto článku si ukážeme, jak tento problém diagnostikovat a vyřešit.

---

## 1. Ověřte, zda je uživatel přihlášen správným účtem
Zařízení musí být přihlášeno doménovým účtem, který je synchronizován s Azure AD. Pokud je uživatel přihlášen **místním účtem**, PRT nebude vydán.

### Jak zkontrolovat účet:
1. Otevřete **Příkazový řádek (cmd) jako správce**.
2. Spusťte příkaz:
   ```powershell
   whoami
   ```
3. Pokud výstup **neobsahuje** `DOMAIN\username`, je uživatel přihlášen místním účtem.

**Řešení:** Odhlaste uživatele a přihlaste se firemním účtem, který je synchronizován s Azure AD.

---

## 2. Ověřte, zda organizace neblokuje přihlášení do Azure AD
Pokud jsou v **Azure AD Conditional Access** nastaveny restriktivní politiky, mohou blokovat vydávání PRT.

### Jak to ověřit:
1. Otevřete **prohlížeč Edge nebo Chrome**.
2. Přihlaste se na **https://myaccount.microsoft.com** firemním účtem.
3. Pokud přihlášení selže, je pravděpodobně **blokováno politikou**.

**Řešení:** Zkontrolujte nastavení **Conditional Access v Azure AD** a ujistěte se, že zařízení mají povolený přístup.

---

## 3. Ověřte konektivitu k Azure AD
Zařízení musí mít přístup k následujícím službám:
- `login.microsoftonline.com`
- `device.login.microsoftonline.com`
- `enterpriseregistration.windows.net`

### Jak zkontrolovat konektivitu:
1. Otevřete **Příkazový řádek jako administrátor**.
2. Spusťte test DNS:
   ```powershell
   nslookup login.microsoftonline.com
   ```
3. Ověřte připojení:
   ```powershell
   Test-NetConnection login.microsoftonline.com -Port 443
   ```

**Řešení:** Pokud testy selžou, upravte **firewall nebo proxy**, aby umožňovaly přístup k těmto doménám.

---

## 4. Zkontrolujte Windows Hello for Business
Pokud je Windows Hello for Business povolen, ale není správně nakonfigurován, může to bránit vydávání PRT.

### Jak zjistit stav Windows Hello:
1. Otevřete **PowerShell jako správce**.
2. Spusťte:
   ```powershell
   dsregcmd /status
   ```
3. Hledejte řádek **"NgcSet"**. Pokud je **NO**, Windows Hello není správně nastaveno.

**Řešení:** Ověřte **Group Policy Management Console (GPMC)**:
- `Computer Configuration > Policies > Administrative Templates > Windows Components > Windows Hello for Business`
- Dočasně zkuste deaktivovat Windows Hello for Business a otestujte PRT.

---

## 5. Resetujte síťové nastavení a proxy
Nevhodně nastavená proxy může blokovat komunikaci s Azure AD.

### Jak resetovat proxy:
1. Otevřete **PowerShell jako administrátor**.
2. Spusťte:
   ```powershell
   netsh winhttp reset proxy
   ```
3. Restartujte **službu Netlogon**:
   ```powershell
   net stop netlogon
   net start netlogon
   ```
4. Restartujte zařízení a znovu ověřte PRT.

---

## 6. Odstranění a nové připojení zařízení k Azure AD
Pokud žádné z výše uvedených řešení nefunguje, zkuste zařízení **odregistrovat a znovu připojit**.

### Jak odregistrovat zařízení:
1. Otevřete **PowerShell jako administrátor**.
2. Spusťte příkaz:
   ```powershell
   dsregcmd /leave
   ```
3. Restartujte zařízení.

### Jak znovu připojit zařízení:
1. Otevřete **Settings > Accounts > Access work or school**.
2. Klikněte na **Connect** a připojte zařízení k Azure AD.

---

## Shrnutí
Pokud vaše zařízení v Azure AD zůstává ve stavu **"Čekající"** a hodnota **AzureAdPrt = NO**, zkuste následující kroky:
✅ Přihlaste se doménovým účtem, ne místním účtem.  
✅ Ověřte, zda nejsou blokovány Azure AD endpointy.  
✅ Zkontrolujte **Conditional Access** politiky.  
✅ Resetujte proxy a firewallová pravidla.  
✅ Pokud nic nepomůže, odregistrovat a znovu připojit zařízení.  

Tento postup vám pomůže vyřešit problém a úspěšně registrovat zařízení v **Hybrid Azure AD Join**. Pokud problém přetrvává, je možné, že je nutná hlubší diagnostika pomocí **Azure AD logs a Event Viewer**.
