---
title: "Nastavení Winlogbeat pro odesílání Windows Event Logů do Graylogu"
date: 2026-04-15
tags: [Graylog, Winlogbeat, Windows, Logging]
layout: post
---

## Nastavení Winlogbeat pro odesílání Windows Event Logů do Graylogu

V tomto článku si ukážeme, jak nastavit **Winlogbeat** na Windows serveru nebo stanici tak, aby odesílal události do **Graylogu**. Na straně Graylogu použijeme **Beats input**, na straně Winlogbeatu **output.logstash**, protože Graylog Beats input používá stejný protokol jako Logstash Beats receiver.

Tento postup je vhodný například pro sběr:

* běžných Windows Event Logů (`Application`, `System`, `Security`)
* Sysmon logů
* PowerShell událostí
* LDAP událostí z logu `Directory Service` na doménových řadičích

---

### 1. Co připravit předem

Než začneme, ověříme si:

* Graylog server běží a je dostupný po síti
* z Windows stroje je dostupný Graylog na portu `TCP/5044`
* na Graylog serveru je povolený firewall pro příjem Beats dat

Pokud navazujeme na předchozí článek o instalaci Graylogu, bude vstup pro Winlogbeat běžet na serveru `graylog01`, tedy na uzlu s Graylog serverem, nikoliv na Data Node.

📌 V tomto scénáři **nepoužíváme Elasticsearch output** ani nespouštíme `winlogbeat setup`, protože data neposíláme do Elastic Stacku, ale do Graylogu.  
📌 Pokud chceme sbírat LDAP události `2886` až `2890`, musí Winlogbeat běžet na **doménovém řadiči** a v systému musí vznikat záznamy v logu **Directory Service**.

---

### 2. Nastavení Beats inputu v Graylogu

V Graylog webovém rozhraní otevřeme:

```text
System / Inputs
```

Ze seznamu inputů vybereme **Beats** a klikneme na **Launch new input**.

Doporučené základní nastavení:

* **Title**: `Winlogbeat`
* **Bind address**: `0.0.0.0`
* **Port**: `5044`
* **Node**: uzel s Graylog serverem, například `graylog01`

Pokud máme jen jeden Graylog server, je to nejjednodušší varianta. V prostředí s více Graylog servery můžeme input spouštět jako **Global**, pokud to odpovídá architektuře.

#### 2.1 Doporučení k volbám inputu

Pro běžný provoz většinou stačí:

* ponechat výchozí **Encoding**
* volitelně zapnout **TCP keepalive**
* ponechat výchozí pojmenování polí z Beats

📌 Pokud v inputu ponecháme výchozí chování prefixů, budeme v Graylogu typicky vidět pole jako `winlogbeat_source`, `winlogbeat_winlog_channel`, `winlogbeat_event_code` a podobně.

Po spuštění inputu zkontrolujeme, že opravdu běží na správném portu a správném uzlu.

#### 2.2 Firewall na Graylog serveru

Na Graylog serveru povolíme příjem na portu `5044/TCP`. Bez toho se Winlogbeat nepřipojí, i když bude konfigurace správná.

---

### 3. Instalace Winlogbeat na Windows

Pokud používáme **MSI instalátor**, nainstalujeme Winlogbeat standardním instalačním balíčkem pro Windows.

Výchozí cílová cesta MSI instalace bývá:

```text
C:\Program Files\Elastic\Beats\<verze>\winlogbeat
```

V našem příkladu budeme používat konkrétní cestu:

```text
C:\Program Files\Elastic\Beats\9.3.0\winlogbeat
```

Po dokončení instalace ověříme, že v systému existuje služba `winlogbeat`:

```powershell
Get-Service winlogbeat
```

📌 Pokud při další aktualizaci přejdeme na novější verzi, změní se i cesta obsahující číslo verze.

---

### 4. Konfigurace `winlogbeat.yml`

Hlavní konfigurace je v souboru:

```text
C:\Program Files\Elastic\Beats\9.3.0\winlogbeat\winlogbeat.yml
```

Do konfigurace nastavíme logy, které chceme číst, a jako výstup použijeme Graylog server na portu `5044`.

Ukázková konfigurace může vypadat takto:

```yaml
winlogbeat.event_logs:
  - name: Application
    ignore_older: 72h

  - name: System

  - name: Security

  - name: Microsoft-Windows-Sysmon/Operational

  - name: Windows PowerShell
    event_id: 400, 403, 600, 800

  - name: Microsoft-Windows-PowerShell/Operational
    event_id: 4103, 4104, 4105, 4106

  - name: Directory Service
    event_id: 2886, 2887, 2888, 2889, 2890

output.logstash:
  hosts: ["graylog01:5044"]
```

V našem příkladu používáme stejný název serveru jako v článku o instalaci Graylogu, tedy `graylog01`.

