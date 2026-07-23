---
title: "Nastavení Winlogbeat pro odesílání Windows Event Logů do Graylogu"
date: 2026-04-15
tags: [Graylog, Winlogbeat, Windows, Logging]
layout: post
---

## Nastavení Winlogbeat pro odesílání Windows Event Logů do Graylogu

Winlogbeat bude číst Windows Event Logy a přes `output.logstash` je odesílat do **Beats inputu** v Graylogu. Příklad navazuje na dvouserverovou instalaci, ve které Graylog běží na `graylog01` a Data Node na `graylog02`.

📌 Pro Graylog používáme `output.logstash`. `output.elasticsearch` ponecháme vypnutý a příkaz `winlogbeat setup` nespouštíme.

---

### 1. Vytvoření Beats inputu

V Graylogu otevřeme:

```text
System / Inputs
```

1. V poli **Select input** vybereme `Beats`.
2. Klikneme na **Launch new input**.
3. Nastavíme:
   * **Title**: `Winlogbeat`
   * **Bind address**: IP adresu Graylogu, případně `0.0.0.0`
   * **Port**: `5044`
   * **Global**: zapneme jen tehdy, když má input poslouchat na všech Graylog serverech
4. Klikneme na **Launch input**.
5. Dokončíme **Input Setup Wizard**. V Graylogu 7.1 input do dokončení průvodce neběží.

📌 V edici Graylog Open obvykle zvolíme **Skip Illuminate** a data nasměrujeme do streamu používajícího `Default index set`. Delší retenci serverových logů nastavíme samostatným streamem podle článku [Základní nastavení Graylogu po instalaci]({% post_url 2026-04-15-Zakladni-nastaveni-Graylogu-po-instalaci %}).

Na firewallu `graylog01` povolíme `TCP/5044` jen z adres spravovaných Windows zařízení. Z klienta ověříme dostupnost:

```powershell
Test-NetConnection graylog01 -Port 5044
```

---

### 2. Instalace Winlogbeatu z MSI

📌 Příklad používá MSI instalaci Winlogbeatu `9.3.0` v cestě:

```text
C:\Program Files\Elastic\Beats\9.3.0\winlogbeat
```

Po instalaci ověříme službu i příkaz, kterým byla zaregistrována:

```powershell
Get-Service winlogbeat
Get-CimInstance Win32_Service -Filter "Name='winlogbeat'" |
    Select-Object Name, State, StartMode, PathName
```

💡 `PathName` je důležité při řešení problémů: ukazuje skutečný konfigurační soubor i případné parametry `path.home`, `path.data` a `path.logs`. Při upgradu se může cesta s číslem verze změnit.

---

### 3. Základní konfigurace stanice nebo serveru

Upravíme soubor:

```text
C:\Program Files\Elastic\Beats\9.3.0\winlogbeat\winlogbeat.yml
```

Základní konfigurace:

```yaml
winlogbeat.event_logs:
  - name: Application
    ignore_older: 72h

  - name: System
    ignore_older: 72h

  - name: Security
    ignore_older: 72h

  - name: Windows PowerShell
    event_id: 400, 403, 600, 800
    ignore_older: 72h

  - name: Microsoft-Windows-PowerShell/Operational
    event_id: 4103, 4104, 4105, 4106
    ignore_older: 72h

output.logstash:
  hosts: ["graylog01:5044"]
```

📌 `ignore_older: 72h` omezuje první načtení starých událostí. Pokud potřebujeme při prvním nasazení poslat i starší záznamy, hodnotu zvětšíme nebo dočasně vynecháme.

⚠️ V konfiguraci ponecháme aktivní právě jeden output. Zakomentujeme zejména případné bloky `output.elasticsearch`, `output.console` a `output.file`. Pro Graylog také nespouštíme `winlogbeat setup`, protože ten připravuje objekty pro Elastic Stack.

---

### 4. Volitelné logy podle role počítače

Do `winlogbeat.event_logs` přidáme jen kanály, které na konkrétním počítači existují. Jejich přesné názvy ověříme příkazem:

```powershell
Get-WinEvent -ListLog * | Where-Object IsEnabled | Select-Object -ExpandProperty LogName
```

#### 4.1 Sysmon

Pokud je Sysmon nainstalovaný:

```yaml
  - name: Microsoft-Windows-Sysmon/Operational
    ignore_older: 72h
```

#### 4.2 Doménový řadič a LDAP události

Pouze na doménové řadiče přidáme:

```yaml
  - name: Directory Service
    event_id: 2886, 2887, 2888, 2889
    ignore_older: 72h
```

📌 Událost `2889`, která identifikuje klienta a typ nezabezpečené LDAP vazby, vyžaduje diagnostické logování kategorie **16 LDAP Interface Events** na úrovni `2`. Postup je v článku [Detekce nezabezpečených LDAP vazeb v prostředí Active Directory]({% post_url 2025-05-15-Detekce-nesifrovanych-LDAP-dotazu-v-prostredi-Active-Directory %}). Události `2886` až `2888` mají jiný účel a zapnutí této diagnostiky pro ně není podmínkou.

