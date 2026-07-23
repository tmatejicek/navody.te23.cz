---
title: "Rotace hesla účtu krbtgt v Active Directory"
date: 2025-09-04
tags: [Active Directory, Security, Kerberos]
layout: post
---

## Rotace hesla účtu krbtgt v Active Directory

Účet `krbtgt` chrání klíče používané pro Kerberos Ticket Granting Tickets. Aby se odstranil původní klíč i jeho předchozí verze, heslo se resetuje **dvakrát** s dostatečnou prodlevou a s ověřením replikace po každém kroku.

📌 **Každá doména má vlastní účet `krbtgt`.** Postup opakujeme samostatně pro každou doménu ve forestu. Netýká se účtů `krbtgt_<číslo>` patřících jednotlivým RODC.

---

### 1. Kdy rotaci provést

Rotaci plánujeme:

- pravidelně podle bezpečnostní politiky; Microsoft Defender for Identity označuje heslo starší než 180 dní,
- po podezření na kompromitaci doménových oprávnění nebo krádež hashů,
- po obnově domény podle forest recovery postupu.

⚠️ Při aktivním incidentu nejdříve odstraníme přístup útočníka a opravíme zdroj kompromitace. Jinak může nové klíče získat znovu. Reset `krbtgt` sám nenahrazuje kompletní obnovu důvěry v doménu.

---

### 2. Příprava

Rotaci provedeme v servisním okně z privilegované administrační stanice. Nejdříve ověříme:

```powershell
repadmin /replsummary
repadmin /showrepl * /errorsonly
dcdiag /e /q
```

❌ Pokud je zapisovatelný DC dlouhodobě offline, má chyby replikace nebo existují lingering objects, nepokračujeme. Nezreplikovaný DC by po návratu používal zastaralé klíče a mohl by způsobovat chyby vydávání nebo ověřování ticketů; server za hranicí tombstone lifetime nevracíme do domény bez podporovaného postupu.

Zjistíme PDC Emulator a výchozí stav:

```powershell
Import-Module ActiveDirectory

$Domain = Get-ADDomain
$Pdc = $Domain.PDCEmulator

Get-ADUser -Server $Pdc -Identity krbtgt `
    -Properties PasswordLastSet, msDS-KeyVersionNumber |
    Select-Object DistinguishedName, PasswordLastSet, msDS-KeyVersionNumber
```

Hodnotu `msDS-KeyVersionNumber` a čas si uložíme do záznamu změny.

#### 2.1 Určení čekací doby

V doménové GPO zkontrolujeme:

**Computer Configuration > Policies > Windows Settings > Security Settings > Account Policies > Kerberos Policy**

Zajímají nás zejména:

- **Maximum lifetime for user ticket**,
- **Maximum lifetime for service ticket**.

📌 Mezi resetem číslo 1 a 2 čekáme déle než vyšší z těchto hodnot. Výchozí hodnota je 10 hodin, ale `10 hodin` není univerzální konstanta; při vlastní Kerberos politice použijeme její skutečné maximum a přidáme provozní rezervu.

---

### 3. První reset

Hodnota hesla zadaná při resetu není pro `krbtgt` významná: AD DS nezávisle vygeneruje silné heslo. Cmdlet přesto parametr vyžaduje, proto zadáme libovolnou hodnotu splňující místní password filter.

```powershell
$PlaceholderPassword = Read-Host `
    'Zadejte jednorázovou komplexní hodnotu pro první reset' `
    -AsSecureString

Set-ADAccountPassword `
    -Server $Pdc `
    -Identity krbtgt `
    -Reset `
    -NewPassword $PlaceholderPassword

