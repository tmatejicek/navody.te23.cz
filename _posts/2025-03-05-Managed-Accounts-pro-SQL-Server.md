---
title: "Použití Managed Service Accounts pro SQL Server"
date: "2025-03-05"
tags: [MSA, gMSA, SQL Server, Active Directory]
layout: post
---

## Použití Managed Service Accounts pro SQL Server

Managed Service Accounts (MSA) a Group Managed Service Accounts (gMSA) jsou moderní způsob, jak zabezpečit SQL Server instance bez nutnosti spravovat hesla. V tomto návodu si ukážeme, jak je správně nastavit a použít.

---

## 1. Začneme ověřením prostředí

Než se pustíme do konfigurace, ujistíme se, že splňujeme následující podmínky:
- Používáme **Windows Server 2012 nebo novější**.
- SQL Server běží na **doménově připojeném serveru**.
- Máme přístup k **Active Directory** a oprávnění k vytváření spravovaných účtů.
- Pokud plánujeme **Group Managed Service Account (gMSA)**, máme **doménový řadič Windows Server 2012 nebo novější**.

---

## 2. Vytvoření Managed Service Account (MSA)

Nejprve vytvoříme klasický Managed Service Account (MSA), který lze použít pro jeden SQL Server.

### 2.1 Vytvoříme MSA v Active Directory

Na **doménovém řadiči** otevřeme PowerShell jako administrátor a spustíme příkaz:

```powershell
New-ADServiceAccount -Name SQLMSA -DNSHostName sqlserver.mojedomena.local -PrincipalsAllowedToRetrieveManagedPassword sqlserver$
```

📌 **Co děláme?**
- `SQLMSA` je název účtu, který použijeme pro SQL Server.
- `-DNSHostName` odpovídá názvu serveru, kde bude SQL Server běžet.
- `-PrincipalsAllowedToRetrieveManagedPassword` definuje, který server může tento účet používat.

### 2.2 Nainstalujeme MSA na SQL Server

Na **SQL Serveru**, kde poběží databázová instance, spustíme PowerShell:

```powershell
Install-ADServiceAccount -Identity SQLMSA
```

Poté ověříme, zda je účet správně nainstalován:

```powershell
Test-ADServiceAccount -Identity SQLMSA
```

Pokud se vrátí **True**, účet je připraven.

---

## 3. Nastavení SQL Serveru pro použití MSA

### 3.1 Změníme účet služby SQL Server

1. Otevřeme **SQL Server Configuration Manager**.
2. Vybereme **SQL Server Services**.
3. Klikneme pravým tlačítkem na **SQL Server (MSSQLSERVER)** a vybereme **Properties**.
4. Na kartě **Log On** zadáme účet **`DOMAIN\SQLMSA$`** (nezapomeneme na `$` na konci!).
5. Nevyplňujeme heslo – MSA ho spravuje automaticky.
6. Uložíme změny a **restartujeme SQL Server**.

### 3.2 Přidáme oprávnění v SQL Serveru

Přihlásíme se do SQL Server Management Studia (SSMS) a spustíme následující příkaz:

```sql
CREATE LOGIN [DOMAIN\SQLMSA$] FROM WINDOWS;
ALTER SERVER ROLE sysadmin ADD MEMBER [DOMAIN\SQLMSA$];
```

Tím zajistíme, že SQL Server bude mít potřebná oprávnění.

---

## 4. Použití Group Managed Service Account (gMSA)

Pokud plánujeme používat **gMSA pro více SQL serverů**, postupujeme takto:

### 4.1 Vytvoříme gMSA v Active Directory

Na doménovém řadiči spustíme PowerShell:

```powershell
New-ADServiceAccount -Name SQLgMSA -DNSHostName sqlserver.mojedomena.local -PrincipalsAllowedToRetrieveManagedPassword "SQLServersGroup"
```

📌 **Co děláme?**
- `SQLgMSA` je název skupinového účtu.
- `-PrincipalsAllowedToRetrieveManagedPassword` definuje doménovou skupinu (`SQLServersGroup`), která obsahuje všechny SQL servery, jež budou tento účet používat.

### 4.2 Nainstalujeme gMSA na SQL Server

Na **každém SQL Serveru**, který bude tento účet používat, spustíme:

```powershell
Install-ADServiceAccount -Identity SQLgMSA
Test-ADServiceAccount -Identity SQLgMSA
```

Výstup **True** znamená, že účet je funkční.

### 4.3 Přidáme gMSA jako účet služby SQL Server

Stejně jako u MSA:
- V **SQL Server Configuration Manageru** nastavíme službu SQL Server na běh pod účtem **`DOMAIN\SQLgMSA$`**.
- Nevyplňujeme heslo.
- Restartujeme SQL Server.

### 4.4 Přidáme oprávnění v SQL Serveru

V SSMS spustíme:

```sql
CREATE LOGIN [DOMAIN\SQLgMSA$] FROM WINDOWS;
ALTER SERVER ROLE sysadmin ADD MEMBER [DOMAIN\SQLgMSA$];
```

---

## 5. Ověříme funkčnost

Nakonec ověříme, že SQL Server běží pod správným účtem. Spustíme v SSMS:

```sql
SELECT SUSER_NAME();
```

Měli bychom vidět výstup **`DOMAIN\SQLMSA$`** nebo **`DOMAIN\SQLgMSA$`**.

---

## Shrnutí

✅ **MSA** je vhodné pro **jednotlivé SQL servery**.  
✅ **gMSA** umožňuje **sdílet jeden účet mezi více SQL servery**.  
✅ **Odpadá nutnost správy hesel** – systém je mění automaticky.  
✅ **Zabezpečení se zvyšuje**, protože přihlašovací údaje nejsou nikde uloženy ručně.  

Tímto jsme úspěšně nastavili SQL Server pro běh pod Managed Service Accounts! 🎯
