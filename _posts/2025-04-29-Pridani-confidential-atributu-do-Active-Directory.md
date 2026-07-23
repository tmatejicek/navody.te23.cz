---
title: "Přidání důvěrného atributu do Active Directory"
date: 2025-04-29
layout: post
tags: [Active Directory, Schema, PowerShell, Security]
---

## Přidání důvěrného atributu do Active Directory

Tento postup vytvoří vlastní jednohodnotový textový atribut, připojí jej ke třídě `user` a deleguje čtení jen určené skupině. Změna schématu je lesní a prakticky nevratná, proto ji nejprve ověříme v testovacím forestu.

⚠️ **Příznak `CONFIDENTIAL` hodnotu nešifruje.** Mění kontrolu oprávnění při čtení: vedle `READ_PROPERTY` je potřeba také objektově specifické `CONTROL_ACCESS`. Správci s širokými právy mohou hodnotu nadále přečíst.

---

### 1. Příprava

Před změnou:

1. Ověříme replikaci pomocí `repadmin /replsummary` a `dcdiag /e /q`.
2. Vytvoříme aktuální System State backup alespoň jednoho zapisovatelného řadiče domény.
3. Přihlásíme se na Schema Master účtem dočasně přidaným do **Schema Admins**.
4. Po přidání do **Schema Admins** se odhlásíme a znovu přihlásíme, aby bylo členství v přístupovém tokenu.

📌 System State záloha je součást havarijní připravenosti, ne podporované „undo“ jedné změny schématu. Přidaný objekt schématu běžně neodstraňujeme; chybný atribut lze nanejvýš deaktivovat a nahradit správně navrženým.

Schema Master zjistíme takto:

```powershell
Import-Module ActiveDirectory
$SchemaMaster = (Get-ADForest).SchemaMaster
$SchemaDn = (Get-ADRootDSE -Server $SchemaMaster).schemaNamingContext
$SchemaMaster
$SchemaDn
```

Hodnotu registru `Schema Update Allowed` běžně nevytváříme. Na podporovaných verzích AD DS není pro standardní změnu schématu pomocí `ldifde` nutná.

---

### 2. Vygenerování jedinečného OID

⚠️ **Každý `attributeID` musí být celosvětově jedinečný. OID z ukázkového článku nebo cizí organizace nikdy nekopírujeme.**

Použijeme vlastní registrovanou OID větev nebo Microsoftem publikovaný skript z dokumentace [Obtaining an Object Identifier from Microsoft](https://learn.microsoft.com/en-us/windows/win32/ad/obtaining-an-object-identifier). Výsledek si uložíme do evidence schématu spolu s vlastníkem a účelem atributu.

V dalším příkladu nahradíme:

- `<UNIQUE_OID>` skutečně jedinečným OID,
- `<SCHEMA_DN>` hodnotou proměnné `$SchemaDn`, například `CN=Schema,CN=Configuration,DC=firma,DC=cz`.

---

### 3. Vytvoření atributu pomocí LDIF

Vytvoříme soubor `confidentialAttribute.ldf`:

```ldif
dn: CN=confidentialAttribute,<SCHEMA_DN>
changetype: add
objectClass: attributeSchema
lDAPDisplayName: confidentialAttribute
adminDisplayName: Confidential Attribute
adminDescription: Důvěrný údaj s řízeným přístupem
attributeID: <UNIQUE_OID>
attributeSyntax: 2.5.5.12
oMSyntax: 64
isSingleValued: TRUE
showInAdvancedViewOnly: TRUE
searchFlags: 128

dn:
changetype: modify
add: schemaUpdateNow
schemaUpdateNow: 1
-

dn: CN=User,<SCHEMA_DN>
changetype: modify
add: mayContain
mayContain: confidentialAttribute
-

dn:
changetype: modify
add: schemaUpdateNow
schemaUpdateNow: 1
-
```

Soubor importujeme z příkazového řádku spuštěného jako správce:

```powershell
ldifde -i -f .\confidentialAttribute.ldf -s $SchemaMaster -j .
```

⚠️ Nepoužíváme přepínač `-k`, protože by mohl skrýt chybu a ponechat import jen částečně provedený. Výsledek ověříme:

```powershell
$Attribute = Get-ADObject `
    -Server $SchemaMaster `
    -Identity "CN=confidentialAttribute,$SchemaDn" `
    -Properties lDAPDisplayName, attributeID, attributeSyntax, oMSyntax, searchFlags

$UserClass = Get-ADObject `
    -Server $SchemaMaster `
    -Identity "CN=User,$SchemaDn" `
    -Properties mayContain

$Attribute | Format-List lDAPDisplayName, attributeID, attributeSyntax, oMSyntax, searchFlags
$UserClass.mayContain -contains 'confidentialAttribute'
repadmin /replsummary
```

✅ `searchFlags` musí být `128` a poslední příkaz pro `mayContain` musí vrátit `True`. Pokud jako důvěrný označujeme již existující atribut, bit `128` přičteme k jeho současné hodnotě `searchFlags`; ostatní příznaky nesmíme přepsat.

---

### 4. Delegování přístupu

Pro čtení důvěrného atributu nestačí běžné **Read Property**. ACE musí pro konkrétní GUID atributu obsahovat **ReadProperty** i **ExtendedRight** (`CONTROL_ACCESS`). Příkaz `dsacls` neumí spolehlivě vytvořit tuto objektově specifickou kombinaci, proto použijeme rozhraní .NET.

Následující skript přidá skupině `DOMENA\HR-Confidential-Readers` právo číst atribut u uživatelů pod zadanou OU:

```powershell
Import-Module ActiveDirectory

$TargetOuDn = 'OU=Zamestnanci,DC=firma,DC=cz'
$ReaderGroup = 'DOMENA\HR-Confidential-Readers'
$SchemaDn = (Get-ADRootDSE).schemaNamingContext

$AttributeBytes = (Get-ADObject `
    -SearchBase $SchemaDn `
    -LDAPFilter '(lDAPDisplayName=confidentialAttribute)' `
    -Properties schemaIDGUID).schemaIDGUID
