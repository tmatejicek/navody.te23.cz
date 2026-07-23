---
title: "Vytvoření vlastní šablony certifikátu pro doménové řadiče"
date: 2025-05-15
tags: [PKI, Active Directory, Security]
layout: post
---

## Vytvoření vlastní šablony certifikátu pro doménové řadiče

Certifikát doménového řadiče se používá například pro LDAPS a certifikátové Kerberos ověřování. Pro moderní šablonu vyjdeme z **Kerberos Authentication**, protože starší šablony **Domain Controller** a **Domain Controller Authentication** neobsahují EKU **KDC Authentication**.

📌 **Šablonu Kerberos Authentication duplikujeme. Neupravujeme přímo vestavěnou šablonu.**

---

### 1. Předpoklady

Před změnou ověříme:

- Enterprise CA je dostupná a její certifikační řetězec je důvěryhodný v doméně.
- CRL a případně OCSP/AIA adresy jsou dostupné všem klientům, kteří budou certifikát ověřovat.
- certifikát vydávající CA je publikovaný v enterprise úložišti `NTAuth`, což ověříme příkazem `certutil -viewstore -enterprise NTAuth`,
- doménové řadiče mají správné DNS názvy a funguje replikace AD.

Certifikát musí skončit v úložišti **Local Computer > Personal**, mít privátní klíč a obsahovat:

- Key Usage: `Digital Signature` a `Key Encipherment`,
- EKU `Server Authentication` (`1.3.6.1.5.5.7.3.1`),
- EKU `Client Authentication` (`1.3.6.1.5.5.7.3.2`),
- EKU `KDC Authentication` (`1.3.6.1.5.2.3.5`),
- DNS jméno řadiče domény v Subject Alternative Name.

📌 `Smart Card Logon` do šablony nepřidáváme, pokud pro něj nemáme samostatný, otestovaný požadavek.

---

### 2. Vytvoření šablony

1. Na CA nebo administrační stanici spustíme `certtmpl.msc`.
2. Na šabloně **Kerberos Authentication** zvolíme **Duplicate Template**.
3. Na kartě **Compatibility** nastavíme:
   - **Certification Authority:** `Windows Server 2016`,
   - **Certificate recipient:** `Windows 10 / Windows Server 2016`.
4. Pokud v prostředí zůstávají starší podporované systémy, kompatibilitu nejprve ověříme a nastavíme podle nejstaršího skutečně používaného systému.

#### 2.1 General

- **Template display name:** `Domain Controller Authentication (Kerberos)`
- **Validity period:** podle PKI politiky, běžně 1 rok
- **Renewal period:** například 6 týdnů

💡 Poznamenáme si také **Template name** bez mezer; zobrazený název a interní název nejsou totéž.

#### 2.2 Request Handling a Cryptography

Na kartě **Request Handling** ponecháme účel `Signature and encryption` a nepovolíme export privátního klíče.

Na kartě **Cryptography** nastavíme:

- **Provider Category:** `Key Storage Provider`,
- **Algorithm name:** `RSA`,
- **Minimum key size:** alespoň `2048`,
- **Request hash:** `SHA256`.

#### 2.3 Subject Name

Na kartě **Subject Name** nastavíme:

- **Build from this Active Directory information**,
- **Subject name format:** `None`,
- v **Include this information in alternate subject name** pouze `DNS name`.

Nezaškrtáváme `Common name`, `Service principal name` ani další položky. Pro identifikaci řadiče je rozhodující DNS jméno v SAN.

#### 2.4 Extensions

Na kartě **Extensions > Application Policies** ověříme přesně tyto tři EKU:

- `Client Authentication`,
- `Server Authentication`,
- `KDC Authentication`.

Protože šablonu duplikujeme z **Kerberos Authentication**, zachová se také požadované rozšíření šablony doménového řadiče. Nemažeme je ani nenahrazujeme vlastním rozšířením.

#### 2.5 Security

Skupině **Domain Controllers** povolíme:

- `Read`,
- `Enroll`,
- `Autoenroll`.

⚠️ Ostatním skupinám nepřidáváme Enroll bez konkrétního důvodu. Privátní klíč nesmí být exportovatelný.

---

### 3. Nahrazení starších šablon

Na kartě **Superseded Templates** nové šablony přidáme šablony, které mají být nahrazeny:

- `Domain Controller`,
- `Domain Controller Authentication`,
- `Kerberos Authentication`,
- případné dřívější vlastní šablony DC.

Poté v `certsrv.msc` otevřeme **Certificate Templates > New > Certificate Template to Issue** a publikujeme novou šablonu.

⚠️ **Starší šablony neodpublikujeme dříve, než pilotní řadiče získají nový certifikát a ověření proběhne úspěšně.** Po dokončení migrace je na vydávajících CA odpublikujeme, aby se DC znovu neenrollovaly podle staré šablony.

---

### 4. Autoenrollment

Vytvoříme GPO propojené pouze s OU **Domain Controllers**:

**Computer Configuration > Policies > Windows Settings > Security Settings > Public Key Policies > Certificate Services Client - Auto-Enrollment**

Nastavíme:

- **Configuration Model:** `Enabled`,
- `Renew expired certificates, update pending certificates, and remove revoked certificates`,
- `Update certificates that use certificate templates`.

Na jednom pilotním DC spustíme jako správce:

```powershell
gpupdate /force
certreq.exe -autoenroll -q
```

📌 Příkaz `certreq -new` bez připraveného INF souboru zde nepoužíváme. Pro tuto šablonu je správnou cestou autoenrollment.

---

### 5. Ověření

Na každém DC otevřeme `certlm.msc` a zkontrolujeme **Personal > Certificates**. Případně použijeme:

```powershell
certutil.exe -q -v -store my
certutil.exe -dcinfo verify
```

Ověříme:

- certifikát pochází z nové šablony,
- DNS SAN odpovídá FQDN řadiče,
- jsou přítomna přesně požadovaná EKU včetně KDC Authentication,
- certifikát má privátní klíč a je platný,
- řetězec, CRL a AIA/OCSP lze úspěšně ověřit,
- LDAPS na `636/TCP` a používané certifikátové Kerberos scénáře fungují.

Události autoenrollmentu najdeme v **Applications and Services Logs > Microsoft > Windows > CertificateServices-Lifecycles-System**. Až po ověření všech DC odpublikujeme nahrazené šablony.

---

### Shrnutí

✅ Vycházíme ze šablony **Kerberos Authentication**.  
✅ Certifikát obsahuje DNS SAN, privátní klíč a EKU Client, Server i KDC Authentication.  
✅ Skupině **Domain Controllers** povolíme `Read`, `Enroll` a `Autoenroll`.  
✅ Nasazení nejprve ověříme na pilotním řadiči včetně LDAPS a certifikačního řetězce.  
⚠️ Starší šablony odpublikujeme až po úspěšné migraci všech doménových řadičů.

### Zdroje

- [Windows Hello for Business on-premises key trust: Configure domain controller certificates](https://learn.microsoft.com/en-us/windows/security/identity-protection/hello-for-business/deploy/on-premises-key-trust#configure-domain-controller-certificates)
- [Certificates and certificate templates](https://learn.microsoft.com/en-us/windows-server/identity/ad-cs/certificate-template-concepts)
