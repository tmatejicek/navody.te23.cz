---
title: "Přidání confidential atributu do Active Directory"
date: 2025-04-29
layout: post
tags: [ActiveDirectory, Schema, PowerShell, Security]
---

## Jak správně přidat vlastní atribut do Active Directory a umožnit řízený přístup

V tomto návodu si ukážeme, jak krok za krokem přidat vlastní atribut do Active Directory, správně ho připojit ke třídě objektů a nastavit oprávnění ke čtení a zápisu. Cílem je mít plnou kontrolu nad tím, kdo může nový atribut vidět a upravovat.

---

## 1. Vytvoření vlastního atributu v AD schématu

### 1.1 Co je potřeba připravit
- 📌 Členství v **Schema Admins**.
- 📌 Aktivní **povolení úprav schématu** (`Schema Update Allowed` v registru).
- 📌 Validní **OID** pro nový atribut.

#### Nastavení Schema Update Allowed v registru
Pro povolení úprav schématu je nutné:

- Spustit `regedit`.
- Přejít na klíč:

```
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\NTDS\Parameters
```

- Vytvořit nový `DWORD (32-bit)` záznam s názvem:

```
Schema Update Allowed
```

- Nastavit jeho hodnotu na `1`.
- Restartovat ADSI Edit nebo správu schématu (není potřeba restart serveru).

⚠ Po dokončení změn v schématu je doporučeno tento klíč opět odebrat nebo nastavit na `0`.

### 1.2 Definice atributu
Použij hodnoty:
- `attributeSyntax: 2.5.5.12` (Unicode string)
- `oMSyntax: 64`
- `isSingleValued: TRUE`
- `searchFlags: 128` (CONFIDENTIAL)

Příklad v LDIF formátu:

```ldif
dn: CN=confidentialAttribute,CN=Schema,CN=Configuration,DC=firma,DC=cz
changetype: add
objectClass: attributeSchema
attributeID: 1.2.840.113556.1.8000.2554.28570892.20250403.1
attributeSyntax: 2.5.5.12
oMSyntax: 64
isSingleValued: TRUE
searchFlags: 128
lDAPDisplayName: confidentialAttribute
adminDisplayName: Confidential Attribute
adminDescription: Tajný atribut pro řízený přístup
```

### 1.3 Kde a jak atribut přidat
Atribut lze přidat dvěma způsoby:

- ⚙ **Použití nástroje ADSI Edit**:
  - Připojit se na kontext **Schema**.
  - Kliknout pravým tlačítkem na `CN=Attributes` > **New > Object**.
  - Vybrat `attributeSchema` a vyplnit potřebné vlastnosti.

- 📄 **Import přes LDIF soubor**:
  - Připravit LDIF soubor podle ukázky.
  - Spustit příkaz:

```powershell
ldifde -i -f cesta\k\souboru.ldf -k -j .
```

Před změnami vždy doporučujeme zálohovat aktuální schéma a testovat v izolovaném prostředí. ⚠ Změny schématu jsou trvalé a nelze je později vrátit.

---

## 2. Připojení atributu ke třídě objektu

### 2.1 Význam `mayContain`
- 📌 `mayContain` definuje, které atributy **mohou být** přítomné u objektů dané třídy.
- ⚠ Pokud atribut nepřidáš do `mayContain`, nebude možné jeho hodnotu uložit.

### 2.2 Přidání do třídy `user`
Postup v ADSI Edit:
- Otevři `CN=User,CN=Schema,CN=Configuration,...`
- Najdi vlastnost `mayContain`.
- Přidej `confidentialAttribute` do seznamu.

---

## 3. Nastavení oprávnění ke čtení a zápisu

### 3.1 Přístup přes ACL
- 📌 Oprávnění se nastavuje na **objekty nebo OU**, ne na samotný atribut ve schématu.

### 3.2 Nastavení oprávnění na celou doménu
Použij nástroj `dsacls`:

```powershell
dsacls "DC=firma,DC=cz" /I:T /G "DOMENA\HR Team:RPWP;confidentialAttribute;user"
```

- `RPWP` znamená **Read Property** a **Write Property**.
- Dědičnost (`/I:T`) zajistí propagaci na všechny podřízené objekty.

✅ **Poznámka**: `RP` (Read Property) umožňuje pouze čtení, `WP` (Write Property) umožňuje zapisovat hodnotu atributu. Pokud chceš obě možnosti, použij kombinaci `RPWP`.

---

## 4. Testování čtení a zápisu

### 4.1 Skript pro čtení atributu

```powershell
$user = Get-ADUser -Identity "testuser" -Properties confidentialAttribute
$user | Select-Object SamAccountName, confidentialAttribute
```

✅ Pokud máš práva, uvidíš hodnotu atributu.

### 4.2 Skript pro zápis atributu

**Pozor:** `Set-ADUser` standardně nepodporuje zápis vlastních atributů, pokud nejsou zahrnuty v oficiálních AD cmdletech. Proto je nutné zapisovat přes ADSI rozhraní.

```powershell
$dn = (Get-ADUser "testuser").DistinguishedName
$user = [ADSI]"LDAP://$dn"
$user.Put("confidentialAttribute", "Tajná hodnota")
$user.SetInfo()
```

✅ Pokud máš práva, hodnota se uloží.

---

## Shrnutí

✅ Přidání vlastního atributu do AD vyžaduje úpravu schématu  
✅ Povolení úprav schématu se nastavuje přes registr pomocí klíče `Schema Update Allowed`  
✅ Atribut musí být připojen k objektové třídě pomocí `mayContain`  
✅ Práva na atribut nastavujeme pomocí ACL na objektech, ne ve schématu  
✅ Pro zápis vlastních atributů je nutné používat ADSI, ne standardní cmdlety  
✅ Před jakoukoliv změnou schématu je nutné mít zálohu a testovat v izolovaném prostředí  
✅ Testování čtení a zápisu ověří správné nastavení práv

