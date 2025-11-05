---
title: "Detekce nešifrovaných LDAP dotazů v prostředí Active Directory"
date: 2025-05-15
tags: [LDAP, Active Directory, Security]
layout: post
---

## Detekce nešifrovaných LDAP dotazů v prostředí Active Directory

Tento návod popisuje, jak v prostředí s Active Directory zjistit, zda dochází k nešifrovaným LDAP dotazům. Cílem je poskytnout přehled metod detekce a postup pro aktivaci logování, které pomáhá vyhodnotit rizika spojená s nezabezpečenou komunikací.

---

### 1. Úvod do problému

LDAP (Lightweight Directory Access Protocol) je často používán bez zabezpečení na portu 389. To může vést k odposlechu nebo úpravě dat v síti.

* Nešifrovaný LDAP umožňuje přenášet citlivé informace v čitelné podobě ⚠
* Pro bezpečný provoz doporučujeme LDAPS (port 636) nebo LDAP over StartTLS

---

### 2. Události v Event Logu

#### 2.1 Základní monitorování (Event ID 2887)

* Každých 24 hodin
* Informuje o tom, že došlo k nešifrovaným LDAP dotazům
* Neobsahuje IP adresy

#### 2.2 Detailní přehled (Event ID 2888, 2889)

* Vyžaduje zapnutí rozšířeného logování
* Obsahuje IP adresy klientů
* Pomáhá identifikovat konkrétní zařízení 📌

#### 2.3 Zablokované dotazy (Event ID 2890)

* Zaznamenává pokusy o nešifrované připojení, které bylo odmítnuto
* Aktivní při nastavení `LDAPServerIntegrity = 2`

#### 2.4 Varování o slabé konfiguraci (Event ID 2886)

* Upozorňuje, že řadič domény nepodporuje LDAP signing ani sealing ⚠
* Indikátor špatně zabezpečené konfigurace

---

### 3. Zapnutí diagnostického logování

#### 3.1 Změna registru lokálně

```powershell
reg add "HKLM\SYSTEM\CurrentControlSet\Services\NTDS\Diagnostics" /v "16 LDAP Interface Events" /t REG_DWORD /d 2 /f
```

* Změny se obvykle projeví bez restartu, ale někdy je vhodné restartovat DC

#### 3.2 Nasazení pomocí GPO (přes Preferences)

* Používáme Group Policy Preferences > Windows Settings > Registry
* Přidáme klíče do `HKLM\SYSTEM\CurrentControlSet\Services\NTDS\Diagnostics`
* GPO aplikujeme na OU s řadiči domény

---

### 4. Vyhodnocení událostí pomocí PowerShellu

#### 4.1 Export unikátních IP, uživatelů a typů spojení z eventů 2889

```powershell
$ComputerName = "localhost"
$Hours = 24

$InsecureLDAPBinds = @()

$StartTime = (Get-Date).AddHours(-$Hours)
$Events = Get-WinEvent -ComputerName $ComputerName -FilterHashtable @{
    LogName = 'Directory Service'
    Id       = 2889
    StartTime = $StartTime
}

foreach ($Event in $Events) {
    try {
        $eventXml = [xml]$Event.ToXml()
        $EventData = $eventXml.Event.EventData.Data

        $ClientInfo = $EventData[0]
        $IPAddress = $ClientInfo.Substring(0, $ClientInfo.LastIndexOf(":"))
        $User = $EventData[1]
        $BindTypeCode = [int]$EventData[2]

        switch ($BindTypeCode) {
            0 { $BindType = "Unsigned" }
            1 { $BindType = "Simple" }
            default { $BindType = "Unknown" }
        }

        $Row = [PSCustomObject]@{
            IPAddress = $IPAddress
            User      = $User
            BindType  = $BindType
        }

        $InsecureLDAPBinds += $Row
    }
    catch {
        Write-Warning "Chyba při zpracování události: $_"
    }
}

$InsecureLDAPBinds | Sort-Object IPAddress, User, BindType -Unique | ft
```

---

### 5. Prevence nešifrovaného LDAP

#### 5.1 Vynucení LDAP signing

```powershell
reg add "HKLM\SYSTEM\CurrentControlSet\Services\NTDS\Parameters" /v "LDAPServerIntegrity" /t REG_DWORD /d 2 /f
```

* Všechny klienty je nutné předem otestovat ✅

---

## Shrnutí

✅ Nešifrovaný LDAP představuje bezpečnostní riziko  
✅ Události 2887–2890 poskytují různé úrovně informací  
✅ Detailní logování lze zapnout přes registr nebo GPO  
✅ PowerShell pomáhá identifikovat konkrétní zařízení  
✅ Nastavení LDAPServerIntegrity=2 zvyšuje bezpečnost, ale může přerušit kompatibilitu
