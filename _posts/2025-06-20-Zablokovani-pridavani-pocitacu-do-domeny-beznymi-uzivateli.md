---
title: "Omezení přidávání počítačů do domény běžnými uživateli"
date: 2025-06-20
tags: [Active Directory, Security, ms-DS-MachineAccountQuota]
layout: post
---

## Omezení přidávání počítačů do domény běžnými uživateli

Ve výchozím nastavení může ověřený uživatel vytvořit v doméně až deset počítačových účtů. Nastavením `ms-DS-MachineAccountQuota` na `0` tuto výchozí možnost vypneme a přidávání zařízení přesuneme na řízený provisioning.

⚠️ **Změna kvóty sama o sobě neruší explicitně delegovaná práva.** Účet s oprávněním **Create Computer objects** na konkrétní OU může počítačový účet vytvořit i při kvótě `0`.

---

### 1. Jak oprávnění k domain join funguje

Je potřeba rozlišit tři situace:

1. **Vytvoření nového účtu přes výchozí kvótu.** Uživatel využije právo `SeMachineAccountPrivilege` (**Add workstations to domain**) a omezuje ho `ms-DS-MachineAccountQuota`, ve výchozím stavu `10`.
2. **Vytvoření nového účtu pomocí delegace na OU.** Oprávnění na kontejneru se řídí ACL a kvótu obchází.
3. **Opětovné použití předem vytvořeného účtu.** Vedle práv k objektu se uplatňuje kontrola vlastníka zavedená aktualizacemi KB5020276.

📌 Možnost vytvořit doménový počítač není sama o sobě automatickou eskalací na správce domény, ale poskytuje útočníkovi další doménovou identitu a rozšiřuje prostor pro zneužití chybných delegací a Kerberos konfigurace.

---

### 2. Kontrola současného stavu

Spustíme jako účet s právem číst doménový objekt:

```powershell
Import-Module ActiveDirectory

$Domain = Get-ADDomain
Get-ADObject `
    -Identity $Domain.DistinguishedName `
    -Properties 'ms-DS-MachineAccountQuota' |
    Select-Object DistinguishedName, 'ms-DS-MachineAccountQuota'
```

Také zkontrolujeme, komu jsou na OU pro počítače delegována práva, a GPO pro řadiče domény:

**Computer Configuration > Policies > Windows Settings > Security Settings > Local Policies > User Rights Assignment > Add workstations to domain**

Kvóta a `SeMachineAccountPrivilege` tvoří jednu cestu. Explicitní delegace na OU je samostatná cesta.

#### 2.1 Přehled účtů vytvořených přes kvótu

`mS-DS-CreatorSID` pomáhá dohledat počítačové účty, u nichž AD eviduje tvůrce:

```powershell
Get-ADComputer -LDAPFilter '(mS-DS-CreatorSID=*)' -Properties mS-DS-CreatorSID |
    ForEach-Object {
        $Sid = [System.Security.Principal.SecurityIdentifier]::new(
            $_.'mS-DS-CreatorSID', 0
        )
        try {
            $Creator = $Sid.Translate(
                [System.Security.Principal.NTAccount]
            ).Value
        }
        catch {
            $Creator = $Sid.Value
        }

        [pscustomobject]@{
            Computer = $_.Name
            Creator  = $Creator
            DN       = $_.DistinguishedName
        }
    } |
    Sort-Object Creator, Computer
```

⚠️ Výsledek před změnou projdeme a neznámé nebo nepoužívané účty ověříme; nemažeme je automaticky.

---

### 3. Nastavení kvóty na nulu

```powershell
Import-Module ActiveDirectory

$Domain = Get-ADDomain
Set-ADDomain `
    -Identity $Domain.DistinguishedName `
    -Replace @{'ms-DS-MachineAccountQuota' = 0}
```

Hodnotu ověříme na PDC Emulatoru a po replikaci také na ostatních DC:

