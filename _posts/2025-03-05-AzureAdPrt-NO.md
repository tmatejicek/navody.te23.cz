---
title: "Jak řešit AzureAdPrt = NO u Microsoft Entra hybrid join"
date: "2025-03-05"
tags: [Microsoft Entra ID, Hybrid Join, PRT, Troubleshooting]
layout: post
---

## Jak řešit AzureAdPrt = NO u Microsoft Entra hybrid join

`AzureAdPrt` popisuje stav **Primary Refresh Tokenu přihlášeného uživatele**. Není to stav registrace počítače. Zařízení proto může být správně hybridně připojené a konkrétní uživatel přesto nemusí mít PRT. Stav **Pending** v Microsoft Entra ID naopak znamená, že synchronizovaný objekt zařízení ještě nedokončil registraci.

Tyto dva problémy je potřeba diagnostikovat odděleně.

📌 **Nejdřív vždy rozlišíme, zda řešíme registraci zařízení, nebo PRT konkrétního uživatele.**

---

### 1. Rozlišení stavu zařízení a uživatele

Nejdřív otevřeme běžný, **nepovýšený** příkazový řádek v relaci uživatele, kterému nefunguje SSO:

```powershell
whoami
dsregcmd /status
```

V části **Device State** očekáváme u dokončeného hybridního připojení:

```text
AzureAdJoined : YES
DomainJoined  : YES
```

V části **SSO State** kontrolujeme uživatelský token:

```text
AzureAdPrt : YES
```

Pokud je `AzureAdJoined : NO`, řešíme registraci zařízení. Pokud je `AzureAdJoined : YES`, ale `AzureAdPrt : NO`, řešíme získání PRT konkrétního uživatele.

📌 `NgcSet : NO` pouze znamená, že uživatel nemá vytvořený klíč Windows Hello for Business. Samo o sobě to není důkaz chyby PRT a kvůli diagnostice PRT není vhodné Windows Hello vypínat.

---

### 2. Diagnostika chybějícího PRT

U `AzureAdPrt : NO` zkontrolujeme ve stejné části výstupu zejména:

* `Previous Prt Attempt`
* `Attempt Status`
* `User Identity`
* `HTTP Error` a `HTTP status`
* `Server Error Code` a `Server Error Description`
* `Correlation ID`

Pokud je `AzureAdPrtUpdateTime` starší než čtyři hodiny, zamkneme a odemkneme relaci uživatele a stav zkontrolujeme znovu:

```powershell
rundll32.exe user32.dll,LockWorkStation
```

Podrobnosti k pokusu o získání PRT najdeme v **Event Viewer**:

```text
Applications and Services Logs
└─ Microsoft
   └─ Windows
      └─ AAD
         ├─ Operational
         └─ Analytic
```

Událost `1006` označuje začátek pokusu, `1007` jeho výsledek. Analytický log je nutné v Event Vieweru nejdřív zobrazit a povolit přes **View > Show Analytic and Debug Logs**. Podle `Correlation ID` a času pak dohledáme odpovídající přihlášení také v **Microsoft Entra admin center > Identity > Monitoring & health > Sign-in logs**.

📌 Úspěšné přihlášení na webový portál pouze ověřuje interaktivní přihlášení. Neprokazuje, že klient získal PRT ani že jej neblokuje Conditional Access.

---

### 3. Diagnostika registrace zařízení a stavu Pending

Pro stav zařízení otevřeme příkazový řádek **jako správce** a spustíme:

```powershell
dsregcmd /status
```

Kromě **Device State** zkontrolujeme část **Diagnostic Data**, zejména fázi registrace, klientský a serverový chybový kód a případnou URL s diagnostikou. Události registrace jsou zde:

```text
Applications and Services Logs
└─ Microsoft
   └─ Windows
      └─ User Device Registration
         └─ Admin
```

Ověříme také, že:

* objekt počítače je v OU synchronizované pomocí Microsoft Entra Connect Sync,
* je správně nakonfigurovaný Service Connection Point (SCP),
* replikace Active Directory a synchronizace do Entra ID probíhají bez chyby,
* zařízení dosáhne na registrační služby Microsoftu.

---

### 4. Kontrola sítě a proxy

Minimálně ověříme DNS a TCP 443:

```powershell
Resolve-DnsName login.microsoftonline.com
Test-NetConnection login.microsoftonline.com -Port 443
Test-NetConnection device.login.microsoftonline.com -Port 443
Test-NetConnection enterpriseregistration.windows.net -Port 443
```

Stav systémové WinHTTP proxy zobrazíme bez změny konfigurace:

```powershell
netsh winhttp show proxy
```

PRT a registrace zařízení používají také systémový kontext. Proxy, TLS inspekce a firewall proto musí fungovat nejen v prohlížeči uživatele. Přesný seznam povolených adres vždy převezmeme z aktuální dokumentace Microsoft 365 a Microsoft Entra.

⚠️ **Příkaz `netsh winhttp reset proxy` nespouštíme naslepo.** V organizaci se spravovanou proxy může odstranit platné nastavení a způsobit další výpadky.

---

### 5. Opakování automatické registrace

Po odstranění příčiny můžeme na hybridně připojovaném počítači spustit vestavěnou úlohu:

```powershell
Start-ScheduledTask -TaskPath "\Microsoft\Windows\Workplace Join\" -TaskName "Automatic-Device-Join"
```

Výsledek znovu ověříme pomocí `dsregcmd /status` a logu **User Device Registration > Admin**.

`dsregcmd /leave` použijeme až jako řízený poslední krok, například u prokazatelně zastaralé registrace po přesunu objektu mimo synchronizační rozsah:

```powershell
dsregcmd /leave
Restart-Computer
```

Po restartu necháme hybridní registraci znovu provést úlohou **Automatic-Device-Join**. Hybridně připojený počítač nepřipojujeme ručně přes **Access work or school > Connect**, protože tím nevytváříme správně řízený Microsoft Entra hybrid join.

---

### Shrnutí

✅ `AzureAdPrt` je stav přihlášeného uživatele, nikoli objektu zařízení.  
✅ PRT kontrolujeme v nepovýšené relaci postiženého uživatele.  
✅ `NgcSet : NO` neznamená poruchu PRT.  
✅ Stav Pending řešíme přes diagnostiku registrace, synchronizaci a úlohu `Automatic-Device-Join`.  
✅ Proxy před změnou nejdřív zkontrolujeme a zařízení ručně nepřevádíme na jiný typ připojení.

### Zdroje

* [Troubleshoot Microsoft Entra hybrid joined devices](https://learn.microsoft.com/en-us/entra/identity/devices/troubleshoot-hybrid-join-windows-current)
* [Troubleshoot primary refresh token issues on Windows devices](https://learn.microsoft.com/en-us/entra/identity/devices/troubleshoot-primary-refresh-token)
* [Pending devices in Microsoft Entra ID](https://learn.microsoft.com/en-us/troubleshoot/entra/entra-id/dir-dmns-obj/pending-devices)