#### 4.1 Co tato konfigurace dělá

* `Application`, `System` a `Security` pokrývají základní Windows logy
* `Microsoft-Windows-Sysmon/Operational` je vhodný pro detailnější bezpečnostní monitoring
* `Windows PowerShell` a `Microsoft-Windows-PowerShell/Operational` doplňují dohled nad PowerShell aktivitou
* `Directory Service` s eventy `2886`, `2887`, `2888`, `2889`, `2890` pokrývá detekci nešifrovaných LDAP dotazů a slabé LDAP konfigurace
* `ignore_older: 72h` u `Application` zabrání jednorázovému načtení příliš starých záznamů

📌 Pokud Sysmon na stanici není nainstalovaný, daný kanál nebude dostupný.  
📌 Události `4103` a `4104` dávají smysl hlavně tehdy, když máme v prostředí zapnuté PowerShell logging politiky.  
📌 Události `2888` a `2889` vyžadují zapnuté rozšířené LDAP diagnostické logování, jak je popsáno v [článku o detekci nešifrovaného LDAPu]({% post_url 2025-05-15-Detekce-nesifrovanych-LDAP-dotazu-v-prostredi-Active-Directory %}).

#### 4.2 Na co si dát pozor

V `winlogbeat.yml` nesmí zůstat aktivní jiný výstup, například:

* `output.elasticsearch`
* `output.console`
* `output.file`

Pro tento scénář chceme mít aktivní pouze:

```yaml
output.logstash
```

---

### 5. Ověření konfigurace a spuštění služby

Nejprve otestujeme syntaxi konfigurace:

```powershell
cd 'C:\Program Files\Elastic\Beats\9.3.0\winlogbeat'
.\winlogbeat.exe test config -c .\winlogbeat.yml -e
```

Pokud je vše v pořádku, službu spustíme:

```powershell
Start-Service winlogbeat
Get-Service winlogbeat
```

Při změně konfigurace službu restartujeme:

```powershell
Restart-Service winlogbeat
```

---

### 6. Ověření příjmu dat v Graylogu

Po spuštění Winlogbeatu zkontrolujeme v Graylogu:

* že na Beats inputu roste počet přijatých zpráv
* že input běží bez chyb
* že se v přehledu zpráv objevují nové Windows události

V detailu inputu nebo v přehledu **Inputs** sledujeme, zda přibývají přijaté zprávy a zda input nehlásí chyby.

Pokud jsme ponechali výchozí prefixování polí, můžeme při hledání využít například pole:

```text
winlogbeat_source
```

nebo:

```text
winlogbeat_winlog_channel
```

Tak snadno ověříme, ze kterého počítače a z jakého event logu zprávy přicházejí.

---

### 7. Volitelné zabezpečení pomocí TLS

Pokud nechceme posílat Beats data nešifrovaně, můžeme na Graylog Beats inputu zapnout TLS. V takovém případě doplníme do Winlogbeatu také nastavení důvěryhodné certifikační autority:

```yaml
output.logstash:
  hosts: ["graylog01:5044"]
  ssl.certificate_authorities: ["C:/Program Files/Elastic/Beats/9.3.0/winlogbeat/certs/ca.pem"]
```

Pokud budeme vyžadovat i klientské certifikáty, je potřeba je nastavit jak v Graylogu, tak ve Winlogbeatu.

---

### 8. Nejčastější problémy

Pokud se data do Graylogu neposílají, nejčastější příčinou bývá:

* Beats input v Graylogu neběží nebo poslouchá na jiném portu
* firewall blokuje `TCP/5044`
* v `hosts` je špatné jméno nebo IP adresa serveru
* ve Winlogbeatu je stále aktivní jiný output než `output.logstash`
* Sysmon nebo PowerShell logy na stanici vůbec nevznikají
* log `Directory Service` není na serveru dostupný nebo na DC nejsou zapnuté LDAP diagnostické události

Na Windows straně je vhodné zkontrolovat i logy Winlogbeatu. Jejich umístění určuje `path.logs`; podle aktuální dokumentace Elastic bývá při běžné instalaci služby často:

```text
C:\Program Files\Winlogbeat-Data\logs
```

---

## Shrnutí

✅ Graylog musí mít spuštěný **Beats input** na `TCP/5044`  
✅ Winlogbeat musí používat **output.logstash**, nikoliv Elasticsearch output  
✅ Základní konfigurace může sbírat `Application`, `System`, `Security`, Sysmon, PowerShell i `Directory Service` logy  
✅ Po spuštění ověřujeme příjem zpráv přes metriky inputu a vyhledávání v Graylogu  
✅ Pokud chceme bezpečnější provoz, můžeme mezi Winlogbeatem a Graylogem zapnout **TLS**
