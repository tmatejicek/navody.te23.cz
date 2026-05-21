---
title: "Použití KDC Proxy Service pro Kerberos ověřování přes HTTPS"
date: "2025-04-29"
tags: [Kerberos, VPN, Windows Server, Active Directory]
layout: post
---

## Použití KDC Proxy Service pro Kerberos ověřování přes HTTPS

Kerberos je standardem pro autentizaci v Active Directory. V prostředích s VPN, split-tunnel VPN nebo v extranetových scénářích ale často není možné spolehlivě otevřít přímou komunikaci klientů na doménové řadiče přes porty `88` a `464`.

Řešením je **KDC Proxy Service (KPSSVC)**, která umožňuje proxyovat Kerberos přes **HTTPS** na portu `443`. V tomto návodu si ukážeme, jak službu správně nastavit, jak nakonfigurovat klienty a jak ověřit, že opravdu používají KDC proxy a ne přímý přístup na doménový řadič.

---

## 1. Příprava certifikátu pro KDC Proxy

Než začneme, musíme připravit certifikát pro HTTPS endpoint KDC proxy.

📌 **Požadavky na certifikát:**
- Použít šablonu **Web Server** nebo vlastní šablonu založenou na Web Server.
- EKU (Extended Key Usage) obsahující **Server Authentication (1.3.6.1.5.5.7.3.1)**.
- Subject Alternative Name (SAN) obsahuje FQDN serveru (např. `kps.example.com`).
- Certifikát obsahuje **privátní klíč**.
- Certifikát uložen v **LocalMachine\\My** store.
- Certifikační řetězec je důvěryhodný pro klienty.
- Klienti mimo firemní síť se dostanou na **CRL/OCSP** endpointy certifikátu. Pokud revocation check neprojde, klient KDC proxy standardně odmítne.

Certifikát vystavíme přes interní CA nebo jinou důvěryhodnou certifikační autoritu.

📌 Pro internetové scénáře používejme **FQDN**, nikoliv IP adresu. Klienti se musí připojovat jménem, které odpovídá certifikátu.

---

## 2. Instalace a konfigurace KDC Proxy na serveru

### 2.1 Předpoklady

Než začneme s konfigurací, ujistíme se, že:
- Používáme **Windows Server 2012 nebo novější**.
- Server je připojený do domény nebo má spolehlivý interní přístup k doménovým řadičům.
- Server umí interně komunikovat s doménovými řadiči na Kerberos portech `88` a `464`.
- Máme **platný Server Authentication certifikát**.
- Port **443/TCP** je otevřen z klientů na KDC proxy server.
- Port `443` nepoužívá jiná služba, která by kolidovala s HTTP.sys bindingem.

### 2.2 Nastavení URL ACL a SSL bindingu

Na serveru spustíme PowerShell jako administrátor a provedeme:

```powershell
$KpsFqdn = "kps.example.com"
$KpsPort = 443
$Cert = Get-ChildItem Cert:\LocalMachine\My | Where-Object {
    $_.DnsNameList.Unicode -contains $KpsFqdn
} | Select-Object -First 1
$Guid = [Guid]::NewGuid()

if (-not $Cert) {
    throw "Certifikát pro $KpsFqdn nebyl nalezen v LocalMachine\My."
}

# Rezervace URL pro HTTP.sys
cmd /c ('netsh http add urlacl url=https://+:{0}/KdcProxy user="NT AUTHORITY\Network Service"' -f $KpsPort)

# Připojení certifikátu na 443/TCP
Add-NetIPHttpsCertBinding -IPPort "0.0.0.0:$KpsPort" -CertificateHash $Cert.Thumbprint -CertificateStoreName My -ApplicationId "{$Guid}" -NullEncryption $false
```

### 2.3 Nastavení registru pro KDC Proxy

```powershell
Set-ItemProperty -Path HKLM:\SYSTEM\CurrentControlSet\Services\KPSSVC\Settings -Name HttpsClientAuth -Type DWord -Value 0 -Force
Set-ItemProperty -Path HKLM:\SYSTEM\CurrentControlSet\Services\KPSSVC\Settings -Name DisallowUnprotectedPasswordAuth -Type DWord -Value 0 -Force
```