$UserClassBytes = (Get-ADObject `
    -SearchBase $SchemaDn `
    -LDAPFilter '(&(objectClass=classSchema)(lDAPDisplayName=user))' `
    -Properties schemaIDGUID).schemaIDGUID

$AttributeGuid = [Guid]::new($AttributeBytes)
$UserClassGuid = [Guid]::new($UserClassBytes)
$Identity = [System.Security.Principal.NTAccount]::new($ReaderGroup)
$Rights = [System.DirectoryServices.ActiveDirectoryRights]::ReadProperty -bor `
          [System.DirectoryServices.ActiveDirectoryRights]::ExtendedRight

$Rule = [System.DirectoryServices.ActiveDirectoryAccessRule]::new(
    $Identity,
    $Rights,
    [System.Security.AccessControl.AccessControlType]::Allow,
    $AttributeGuid,
    [System.DirectoryServices.ActiveDirectorySecurityInheritance]::Descendents,
    $UserClassGuid
)

$Ou = [ADSI]("LDAP://" + $TargetOuDn)
$Acl = $Ou.psbase.ObjectSecurity
$Acl.AddAccessRule($Rule)
$Ou.psbase.ObjectSecurity = $Acl
$Ou.psbase.CommitChanges()
```

Pro skupinu, která má atribut měnit, vytvoříme samostatnou ACE s právem `WriteProperty` a stejným `$AttributeGuid`.

❌ Skupinám nedáváme **Full Control** ani obecné **All Extended Rights**, protože by zpřístupnily i jiné citlivé operace.

📌 Delegace se dědí jen na objekty s povolenou dědičností. Na chráněné administrátorské účty spravované pomocí AdminSDHolder se oprávnění z OU zpravidla nepřenese.

---

### 5. Zápis a test oprávnění

Vlastní atribut lze zapisovat standardním modulem ActiveDirectory:

```powershell
Set-ADUser -Identity testuser -Replace @{
    confidentialAttribute = 'Testovací hodnota'
}

# Odstranění hodnoty
Set-ADUser -Identity testuser -Clear confidentialAttribute
```

Pro test hodnotu znovu nastavíme a čtení ověříme pod třemi účty: člen čtecí skupiny, běžný uživatel mimo skupinu a účet s právem zápisu.

```powershell
$Credential = Get-Credential
Get-ADUser `
    -Identity testuser `
    -Properties confidentialAttribute `
    -Credential $Credential `
    -Server dc1.firma.cz |
    Select-Object SamAccountName, confidentialAttribute
```

Člen `HR-Confidential-Readers` hodnotu uvidí. Běžnému uživateli se atribut nevrátí. Samostatně ověříme, že zapisovací skupina může hodnotu změnit a nemá širší práva, než bylo zamýšleno.

Po dokončení odebereme administrační účet ze **Schema Admins**, odhlásíme jej a uložíme LDIF, OID, výstup importu a schválení změny do dokumentace.

---

### Shrnutí

✅ Atribut nejprve navrhneme a otestujeme v odděleném forestu.  
✅ Pro `attributeID` použijeme vlastní jedinečný OID.  
✅ Důvěrnost nastavuje bit `128` v `searchFlags`; nejde o šifrování hodnoty.  
✅ Čtení delegujeme kombinací `ReadProperty` a objektově specifického `ExtendedRight`.  
❌ Změnu schématu nepovažujeme za snadno vratnou a importní chyby neskrýváme přepínačem `-k`.

### Zdroje

- [Mark an attribute as confidential](https://learn.microsoft.com/en-us/troubleshoot/windows-server/windows-security/mark-attribute-as-confidential)
- [Obtaining an Object Identifier from Microsoft](https://learn.microsoft.com/en-us/windows/win32/ad/obtaining-an-object-identifier)
- [Set-ADUser](https://learn.microsoft.com/en-us/powershell/module/activedirectory/set-aduser)