Remove-Variable PlaceholderPassword
```

Volitelně urychlíme replikaci a následně ji vždy ověříme:

```powershell
repadmin /syncall $Pdc /AdeP
repadmin /replsummary
repadmin /showrepl * /errorsonly
```

Na všech zapisovatelných DC musí být stejná nová hodnota KVNO a `PasswordLastSet`:

```powershell
Get-ADDomainController -Filter * |
    Where-Object { -not $_.IsReadOnly } |
    Sort-Object HostName |
    ForEach-Object {
        try {
            $Krbtgt = Get-ADUser `
                -Server $_.HostName `
                -Identity krbtgt `
                -Properties PasswordLastSet, msDS-KeyVersionNumber

            [pscustomobject]@{
                DomainController = $_.HostName
                PasswordLastSet  = $Krbtgt.PasswordLastSet
                KeyVersion       = $Krbtgt.'msDS-KeyVersionNumber'
            }
        }
        catch {
            [pscustomobject]@{
                DomainController = $_.HostName
                PasswordLastSet  = $null
                KeyVersion       = "CHYBA: $($_.Exception.Message)"
            }
        }
    } | Format-Table -AutoSize
```

⚠️ **Dokud nejsou všechny zapisovatelné DC konzistentní, nezačínáme odpočítávat čekací dobu a druhý reset neprovádíme.**

---

### 4. Čekání a průběžná kontrola

Vyčkáme dobu určenou v kapitole 2.1. První reset ponechá původní klíč jako předchozí, takže běžné tikety vydané před změnou mohou doběhnout. Příliš rychlý druhý reset by odstranil klíč, kterým jsou ještě podepsané platné uživatelské a servisní tikety, a mohl by vyvolat plošné chyby autentizace.

Během čekání sledujeme zejména Security události `4768`, `4769`, `4770` a `4771`, autentizační chyby služeb a stav replikace.

---

### 5. Druhý reset

Po uplynutí čekací doby a nové kontrole replikace provedeme druhý reset na stejném PDC Emulatoru:

```powershell
$PlaceholderPassword = Read-Host `
    'Zadejte jednorázovou komplexní hodnotu pro druhý reset' `
    -AsSecureString

Set-ADAccountPassword `
    -Server $Pdc `
    -Identity krbtgt `
    -Reset `
    -NewPassword $PlaceholderPassword

Remove-Variable PlaceholderPassword

repadmin /syncall $Pdc /AdeP
repadmin /replsummary
repadmin /showrepl * /errorsonly
```

✅ Znovu spustíme přehled KVNO z kapitoly 3. Oproti výchozímu stavu se musí hodnota na každém zapisovatelném DC zvýšit o dvě.

---

### 6. Ověření provozu

Na testovacích klientech můžeme vyčistit tikety a vynutit nové přihlášení:

```powershell
klist purge
klist get krbtgt
```

Ověříme přihlášení, přístup k souborovým službám, aplikace s Kerberos SSO, plánované úlohy a služby používající doménové identity. Pokračujeme ve sledování událostí `4768`, `4769` a `4771`.

Uživatelé mohou být po druhém resetu vyzváni k novému ověření. Samotná přítomnost několika chyb starého ticketu krátce po změně není důvodem vracet klíč; důležité je, zda klient po novém přihlášení získá platný TGT.

---

### 7. Evidence a pravidelný proces

Do evidence uložíme:

- doménu, PDC a časy obou resetů,
- KVNO před změnou, po prvním a po druhém resetu,
- výsledky replikace a provozních testů,
- použitou čekací dobu odvozenou z Kerberos policy.

⚠️ Rotaci automatizujeme pouze skriptem, který před každým krokem kontroluje replikaci a vynucuje bezpečnou prodlevu. Jednoduchá naplánovaná dvojice příkazů bez kontrol není bezpečný proces.

---

### Shrnutí

✅ Heslo `krbtgt` resetujeme dvakrát pro každou doménu samostatně.  
✅ Před každým resetem i po něm kontrolujeme zdraví a replikaci všech zapisovatelných DC.  
✅ Mezi resety čekáme déle než skutečné maximum platnosti Kerberos ticketů.  
✅ Po druhém resetu musí být KVNO na všech zapisovatelných DC o dvě vyšší.  
❌ Dva resety bez bezpečné prodlevy a kontroly replikace nejsou bezpečný postup.

### Zdroje

- [Active Directory forest recovery: Reset the krbtgt password](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/forest-recovery-guide/ad-forest-recovery-reset-the-krbtgt-password)
- [Microsoft Defender for Identity: Change password for krbtgt account](https://learn.microsoft.com/en-us/defender-for-identity/security-posture-assessments/accounts#change-password-for-krbtgt-account)