Tyto hodnoty odpovídají běžnému **password-based Kerberos over HTTPS** scénáři pro standardní klienty.

📌 Pokud z nějakého důvodu nepoužíváme výchozí port `443`, musíme tomu přizpůsobit i SSL binding a klientské mapování v GPO/Intune.

### 2.4 Spuštění služby a otevření portu

```powershell
Set-Service -Name KPSSVC -StartupType Automatic
Start-Service -Name KPSSVC

New-NetFirewallRule -DisplayName "Allow KDCProxy TCP $KpsPort" -Direction Inbound -Protocol TCP -LocalPort $KpsPort -Action Allow
```

---

## 3. Nastavení klientů

### 3.1 Pomocí Group Policy

1. Otevřeme **Group Policy Management Console (GPMC.msc)**.
2. Vytvoříme novou Group Policy Object (GPO) např. `KDC Proxy Settings`.
3. Přejdeme na:
   `Computer Configuration > Administrative Templates > System > Kerberos`
4. Otevřeme nastavení **Specify KDC proxy servers for Kerberos clients**.
5. Nastavíme politiku na **Enabled** a klikneme na **Show...**
6. V dialogu zadáme mapování:

```text
Value Name: example.com
Value: <https kps.example.com:443:kdcproxy />
```

`Value Name` odpovídá internímu DNS suffixu domény nebo realm, pro který má klient proxy použít.

📌 Pokud máme více proxy serverů pro stejnou doménu, oddělíme je ve stejné hodnotě mezerou:

```text
Value Name: example.com
Value: <https kps1.example.com:443:kdcproxy kps2.example.com:443:kdcproxy />
```

📌 Pokud máme více interních doménových suffixů, vytvoříme **samostatný řádek pro každý suffix**.

- Po aplikaci GPO doporučujeme klienty restartovat nebo spustit `gpupdate /force`.

### 3.2 Lokálně pomocí registrů (pro troubleshooting)

Pokud potřebujeme KDC proxy otestovat na jednom klientovi bez GPO a Intune, můžeme stejné mapování nastavit lokálně v registru.

Nejprve povolíme samotnou politiku:

```powershell
reg add "HKLM\Software\Microsoft\Windows\CurrentVersion\Policies\System\Kerberos" /v KdcProxyServer_Enabled /t REG_DWORD /d 1 /f
```

Potom vytvoříme mapování interního DNS suffixu domény na externí KDC proxy URL:

```powershell
reg add "HKLM\Software\Microsoft\Windows\CurrentVersion\Policies\System\Kerberos\KdcProxy\ProxyServers" /v example.com /t REG_SZ /d "<https kps.example.com:443:kdcproxy />" /f
```

Toto nastavení odpovídá stejnému mapování jako v GPO:

```text
example.com -> <https kps.example.com:443:kdcproxy />
```

📌 Tuto variantu berme hlavně jako nouzové lokální nastavení pro testování a troubleshooting.  
📌 Po změně registru doporučujeme klient restartovat a před dalším testem provést `klist purge`.

Pro čistě diagnostické účely můžeme dočasně vypnout kontrolu revokace TLS certifikátu KDC proxy:

```powershell
reg add "HKLM\Software\Microsoft\Windows\CurrentVersion\Policies\System\Kerberos\Parameters" /v NoRevocationCheck /t REG_DWORD /d 1 /f
```

⚠ Toto nastavení používejme jen pro troubleshooting. V produkčním provozu má být revocation check zapnutý.

### 3.3 Pomocí Intune (Microsoft Endpoint Manager)

Pokud spravujeme klienty pomocí **Intune**, nejpraktičtější je dnes použít **Settings catalog**, protože Microsoft do něj promítá i Windows **Administrative Templates (ADMX)** nastavení a není nutné ručně skládat OMA-URI payload.

