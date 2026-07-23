---
title: "Použití Managed Service Accounts pro SQL Server"
date: "2025-03-05"
tags: [sMSA, gMSA, SQL Server, Active Directory]
layout: post
---

## Použití Managed Service Accounts pro SQL Server

Spravované servisní účty odstraňují ruční správu hesel služeb. Pro jeden samostatný SQL Server můžeme použít **standalone Managed Service Account (sMSA)**. Pro více serverů, Always On Availability Groups nebo failover cluster instance použijeme **group Managed Service Account (gMSA)**.

---

### 1. Požadavky a volba účtu

* SQL Server musí běžet na doménově připojeném Windows Serveru.
* gMSA pro SQL Server vyžaduje Windows Server 2012 R2 nebo novější.
* SQL Server podporuje gMSA od SQL Serveru 2014 pro samostatné instance a od SQL Serveru 2016 pro failover cluster instance a Availability Groups.
* sMSA je vázaný na jediný počítač a není vhodný pro cluster.
* Pro službu Database Engine a SQL Server Agent je vhodné použít samostatné účty.

Příklady používají doménu `MOJEDOMENA`, server `SQL01` a skupinu `SQL-Servers-gMSA`. Názvy upravíme podle prostředí.

📌 **sMSA volíme pro jednu službu na jednom serveru. gMSA použijeme všude, kde stejný účet potřebuje více serverů nebo cluster.**

---

### 2. Varianta A: sMSA pro jeden SQL Server

Na počítači s modulem **ActiveDirectory** vytvoříme účet omezený na jeden server a přiřadíme jej počítači:

```powershell
New-ADServiceAccount -Name SQLMSA -RestrictToSingleComputer
Add-ADComputerServiceAccount -Identity SQL01 -ServiceAccount SQLMSA
```

KDS root key pro sMSA nepotřebujeme.

Na serveru `SQL01` doinstalujeme modul Active Directory, účet nainstalujeme a otestujeme:

```powershell
Install-WindowsFeature RSAT-AD-PowerShell
Install-ADServiceAccount -Identity SQLMSA
Test-ADServiceAccount -Identity SQLMSA
```

✅ Výsledek posledního příkazu musí být `True`.

---

### 3. Varianta B: gMSA pro více SQL serverů

#### 3.1 KDS root key

gMSA vyžaduje KDS root key. Jeho existenci ověříme na doménovém řadiči:

```powershell
Get-KdsRootKey
```

Pokud neexistuje, v produkční doméně jej vytvoříme a před prvním použitím vyčkáme alespoň deset hodin na replikaci:

```powershell
Add-KdsRootKey -EffectiveImmediately
```

Pouze v jediné DC testovací doméně lze čekání obejít zpětným datováním klíče:

```powershell
Add-KdsRootKey -EffectiveTime ((Get-Date).AddHours(-10))
```

⚠️ **Zpětné datování nepoužíváme v produkční doméně s více doménovými řadiči.**

#### 3.2 Skupina serverů a účet

Do bezpečnostní skupiny přidáme **počítačové účty** všech SQL serverů:

```powershell
New-ADGroup -Name "SQL-Servers-gMSA" -GroupScope Global -GroupCategory Security
Add-ADGroupMember -Identity "SQL-Servers-gMSA" -Members "SQL01$","SQL02$"
```

Potom vytvoříme gMSA:

```powershell
New-ADServiceAccount `
  -Name SQLgMSA `
  -DNSHostName SQLgMSA.mojedomena.local `
  -KerberosEncryptionType AES128,AES256 `
  -PrincipalsAllowedToRetrieveManagedPassword "SQL-Servers-gMSA"
```

📌 Příklad povoluje pouze AES. Nejdřív ověříme, že řadiče domény i podporované servery používají moderní Kerberos; DES nepovolujeme a závislost na RC4 raději odstraníme, než abychom ji bez ověření zachovali.

Po novém přidání počítače do skupiny je nejjistější cílový server restartovat, aby počítač získal aktuální členství. Na každém SQL serveru pak spustíme:

```powershell
Install-WindowsFeature RSAT-AD-PowerShell
Install-ADServiceAccount -Identity SQLgMSA
Test-ADServiceAccount -Identity SQLgMSA
```

✅ Výsledek musí být `True`.

---

### 4. Nastavení služby SQL Server

Účet služby měníme výhradně přes **SQL Server Configuration Manager**, ne přes `services.msc`. Configuration Manager upraví také potřebná místní oprávnění, ochranu Service Master Key a související nastavení služby.

1. Otevřeme **SQL Server Configuration Manager**.
2. Přejdeme do **SQL Server Services**.
3. Otevřeme vlastnosti služby **SQL Server (MSSQLSERVER)** nebo příslušné pojmenované instance.
4. Na kartě **Log On** zadáme `MOJEDOMENA\SQLMSA$` nebo `MOJEDOMENA\SQLgMSA$`.
5. Pole pro heslo necháme prázdná.
6. Změnu potvrdíme a službu restartujeme.

Stejným způsobem lze nastavit SQL Server Agent, ideálně pod jiným sMSA/gMSA.

📌 Servisní účet nepřidáváme jako samostatný login do SQL Serveru a neudělujeme mu `sysadmin`. SQL Server používá pro interní oprávnění per-service SID (`NT SERVICE\MSSQLSERVER`, případně SID pojmenované instance), který instalace už správně vytvořila.

---

### 5. Ověření

V SQL Server Configuration Manageru musí být služba ve stavu **Running** pod očekávaným účtem. Účet ověříme také v SSMS:

```sql
SELECT
    servicename,
    service_account,
    status_desc,
    startup_type_desc
FROM sys.dm_server_services;
```

Pro SQL Server 2019 a starší vyžaduje tento pohled oprávnění `VIEW SERVER STATE`; pro SQL Server 2022 a novější `VIEW SERVER SECURITY STATE`.

Pokud klienti používají Kerberos, ověříme registraci SPN a skutečný autentizační mechanismus:

```powershell
setspn -L MOJEDOMENA\SQLgMSA$
setspn -Q MSSQLSvc/sql01.mojedomena.local:1433
```

```sql
SELECT auth_scheme
FROM sys.dm_exec_connections
WHERE session_id = @@SPID;
```

U vzdáleného spojení má `auth_scheme` vrátit `KERBEROS`. Duplicitní nebo chybějící SPN nejdřív opravíme v Active Directory; neřešíme je přidáním servisního účtu do `sysadmin`.

---

### Shrnutí

✅ sMSA používáme pro samostatnou službu na jednom serveru.  
✅ gMSA používáme pro více serverů, FCI a Availability Groups.  
✅ KDS root key je nutný pro gMSA, nikoli pro sMSA.  
✅ Účet služby měníme přes SQL Server Configuration Manager.  
❌ Servisnímu účtu nevytváříme SQL login s rolí `sysadmin`.  
✅ Funkčnost ověřujeme přes `Test-ADServiceAccount`, `sys.dm_server_services` a podle potřeby SPN.

### Zdroje

* [Configure Windows service accounts and permissions](https://learn.microsoft.com/en-us/sql/database-engine/configure-windows/configure-windows-service-accounts-and-permissions)
* [Manage group Managed Service Accounts](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/group-managed-service-accounts/group-managed-service-accounts/manage-group-managed-service-accounts)
* [sys.dm_server_services](https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-server-services-transact-sql)
