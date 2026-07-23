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
$Now = Get-Date
$Candidates = @(Get-ChildItem Cert:\LocalMachine\My | Where-Object {
    $_.HasPrivateKey -and
    $_.NotBefore -le $Now -and
    $_.NotAfter -gt $Now -and
    $_.DnsNameList.Unicode -contains $KpsFqdn -and
    $_.EnhancedKeyUsageList.ObjectId.Value -contains '1.3.6.1.5.5.7.3.1'
})
$ApplicationId = '{' + [Guid]::NewGuid().ToString() + '}'

if ($Candidates.Count -ne 1) {
    $Candidates | Format-Table Subject, Thumbprint, NotAfter
    throw "Pro $KpsFqdn musí být nalezen právě jeden platný Server Authentication certifikát s privátním klíčem."
}
$Cert = $Candidates[0]

$ExistingBindings = (& netsh.exe http show sslcert | Out-String)
if ($LASTEXITCODE -ne 0) {
    throw "Nelze vypsat HTTP.sys SSL bindingy."
}
if ($ExistingBindings.Contains("0.0.0.0:$KpsPort")) {
    throw "Na 0.0.0.0:$KpsPort již existuje HTTP.sys SSL binding. Nejdříve ověřte jeho vlastníka a účel."
}

# Rezervace URL pro HTTP.sys
& netsh.exe http add urlacl "url=https://+:$KpsPort/KdcProxy" 'user=NT AUTHORITY\Network Service'
if ($LASTEXITCODE -ne 0) {
    throw "Vytvoření URL ACL selhalo. Ověřte existující rezervace příkazem netsh http show urlacl."
}

# Připojení certifikátu na 443/TCP
& netsh.exe http add sslcert `
    "ipport=0.0.0.0:$KpsPort" `
    "certhash=$($Cert.Thumbprint)" `
    "appid=$ApplicationId" `
    'certstorename=MY'
if ($LASTEXITCODE -ne 0) {
    throw "Vytvoření HTTP.sys SSL bindingu selhalo. URL ACL již může existovat; před opakováním stav zkontrolujte."
}
```

⚠️ Příkaz záměrně nepřepisuje existující SSL binding. Pokud port používá IIS, RD Gateway, DirectAccess nebo jiná služba, nejdříve vyhodnotíme sdílení HTTP.sys a existující konfiguraci; binding ani URL ACL nemažeme naslepo. Pokud skript skončí až po vytvoření URL ACL, před dalším spuštěním existující rezervaci zkontrolujeme.

### 2.3 Nastavení registru pro KDC Proxy

```powershell
$SettingsPath = 'HKLM:\SYSTEM\CurrentControlSet\Services\KPSSVC\Settings'
New-Item -Path $SettingsPath -Force | Out-Null

New-ItemProperty -Path $SettingsPath -Name HttpsClientAuth -PropertyType DWord -Value 0 -Force | Out-Null
New-ItemProperty -Path $SettingsPath -Name DisallowUnprotectedPasswordAuth -PropertyType DWord -Value 0 -Force | Out-Null