📌 Doporučený postup:
- V Intune otevřeme `Devices > Configuration > Create > New policy`.
- Zvolíme:
  - **Platform:** `Windows 10 and later`
  - **Profile type:** `Settings catalog`
- V části **Add settings** vyhledáme:
  `Specify KDC proxy servers for Kerberos clients`
- Přidáme toto nastavení a nakonfigurujeme stejné mapování jako v GPO:

```text
Value Name: example.com
Value: <https kps.example.com:443:kdcproxy />
```

📌 Pokud spravujeme více doménových suffixů, přidáme samostatnou položku pro každý suffix.  
📌 Politiku doporučujeme přiřazovat na **device groups**, protože jde o **Computer Configuration** nastavení.

Pokud ve vaší tenant verzi nebo v konkrétním prostředí tento setting v **Settings catalog** nevidíte, použijeme fallback přes **Custom** profil s ADMX-backed policy:

- **OMA-URI:** `./Device/Vendor/MSFT/Policy/Config/ADMX_Kerberos/KdcProxyServer`
- **Data type:** `String (XML file)`

V takovém případě ale musí být hodnota zadaná jako **SyncML / XML-encoded ADMX-backed payload** podle aktuální Microsoft dokumentace. Proto je v běžné praxi bezpečnější a jednodušší použít nejdřív **Settings catalog** a teprve až jako nouzové řešení sahat po custom OMA-URI profilu.

📌 Pro troubleshooting lze klientům dočasně nasadit i politiku **Disable revocation checking for the SSL certificate of KDC proxy servers**, ale Microsoft ji doporučuje jen pro diagnostiku, ne pro produkční provoz.

---

## 4. Troubleshooting

### 4.1 Ověření dostupnosti KPSSVC

```powershell
Get-Service KPSSVC
```

Služba musí běžet (Status: Running).

### 4.2 Kontrola URL ACL, SSL bindingu a otevřeného portu

```powershell
netsh http show urlacl | findstr /i KdcProxy
Get-NetIPHttpsCertBinding -IPPort "0.0.0.0:443"
netstat -ano | findstr :443
```

### 4.3 Kontrola SSL certifikátu

Ověříme, že certifikát obsahuje:
- Server Authentication v EKU
- Správný FQDN v SAN
- Platný privátní klíč
- Pro externí klienty dostupný CRL/OCSP endpoint

### 4.4 Event Logy

Chyby najdeme v:
- serverových logách **KdcProxy**
- klientském logu **Applications and Services Logs > Microsoft > Windows > Security-Kerberos > Operational**

### 4.5 Reálný test použití KDC proxy

Na klientovi test provádíme **mimo firemní síť**, kde:

* je dostupný `kps.example.com:443`
* není dostupný přímý Kerberos na doménový řadič přes `88/464`

Pomocné testy mohou vypadat například takto:

```powershell
Test-NetConnection kps.example.com -Port 443
Test-NetConnection dc1.example.com -Port 88
Test-NetConnection dc1.example.com -Port 464
klist purge
```

Teprve potom provedeme přístup k doménovému prostředku nebo vyžádání Kerberos ticketu a ověříme v klientských a serverových logách, že požadavek skutečně prošel přes KDC proxy.

📌 Samotné `klist get krbtgt` nestačí jako důkaz použití proxy. Pokud je klient stále schopný dosáhnout přímo na doménový řadič, Kerberos může fungovat i bez KDC proxy.

---

## 5. Shrnutí

✅ **KDC Proxy umožňuje bezpečné Kerberos ověřování přes HTTPS.**  
✅ **Nepožaduje instalaci IIS, KPSSVC běží nativně v systému.**  
✅ **Vyžaduje platný Server Authentication certifikát.**  
✅ **Klienty je nutné mapovat podle interního DNS suffixu domény na externí KDC proxy URL.**  
✅ **V Intune používáme ADMX-backed politiku `KdcProxyServer`, nikoliv vlastní nezdokumentovaný XML payload.**  
✅ **Správné ověření musí proběhnout mimo firemní síť a s kontrolou logů, ne jen pomocí `klist`.**