```powershell
$Domain = Get-ADDomain
Get-ADObject `
    -Server $Domain.PDCEmulator `
    -Identity $Domain.DistinguishedName `
    -Properties 'ms-DS-MachineAccountQuota' |
    Select-Object -ExpandProperty 'ms-DS-MachineAccountQuota'

repadmin /replsummary
```

✅ Očekávaný výsledek je `0`. Existující počítačové účty se změnou neodstraní a již připojené počítače z domény nevypadnou.

---

### 4. Řízený způsob přidávání počítačů

Vytvoříme například skupinu `GG-DomainJoin-Workstations` a delegujeme jí práva pouze na OU `OU=Workstations`.

❌ Provisioning účtu nedáváme členství v **Domain Admins**.

Preferované varianty jsou:

- provisioning systém vytvoří účet počítače ve správné OU a provede join,
- stejný důvěryhodný účet počítačový objekt vytvoří i použije,
- použijeme offline domain join (`djoin.exe`), který na cílovém zařízení nevyžaduje oprávnění vytvářet objekt v AD.

Při ruční delegaci se řídíme požadovanými právy pro vytvoření nebo opětovné použití účtu z dokumentace Microsoftu. Samotné **Create Computer objects** nemusí stačit pro rejoin existujícího objektu; ten může vyžadovat mimo jiné reset hesla, validované zápisy DNS hostname a SPN a zápis account restrictions.

#### 4.1 Opětovné použití existujícího účtu

Aktualizované systémy blokují reuse účtu, pokud jeho vlastník není považován za důvěryhodného. Typická chyba je:

```text
0xaac (2732): NERR_AccountReuseBlockedByPolicy
An account with the same name exists in Active Directory.
Re-using the account was blocked by security policy.
```

Pokud provisioning účty předem vytváří jiná důvěryhodná služba, nastavíme na všech DC GPO:

**Computer Configuration > Policies > Windows Settings > Security Settings > Local Policies > Security Options > Domain controller: Allow computer account re-use during domain join**

⚠️ Do allowlistu přidáme jen skupiny důvěryhodných **vlastníků/tvůrců** počítačových účtů, nikoli účet, který pouze provádí join, a nikdy `Authenticated Users` nebo `Everyone`. Registr `ComputerAccountReuseAllowList` ručně neupravujeme.

---

### 5. Test a troubleshooting

Po replikaci provedeme dva testy:

1. Běžný uživatel bez delegace se pokusí přidat počítač s novým názvem. Vytvoření musí selhat.
2. Schválený provisioning postup vytvoří účet ve správné OU a join musí projít.

Při chybě zkontrolujeme na klientovi:

```text
C:\Windows\debug\NetSetup.log
```

Události povoleného nebo blokovaného reuse jsou v systémovém logu klienta jako Netjoin `4100` a `4101`; související kontroly allowlistu se zapisují také do systémového logu řadiče domény.

---

### Shrnutí

✅ `ms-DS-MachineAccountQuota = 0` vypne výchozí možnost běžných uživatelů vytvářet počítačové účty.  
✅ Explicitní delegace na určené OU zůstává funkční a používáme ji pro řízený provisioning.  
✅ Před změnou zkontrolujeme `mS-DS-CreatorSID`, delegace na OU a právo **Add workstations to domain**.  
✅ Po změně otestujeme zamítnutý join běžného uživatele i schválený provisioning.  
⚠️ Reuse existujícího účtu nastavujeme podle KB5020276 a allowlist nikdy neotevíráme všem uživatelům.

### Zdroje

- [Active Directory domain join permissions](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/active-directory-domain-join-permissions)
- [Default limit to number of workstations a user can join to the domain](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/default-workstation-numbers-join-domain)
- [KB5020276: Netjoin domain join hardening changes](https://support.microsoft.com/en-us/topic/kb5020276-netjoin-domain-join-hardening-changes-2b65a0f3-1f4c-42ef-ac0f-1caaf421baf8)