# HttpsUrlGroup je potřeba jen pro nestandardní port nebo cestu.
if ($KpsPort -ne 443) {
    $ExistingUrlGroups = @(
        (Get-ItemProperty -Path $SettingsPath -Name HttpsUrlGroup -ErrorAction SilentlyContinue).HttpsUrlGroup |
            Where-Object { $_ }
    )
    $RequiredUrlGroup = "+:$KpsPort"
    if ($RequiredUrlGroup -notin $ExistingUrlGroups) {
        $UrlGroups = @($ExistingUrlGroups + $RequiredUrlGroup | Select-Object -Unique)
        New-ItemProperty -Path $SettingsPath -Name HttpsUrlGroup -PropertyType MultiString -Value $UrlGroups -Force | Out-Null
    }
}
```

📌 Tyto hodnoty odpovídají běžnému **password-based Kerberos over HTTPS** scénáři pro standardní klienty.

Pokud nepoužíváme výchozí port `443`, musí stejný port odpovídat v URL ACL, SSL bindingu, `HttpsUrlGroup`, firewallu a klientském mapování v GPO nebo Intune.

⚠️ Otisk certifikátu je součástí HTTP.sys bindingu. Po obnově certifikátu proto nestačí, že je nový certifikát v úložišti: v servisním okně musíme binding převázat na jeho nový thumbprint a znovu ověřit HTTPS. Existující binding nemažeme bez kontroly, zda jej nesdílí další služba.

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

📌 Tuto variantu berme hlavně jako nouzové lokální nastavení pro testování a troubleshooting. Po změně registru klient restartujeme a před dalším testem provedeme `klist purge`.

Pro čistě diagnostické účely můžeme dočasně vypnout kontrolu revokace TLS certifikátu KDC proxy:

```powershell
reg add "HKLM\Software\Microsoft\Windows\CurrentVersion\Policies\System\Kerberos\Parameters" /v NoRevocationCheck /t REG_DWORD /d 1 /f
```

⚠️ Toto nastavení používejme jen pro troubleshooting. V produkčním provozu má být revocation check zapnutý.

Po testu diagnostickou výjimku bezpodmínečně odstraníme a klient restartujeme:

```powershell
Remove-ItemProperty -Path 'HKLM:\Software\Microsoft\Windows\CurrentVersion\Policies\System\Kerberos\Parameters' -Name NoRevocationCheck -ErrorAction SilentlyContinue
```

### 3.3 Pomocí Intune (Microsoft Endpoint Manager)

Pokud spravujeme klienty pomocí **Intune**, nejdříve ověříme, zda je politika dostupná v **Settings catalog**. Microsoft do katalogu průběžně přidává Windows Administrative Templates (ADMX), ale dostupnost konkrétní položky se může mezi tenanty a verzemi katalogu lišit.

📌 Doporučený postup:
- V Intune otevřeme `Devices > Manage devices > Configuration > Create > New policy`.
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

Pokud položku v **Settings catalog** nevidíme, použijeme profil **Templates > Custom** s dokumentovanou ADMX-backed policy:

- **OMA-URI:** `./Device/Vendor/MSFT/Policy/Config/ADMX_Kerberos/KdcProxyServer`
- **Data type:** `String`

Hodnota musí být SyncML payload pro zapnutí politiky a seznam mapování musí být XML-encoded. Nevkládáme do OMA-URI pouze řetězec `<https ... />`; takový profil není platnou ADMX-backed konfigurací. Payload vytvoříme podle aktuální dokumentace Policy CSP a před produkčním nasazením ověříme na jednom zařízení. Pokud nechceme SyncML spravovat ručně, použijeme GPO nebo lokální registr z kapitoly 3.2.

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
netsh http show sslcert ipport=0.0.0.0:443
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
- serverovém logu **Applications and Services Logs > Microsoft > Windows > Kerberos-KDCProxy > Operational**,
- klientském logu **Applications and Services Logs > Microsoft > Windows > Security-Kerberos > Operational**.

Přesné názvy dostupných kanálů a jejich stav ověříme z příkazového řádku spuštěného jako správce:

```powershell
wevtutil.exe el | findstr /i "Kerberos KDCProxy"
wevtutil.exe gl Microsoft-Windows-Kerberos-KDCProxy/Operational
wevtutil.exe gl Microsoft-Windows-Security-Kerberos/Operational
```

Pokud je potřebný kanál vypnutý, zapneme jej například příkazem `wevtutil.exe sl <název-kanálu> /e:true` s názvem přesně opsaným z prvního výpisu. Na klientovi jsou pro hledání příčiny užitečné mimo jiné události `108`, `109` a `200`.

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

### 4.6 Chyba `0x51f` / `0xC000005E`

Výstup:

```text
Error calling API LsaCallAuthenticationPackage (GetTicket substatus): 0x51f
klist failed with 0xc000005e
```

❌ Chyba znamená **STATUS_NO_LOGON_SERVERS**. Nejde automaticky o špatné heslo; klient nedokázal získat použitelný KDC přímo ani přes nakonfigurovanou proxy. Zkontrolujeme:

```powershell
reg query "HKLM\Software\Microsoft\Windows\CurrentVersion\Policies\System\Kerberos" /s
Get-Service iphlpsvc
Test-NetConnection kps.example.com -Port 443
$env:LOGONSERVER
nltest /dsgetdc:example.com
nltest /dclist:example.com
```

`nltest` ověřuje přímé vyhledání doménového řadiče; KDC Proxy neposkytuje obecnou LDAP ani DC Locator konektivitu. Mimo firemní síť proto může `nltest` selhat i při funkční KDC proxy. Rozhodující je správné mapování suffixu, běžící služba **IP Helper** (`iphlpsvc`), důvěryhodný certifikát včetně dostupné revokace, port `443` a události v obou Kerberos logách.

---

## 5. Shrnutí

✅ **KDC Proxy umožňuje bezpečné Kerberos ověřování přes HTTPS.**  
✅ **Nepožaduje instalaci IIS, KPSSVC běží nativně v systému.**  
✅ **Vyžaduje platný Server Authentication certifikát.**  
✅ **Klienty je nutné mapovat podle interního DNS suffixu domény na externí KDC proxy URL.**  
✅ **V Intune používáme ADMX-backed politiku `KdcProxyServer`, nikoliv vlastní nezdokumentovaný XML payload.**  
✅ **Správné ověření musí proběhnout mimo firemní síť a s kontrolou logů, ne jen pomocí `klist`.**

## Zdroje

- [SMB over QUIC - konfigurace KDC Proxy pomocí PowerShellu](https://learn.microsoft.com/en-us/windows-server/storage/file-server/smb-over-quic)
- [ADMX_Kerberos Policy CSP](https://learn.microsoft.com/en-us/windows/client-management/mdm/policy-csp-admx-kerberos)
- [Netsh HTTP](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/netsh-http)
