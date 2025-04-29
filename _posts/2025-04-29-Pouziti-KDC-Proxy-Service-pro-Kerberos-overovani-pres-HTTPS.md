---
title: "Použití KDC Proxy Service pro Kerberos ověřování přes HTTPS"
date: "2025-04-29"
tags: [Kerberos, VPN, Windows Server, Active Directory]
layout: post
---

## Použití KDC Proxy Service pro Kerberos ověřování přes HTTPS

Kerberos je standardem pro autentizaci v Active Directory. Však v prostředích s VPN (zejména split-tunnel VPN) nebo v extranetových scénářích může být klasické Kerberos ověřování problémové kvůli blokovaným portům 88/464 TCP. 

Řešením je **KDC Proxy Service (KPSSVC)**, která umožňuje proxyování Kerberosu přes HTTPS (port 443). V tomto návodu si ukážeme, jak ji správně nastavit a použít.

---

## 1. Příprava certifikátu pro KDC Proxy

Než začneme, musíme připravit certifikát:

📌 **Požadavky na certifikát:**
- Použit šablonu **Web Server** nebo vlastní šablonu založenou na Web Server.
- EKU (Extended Key Usage) obsahující **Server Authentication (1.3.6.1.5.5.7.3.1)**.
- Subject Alternative Name (SAN) obsahuje FQDN serveru (např. `kps.example.com`).
- Certifikát uložen v **LocalMachine\\My** store.

Certifikát vystavíme přes interní CA nebo jinou důvěryhodnou certifikační autoritu.

---

## 2. Instalace a konfigurace KDC Proxy na serveru

### 2.1 Předpoklady

Než začneme s konfigurací, ujistíme se, že:
- Používáme **Windows Server 2012 nebo novější**.
- Máme **platný Server Authentication certifikát**.
- Port **443/TCP** je otevřen.

### 2.2 Nastavení URL ACL a SSL certifikátu

Na serveru spustíme PowerShell jako administrátor a provedeme:

```powershell
$KpsFqdn = "kps.example.com"
$KpsPort = 443

# Povolit HTTP namespace
cmd /c "netsh http add urlacl url=https://+:{0}/KdcProxy user=\"NT AUTHORITY\\Network Service\"" -f $KpsPort

# Najít certifikát podle Subject
$Cert = Get-ChildItem -Path Cert:\\LocalMachine\\My | Where-Object { $_.Subject -like "*$KpsFqdn*" }
$Guid = [Guid]::NewGuid().ToString("B")

# Připojit SSL certifikát
cmd /c "netsh http add sslcert hostnameport={0}:{1} certhash={2} appid={3} certstorename=MY" -f $KpsFqdn, $KpsPort, $Cert.Thumbprint, $Guid
```

### 2.3 Nastavení registru pro KDC Proxy

```powershell
New-ItemProperty -Path HKLM:\SYSTEM\CurrentControlSet\Services\KPSSVC\Settings -Name HttpsClientAuth -Type Dword -Value 1 -Force
New-ItemProperty -Path HKLM:\SYSTEM\CurrentControlSet\Services\KPSSVC\Settings -Name DisallowUnprotectedPasswordAuth -Type Dword -Value 1 -Force
New-ItemProperty -Path HKLM:\SYSTEM\CurrentControlSet\Services\KPSSVC\Settings -Name HttpsUrlGroup -Type MultiString -Value "+:$KpsPort" -Force
```

### 2.4 Spuštění služby a otevření portu

```powershell
Set-Service -Name KPSSVC -StartupType Automatic
Start-Service -Name KPSSVC

New-NetFirewallRule -DisplayName "Allow KDCProxy TCP $KpsPort" -Direction Inbound -Protocol TCP -LocalPort $KpsPort -Action Allow
```

---

## 3. Nastavení klientů pomocí Group Policy

### 3.1 Vytvoření a konfigurace GPO

1. Otevřeme **Group Policy Management Console (GPMC.msc)**.
2. Vytvoříme novou Group Policy Object (GPO) např. `KDC Proxy Settings`.
3. Přejdeme na:
   `Computer Configuration > Administrative Templates > System > Kerberos`
4. Otevřeme nastavení **Specify KDC proxy servers for Kerberos clients**.
5. Nastavíme hodnotu:

```
<https kps.example.com:443:KdcProxy />
```

📌 Pokud máme více proxy serverů, oddělíme je mezerou:

```
<https kps1.example.com:443:KdcProxy kps2.example.com:443:KdcProxy />
```

### 3.2 Aplikace změn

- Po aplikaci GPO doporučujeme klienty restartovat nebo spustit `gpupdate /force`.
- Klienti musí mít přístup k CRL/CDP bodům pro ověření certifikátu serveru.

---

## 4. Troubleshooting

### 4.1 Ověření dostupnosti KPSSVC

```powershell
Get-Service KPSSVC
```

Služba musí běžet (Status: Running).

### 4.2 Kontrola otevřeného portu

```powershell
netstat -ano | findstr :443
```

### 4.3 Kontrola SSL certifikátu

Ověříme, že certifikát obsahuje:
- Server Authentication v EKU
- Správný FQDN v SAN

### 4.4 Event Logy

Chyby najdeme v:
- **Applications and Services Logs > Microsoft > Windows > KdcProxy Server**

### 4.5 Test Kerberos ověření

Na klientovi spustíme:

```powershell
klist get krbtgt
```

Pokud proxy funguje, ticket bude úspěšně vydán.

---

## 5. Shrnutí

✅ **KDC Proxy umožňuje bezpečné Kerberos ověřování přes HTTPS.**  
✅ **Nepožaduje instalaci IIS, KPSSVC běží nativně v systému.**  
✅ **Vyžaduje platný Server Authentication certifikát.**  
✅ **Klienti jsou nastaveni pomocí Group Policy.**  
✅ **Troubleshooting je snadný pomocí event logů a klist testů.**

Tímto jsme úěšně nastavili KDC Proxy pro bezpečné Kerberos ověřování přes újišťěný kanál! 🌟

