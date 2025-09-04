---
title: "Rotace hesla účtu krbtgt v Active Directory"
date: 2025-09-04
tags: [Active Directory, Security, Kerberos]
layout: post
---

## Rotace hesla účtu krbtgt v Active Directory

V tomto článku si projdeme praktický postup bezpečné rotace hesla účtu **krbtgt** v prostředí Active Directory. Cílem je zneplatnit případné falešné Kerberos lístky (Golden Ticket) a zároveň předejít výpadkům autentizace.

---

### 1. Proč rotovat heslo krbtgt

* Účet krbtgt slouží jako klíč pro podepisování všech Kerberos lístků.
* Kompromitace tohoto účtu umožňuje útočníkovi vydávat si falešné TGT.
* 📌 Pravidelná rotace zvyšuje celkovou bezpečnost domény.

---

### 2. Kontrola stavu účtu

#### 2.1 Zobrazení verze klíče

Jednoduše ověříme aktuální hodnotu verze klíče:

```powershell
Get-ADUser krbtgt -Property msDS-KeyVersionNumber
```

#### 2.2 Kontrola replikace

* Ověříme, zda všechny DC replikují správně.
* ⚠ Problémy s replikací mohou způsobit výpadky ověřování.

Jednoduchý příkaz pro kontrolu replikace:

```powershell
repadmin /replsummary
```

---

### 3. První změna hesla

#### 3.1 Vygenerování nového hesla

Použijeme bezpečné náhodné heslo založené na GUID:

```powershell
$NewPassword = ConvertTo-SecureString -AsPlainText -Force -String (New-Guid).Guid
Set-ADAccountPassword -Identity krbtgt -NewPassword $NewPassword -Reset
```

#### 3.2 Ověření změny

Zkontrolujeme, že se verze klíče zvýšila o 1:

```powershell
Get-ADUser krbtgt -Property msDS-KeyVersionNumber
```

---

### 4. Čekací doba

* Standardní platnost TGT je přibližně 10 hodin.
* Vyčkáme alespoň 10 hodin, než provedeme druhou změnu.
* Během této doby staré lístky přirozeně expirují.

---

### 5. Druhá změna hesla

#### 5.1 Vytvoření dalšího klíče

Nastavíme nové náhodné heslo pro druhou rotaci:

```powershell
$NewPassword2 = ConvertTo-SecureString -AsPlainText -Force -String (New-Guid).Guid
Set-ADAccountPassword -Identity krbtgt -NewPassword $NewPassword2 -Reset
```

#### 5.2 Ověření

Ověříme, že se verze klíče zvýšila znovu o 1:

```powershell
Get-ADUser krbtgt -Property msDS-KeyVersionNumber
```

---

### 6. Ověření funkčnosti

* Provedeme test přihlášení uživatele k doménovému počítači.
* Zkontrolujeme logy v Event Vieweru (ID 4769, 4770).
* 📌 Spustíme replikaci mezi DC pro urychlení doručení změn:

```powershell
repadmin /syncall /AdeP
```

---

## Shrnutí

✅ Rotace hesla krbtgt chrání proti Golden Ticket útokům  
✅ Postup probíhá ve dvou krocích s čekací dobou alespoň 10 hodin  
✅ Každá změna zvyšuje msDS-KeyVersionNumber o 1  
✅ Po dokončení ověříme přihlášení, logy a replikaci  
✅ Doporučujeme cyklus rotace minimálně jednou ročně  
