---
title: "Vytvoření vlastní šablony certifikátu pro doménové řadiče"
date: "2025-05-15"
tags: [PKI, Active Directory, Security]
layout: post
---

## Vytvoření vlastní šablony certifikátu pro doménové řadiče

V tomto článku si ukážeme, jak vytvořit vlastní šablonu certifikátu pro doménové řadiče v prostředí Active Directory. Cílem je zajistit, aby vydané certifikáty obsahovaly požadované EKU (včetně KDC Authentication) a odpovídající informace v Subject a SAN, což je často požadováno v rámci bezpečnostních auditů a penetračních testů.

---

### 1. Výchozí bod: Kerberos Authentication šablona

#### 1.1 Proč ji použít

* Obsahuje správné EKU pro doménové řadiče
* Je vhodná jako základ pro šablonu, která má sloužit pro ověřování pomocí Kerberosu

#### 1.2 Co jí chybí

* Ve výchozím nastavení nevynucuje přítomnost Subject (CN)
* SAN může chybět, pokud není správně nakonfigurována

---

### 2. Vytvoření nové šablony

#### 2.1 Spuštění správce šablon

* Spustíme `certtmpl.msc`
* Najdeme šablonu **Kerberos Authentication**
* Klikneme pravým tlačítkem → **Duplicate Template**

#### 2.2 Výběr verze šablony

* Doporučujeme: **Windows Server 2008** nebo novější
* Umožní použití moderních možností a autoenrollmentu

---

### 3. Konfigurace šablony

#### 3.1 Základní nastavení (General)

* Name: `Secured DC Authentication`
* Platnost: 2 roky
* Obnovení: 6 měsíců před expirací

#### 3.2 Zpracování požadavku (Request Handling)

* Purpose: `Signature and encryption`
* ⚠ Nezaškrtáváme export privátního klíče (zbytečné pro DC)

#### 3.3 Subjekt a SAN (Subject Name)

* Vybereme: **Build from this Active Directory information**
* Zaškrtneme:

    * `Common Name`
    * `DNS Name`
    * `Service Principal Name (SPN)` 📌

#### 3.4 Rozšíření (Extensions)

* Ověříme nebo přidáme EKU:

    * `Server Authentication`
    * `Client Authentication`
    * `Smart Card Logon`
    * `KDC Authentication` ✅

#### 3.5 Oprávnění (Security)

* Přidáme skupinu `Domain Controllers`
* Povolíme:

    * `Read`
    * `Enroll`
    * `Autoenroll` (volitelné)

---

### 4. Publikace a použití

#### 4.1 Publikace v certifikační autoritě

* Spustíme `certsrv.msc`
* Pravým tlačítkem na **Certificate Templates** → **New → Certificate Template to Issue**
* Vybereme naši šablonu `Secured DC Authentication`

#### 4.2 Získání certifikátu doménovým řadičem

* Automaticky přes GPO (pokud je nastaven autoenrollment)
* Nebo ručně:

```powershell
certreq -new -config "CA\NAZEV" -attrib "CertificateTemplate:SecuredDCAuthentication" request.inf
```

---

## Shrnutí

✅ Vyšli jsme ze šablony Kerberos Authentication
✅ Doplnili jsme Subject a SAN pomocí AD atributů
✅ Zkontrolovali jsme a doplnili potřebné EKU, včetně KDC Authentication
✅ Upravili jsme oprávnění pro vydávání doménovým řadičům
✅ Publikovali jsme šablonu a připravili ji k použití