#### 4.3 Microsoft DHCP Server

Pouze na DHCP server přidáme:

```yaml
  - name: Microsoft-Windows-DHCP Server Events/Admin
    ignore_older: 72h

  - name: Microsoft-Windows-DHCP Server Events/Operational
    ignore_older: 72h
```

#### 4.4 Hyper-V

Pouze na Hyper-V hosty přidáme alespoň administrační kanály správy virtuálních počítačů a worker procesů:

```yaml
  - name: Microsoft-Windows-Hyper-V-VMMS/Admin
    ignore_older: 72h

  - name: Microsoft-Windows-Hyper-V-Worker/Admin
    ignore_older: 72h
```

Další Hyper-V kanály přidáváme až podle konkrétní potřeby a po ověření jejich názvu pomocí `Get-WinEvent -ListLog *Hyper-V*`.

---

### 5. Kontrola konfigurace a spojení

PowerShell spustíme jako správce:

```powershell
Set-Location 'C:\Program Files\Elastic\Beats\9.3.0\winlogbeat'
.\winlogbeat.exe test config -c .\winlogbeat.yml -e
.\winlogbeat.exe test output -c .\winlogbeat.yml -e
```

První příkaz kontroluje syntaxi, druhý připojení k Beats inputu. Potom službu restartujeme:

```powershell
Restart-Service winlogbeat
Get-Service winlogbeat
```

Pokud služba dosud neběžela, použijeme místo restartu `Start-Service winlogbeat`.

✅ Než službu spustíme, musí projít jak `test config`, tak `test output`.

V Graylogu otevřeme `System / Inputs`, u inputu vybereme **Input Diagnosis** a ověříme příchozí zprávy. V `Search` lze při výchozím prefixování Beats polí použít například:

```text
_exists_:winlogbeat_winlog_computer_name
```

Na reálné zprávě ověříme zejména pole:

* `winlogbeat_winlog_computer_name`
* `winlogbeat_winlog_channel`
* `winlogbeat_event_code`

📌 Prefix závisí na nastavení inputu. Pokud se pole jmenují jinak, další dotazy přizpůsobíme skutečné zprávě, ne pouze příkladům v článku.

---

### 6. Zapnutí TLS

⚠️ Bez TLS jsou události na síti nešifrované. Pro produkční provoz na Beats inputu nastavíme serverový certifikát a privátní klíč a ve Winlogbeatu důvěryhodnou CA:

```yaml
output.logstash:
  hosts: ["graylog01.example.cz:5044"]
  ssl.certificate_authorities:
    - "C:/Program Files/Elastic/Beats/9.3.0/winlogbeat/certs/ca.pem"
  ssl.verification_mode: full
```

📌 DNS jméno v `hosts` musí být v `Subject Alternative Name` serverového certifikátu a certifikát nesmí být expirovaný. `ssl.verification_mode: none` nepoužíváme jako trvalé řešení. Pokud na inputu zapneme povinné klientské certifikáty, musíme ve Winlogbeatu doplnit také `ssl.certificate` a `ssl.key`.

Po zapnutí TLS znovu provedeme `test output`.

---

### 7. Řešení problémů

Pokud zprávy nepřicházejí, zkontrolujeme v tomto pořadí:

1. `Test-NetConnection graylog01 -Port 5044`.
2. Stav a **Input Diagnosis** v `System / Inputs`.
3. Výstup `winlogbeat.exe test config` a `test output`.
4. Skutečný příkaz služby v `Win32_Service.PathName`.
5. Logy v adresáři určeném parametrem `path.logs`.
6. Existenci nakonfigurovaných Event Log kanálů.
7. Oprávnění služby číst chráněné kanály, zejména `Security`.

Nejčastější příčiny jsou firewall, chybné DNS jméno, nedůvěryhodný TLS certifikát, souběžně zapnutý jiný output nebo nakonfigurovaný kanál, který na daném počítači neexistuje.

---

### Shrnutí

✅ Graylog musí mít dokončený a spuštěný **Beats input** na `TCP/5044`.  
✅ Winlogbeat používá `output.logstash` a v konfiguraci zůstává aktivní jediný output.  
✅ Kanály přidáváme podle role počítače a ověřujeme jejich skutečný název.  
✅ Před restartem služby spustíme `test config` i `test output`.  
✅ V produkčním provozu zabezpečíme spojení mezi Winlogbeatem a Graylogem pomocí TLS.

---

### Zdroje

* [Graylog: Set Up an Input](https://go2docs.graylog.org/current/getting_in_log_data/setup_an_input.htm)
* [Elastic: Install Winlogbeat](https://www.elastic.co/docs/reference/beats/winlogbeat/winlogbeat-installation-configuration)
* [Elastic: Secure communication with Logstash](https://www.elastic.co/docs/reference/beats/winlogbeat/configuring-ssl-logstash)
* [Elastic: Winlogbeat command reference](https://www.elastic.co/docs/reference/beats/winlogbeat/command-line-options)
