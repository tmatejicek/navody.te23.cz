---
title: "Praktické dashboardy a uložená hledání v Graylogu"
date: 2026-05-21
tags: [Graylog, Dashboards, Monitoring, Windows, Networking]
layout: post
---

## Praktické dashboardy a uložená hledání v Graylogu

Příklady jsou určené pro menší organizaci s Active Directory, Hyper-V, Microsoft DHCP, MikroTikem a UniFi. Předpokládají výchozí prefixování polí z Beats inputu, které vytváří například `winlogbeat_winlog_computer_name` a `winlogbeat_event_code`.

📌 **Než dotaz uložíme, otevřeme v `Search` jednu skutečnou zprávu a ověříme názvy i obsah polí.**

⚠️ Pokud Graylog hlásí **Unknown field**, pole není dostupné ve vybraných streamech nebo časovém rozsahu, případně se v dané instalaci jmenuje jinak.

---

### 1. Uložené hledání a dashboard

**Saved search** uchovává dotaz, časový rozsah, vybrané streamy, filtry a rozložení Search stránky. Používáme jej pro opakované šetření.

**Dashboard** obsahuje widgety, z nichž každý má vlastní dotaz, streamy a časový rozsah. Používáme jej pro rychlou kontrolu stavu. Hledání lze exportovat do dashboardu, ale pozdější změnu uloženého hledání nepovažujeme za automatickou změnu již vytvořeného widgetu; po úpravě vždy zkontrolujeme obě entity.

📌 Search bar nad dashboardem pouze dočasně přidává nebo přepisuje filtr zobrazených widgetů. Nemění jejich uloženou konfiguraci.

Pracovní postup:

1. V `Search` nastavíme čas, stream a dotaz.
2. Ověříme výsledky na několika zprávách.
3. Přes **Save** vytvoříme uložené hledání.
4. Přes **Create +** vytvoříme widget.
5. Widget uložíme do dashboardu a zkontrolujeme jeho vlastní časový rozsah.

---

### 2. Praktická uložená hledání

#### Neúspěšná přihlášení za 24h

```text
winlogbeat_winlog_channel:"Security" AND winlogbeat_event_code:4625
```

#### Interaktivní přihlášení uživatelů za 24h

```text
winlogbeat_winlog_channel:"Security" AND winlogbeat_event_code:4624 AND winlogbeat_winlog_event_data_LogonType:(2 OR 7 OR 10 OR 11) AND NOT winlogbeat_winlog_event_data_TargetUserName:/.*\$/ AND NOT winlogbeat_winlog_event_data_TargetUserName:(SYSTEM OR "ANONYMOUS LOGON" OR "LOCAL SERVICE" OR "NETWORK SERVICE") AND NOT winlogbeat_winlog_event_data_TargetUserName:/DWM-.*/ AND NOT winlogbeat_winlog_event_data_TargetUserName:/UMFD-.*/
```

Typy přihlášení jsou `2` interaktivní, `7` odemknutí, `10` RemoteInteractive/RDP a `11` CachedInteractive. Dotaz záměrně nepočítá síťová přihlášení typu `3`, která jsou velmi častá a obvykle nereprezentují uživatele pracujícího na počítači.

#### PowerShell 4104 za 24h

```text
winlogbeat_winlog_channel:"Microsoft-Windows-PowerShell/Operational" AND winlogbeat_event_code:4104
```

#### LDAP 2889 za 24h

```text
winlogbeat_winlog_channel:"Directory Service" AND winlogbeat_event_code:2889
```

Událost `2889` ukazuje klienty a identity používající nezabezpečené LDAP vazby. Její zapnutí popisuje článek [Detekce nezabezpečených LDAP vazeb v prostředí Active Directory]({% post_url 2025-05-15-Detekce-nesifrovanych-LDAP-dotazu-v-prostredi-Active-Directory %}).

#### Microsoft DHCP události za 24h

```text
winlogbeat_winlog_computer_name:DHCP01 AND (winlogbeat_winlog_channel:"Microsoft-Windows-DHCP Server Events/Admin" OR winlogbeat_winlog_channel:"Microsoft-Windows-DHCP Server Events/Operational")
```

`DHCP01` nahradíme skutečnou hodnotou pole `winlogbeat_winlog_computer_name`. Oba kanály musí být přidané v konfiguraci Winlogbeatu na DHCP serveru.

#### Hyper-V administrační události

Pokud Winlogbeat sbírá příslušné kanály:

```text
winlogbeat_winlog_computer_name:(HV01 OR HV02) AND (winlogbeat_winlog_channel:"Microsoft-Windows-Hyper-V-VMMS/Admin" OR winlogbeat_winlog_channel:"Microsoft-Windows-Hyper-V-Worker/Admin")
```

Chceme-li vybrat jen chyby a varování, nejprve na skutečné zprávě ověříme pole s úrovní a jeho hodnoty. Podle verze ingestu může být dostupné například `winlogbeat_winlog_level`; neověřený název pole do provozního dotazu nepřidáváme.

#### MikroTik chyby a přihlášení

```text
source:mikrotik01 AND (error OR critical OR login OR failure OR configuration)
```

#### UniFi důležité události

```text
source:(unifi-gateway01 OR unifi-switch01 OR unifi-ap01) AND (error OR disconnect* OR uplink OR wan OR dhcp)
```

💡 U syslogu upravíme hodnoty `source` a hledané výrazy podle skutečných zpráv. Text může být závislý na verzi zařízení i jazyku.

---

### 3. Vytvoření dashboardu

Otevřeme:

```text
Dashboards / Create new dashboard
```

Dashboard pojmenujeme například `Ranní kontrola infrastruktury`. Pro každý widget nastavíme vlastní rozsah `1 day ago - Now`; dashboardový filtr pak můžeme použít jen pro dočasné zúžení.

Graylog 7.1 nabízí mimo jiné:

* **Aggregation** s vizualizací `Data Table`, `Single Number`, `Line Chart` nebo `Bar Chart`
* **Message Table** pro konkrétní zprávy
* **Text (Markdown)** pro stručný návod k dashboardu

📌 `Time Series` není samostatný typ widgetu. Časový trend vytvoříme jako `Aggregation` s `Group By = timestamp` a vizualizací `Line Chart`.

---

### 4. Počítače, které poslaly Windows logy

Dotaz:

```text
_exists_:winlogbeat_winlog_computer_name
```

Nastavení widgetu:

1. V `Search` ověříme dotaz a nastavíme `1 day ago - Now`.
2. Klikneme na **Create + / Aggregation**.
3. V **Group By** ponecháme směr `Row` a vybereme `winlogbeat_winlog_computer_name`.
4. **Limit** nastavíme alespoň na očekávaný počet zařízení, například `150`.
5. V **Metrics** zvolíme **Latest Value**.
6. Jako **Field** vybereme `winlogbeat_@timestamp` a jako název `Poslední událost`.
7. V **Sort** zvolíme `Poslední událost` a směr `Descending`.
8. V **Visualization** vybereme `Data Table`.
9. Widget pojmenujeme `Počítače, které poslaly logy` a uložíme.

✅ Výsledkem je jeden řádek na počítač a čas jeho poslední zprávy v daném časovém okně. `Count` ani `Percentage` pro tento účel nepotřebujeme, protože by ukazovaly počet nebo podíl zpráv, nikoli stav počítače.

Chceme-li samostatné číslo:

1. Vytvoříme další **Aggregation** nad stejným dotazem.
2. Bez `Group By` nastavíme metriku **Cardinality**.
3. Jako pole vybereme `winlogbeat_winlog_computer_name`.
4. Vizualizaci nastavíme na `Single Number`.

⚠️ Toto číslo je počet unikátních počítačů, které v časovém okně poslaly alespoň jednu zprávu. Nejde o inventář ani důkaz, že jsou právě online. Pro kontrolu úplnosti porovnáme výsledek se seznamem očekávaných zařízení.

---

### 5. Uživatelé s úspěšným interaktivním přihlášením

Dotaz:

```text
winlogbeat_winlog_channel:"Security" AND winlogbeat_event_code:4624 AND winlogbeat_winlog_event_data_LogonType:(2 OR 7 OR 10 OR 11) AND NOT winlogbeat_winlog_event_data_TargetUserName:/.*\$/ AND NOT winlogbeat_winlog_event_data_TargetUserName:(SYSTEM OR "ANONYMOUS LOGON" OR "LOCAL SERVICE" OR "NETWORK SERVICE") AND NOT winlogbeat_winlog_event_data_TargetUserName:/DWM-.*/ AND NOT winlogbeat_winlog_event_data_TargetUserName:/UMFD-.*/
```

Nastavení widgetu:

1. V `Search` ověříme dotaz a nastavíme `1 day ago - Now`.
2. Klikneme na **Create + / Aggregation**.
3. V **Group By / Row** vybereme `winlogbeat_winlog_event_data_TargetUserName`.
4. **Limit** nastavíme podle počtu uživatelů.
5. V **Metrics** vybereme **Latest Value**.
6. Jako pole vybereme `winlogbeat_@timestamp` a název `Poslední přihlášení`.
7. V **Sort** zvolíme `Poslední přihlášení / Descending`.
8. V **Visualization** vybereme `Data Table`.
9. Widget pojmenujeme `Uživatelé s úspěšným přihlášením` a uložíme.

Pro počet unikátních uživatelů vytvoříme druhý widget bez `Group By`, s metrikou **Cardinality** nad polem `winlogbeat_winlog_event_data_TargetUserName` a vizualizací `Single Number`.

⚠️ Widget zobrazuje uživatele zachycené v událostech `4624` vybraných typů. Neukazuje aktuálně přihlášené relace a z absence uživatele nelze odvodit, že není přihlášený na zařízení, které neposílá Security log.

---

### 6. Neúspěšná přihlášení

Dotaz:

```text
winlogbeat_winlog_channel:"Security" AND winlogbeat_event_code:4625
```

#### Celkový počet

1. Vytvoříme **Aggregation**.
2. `Group By` nepřidáváme.
3. V **Metrics** zvolíme **Count** a pole necháme prázdné.
4. V **Visualization** zvolíme `Single Number`.
5. Widget uložíme jako `Neúspěšná přihlášení za 24h`.

#### Trend v čase

1. Nad stejným dotazem vytvoříme další **Aggregation**.
2. V **Group By / Row** vybereme `timestamp`.
3. V nastavení časového seskupení ponecháme interval `Auto`, případně zvolíme `1 hour`.
4. V **Metrics** zvolíme **Count** a pole necháme prázdné.
5. V **Visualization** vybereme `Line Chart`.
6. Widget uložíme jako `Trend neúspěšných přihlášení`.

Pro rychlé určení zdroje lze přidat třetí `Data Table`, seskupit ji podle `winlogbeat_winlog_computer_name` nebo pole se zdrojovou IP z konkrétní události a použít metriku `Count`.

---

### 7. PowerShell a LDAP

#### Poslední PowerShell 4104 události

```text
winlogbeat_winlog_channel:"Microsoft-Windows-PowerShell/Operational" AND winlogbeat_event_code:4104
```

1. Klikneme na **Create + / Message Table**.
2. Ponecháme sloupce `timestamp`, `winlogbeat_winlog_computer_name`, `winlogbeat_event_code` a `message`.
3. Widget pojmenujeme `PowerShell 4104 - poslední události`.

⚠️ Script Block Logging může obsahovat citlivé příkazy nebo data. Přístup k dashboardu omezíme jen na oprávněné techniky.

#### Nezabezpečené LDAP vazby

```text
winlogbeat_winlog_channel:"Directory Service" AND winlogbeat_event_code:2889
```

Vytvoříme dva widgety:

* `Single Number`: metrika **Count**, pole prázdné
* `Message Table`: sloupce `timestamp`, `winlogbeat_winlog_computer_name` a `message`

Číselný widget ukáže, zda se problém stále objevuje, a tabulka umožní dohledat klienta a identitu.

---

### 8. Microsoft DHCP

Dotaz:

```text
winlogbeat_winlog_computer_name:DHCP01 AND (winlogbeat_winlog_channel:"Microsoft-Windows-DHCP Server Events/Admin" OR winlogbeat_winlog_channel:"Microsoft-Windows-DHCP Server Events/Operational")
```

#### Přehled typů událostí

1. Vytvoříme **Aggregation**.
2. V **Group By / Row** vybereme `winlogbeat_event_code`.
3. V **Metrics** přidáme **Count** s prázdným polem.
4. Přidáme druhou metriku **Latest Value** nad `winlogbeat_@timestamp` a pojmenujeme ji `Poslední událost`.
5. V **Sort** zvolíme `Poslední událost / Descending`.
6. V **Visualization** zvolíme `Data Table`.
7. Widget uložíme jako `Microsoft DHCP - typy událostí`.

#### Poslední události

Nad stejným dotazem vytvoříme `Message Table` se sloupci `timestamp`, `winlogbeat_winlog_computer_name`, `winlogbeat_event_code` a `message`.

📌 Bez znalosti konkrétních event ID netvrdíme, že každá událost je chyba; význam kódu ověříme v textu zprávy a v dokumentaci Microsoftu.

---

### 9. Síť a Hyper-V

Pro MikroTik a UniFi vytvoříme `Message Table` nad ověřenými dotazy ze sekce 2 a ponecháme `timestamp`, `source` a `message`. Další `Data Table` lze seskupit podle `source`, přidat metriku `Latest Value` nad `timestamp` a získat tak poslední událost od každého síťového zařízení.

Pro Hyper-V je praktičtější sbírat administrační kanály `Microsoft-Windows-Hyper-V-VMMS/Admin` a `Microsoft-Windows-Hyper-V-Worker/Admin` než hledat slova `error` a `fail` pouze v obecných logách `System` a `Application`. Nad těmito kanály vytvoříme:

* `Message Table` s posledními konkrétními událostmi
* `Data Table` seskupenou podle `winlogbeat_event_code`, s metrikou `Count`

Pokud potřebujeme pouze chyby, filtrujeme podle skutečné hodnoty pole s úrovní události nebo podle ověřeného seznamu event ID.

---

### 10. Co kontrolovat každé ráno

Na jednom dashboardu obvykle stačí:

* poslední zpráva od každého Windows počítače
* počet unikátních počítačů proti očekávanému počtu
* poslední interaktivní přihlášení uživatelů
* počet a trend událostí `4625`
* počet událostí `2889`
* poslední důležité DHCP, Hyper-V a síťové zprávy

⚠️ Dashboard není náhradou inventáře, aktivního monitoringu ani systému pro správu relací. Ukazuje pouze to, co lze odvodit ze zpráv přijatých ve zvoleném časovém okně.

---

### Shrnutí

✅ Uložená hledání používáme pro opakované šetření, dashboard pro rychlou provozní kontrolu.  
✅ Každý widget vytváříme nad dotazem ověřeným na skutečných zprávách.  
✅ Počítače a uživatele počítáme pomocí `Cardinality`, poslední aktivitu pomocí `Latest Value`.  
✅ Trend vytváříme jako `Aggregation` s `timestamp` a vizualizací `Line Chart`.  
⚠️ Widgety ukazují pouze události přijaté ve zvoleném časovém okně, nikoli úplný inventář nebo online stav.

---

### Zdroje

* [Graylog: Saved Searches](https://go2docs.graylog.org/current/making_sense_of_your_log_data/saved_searches.html)
* [Graylog: Dashboards](https://go2docs.graylog.org/current/interacting_with_your_log_data/dashboards.html)
* [Graylog: Widgets](https://go2docs.graylog.org/current/interacting_with_your_log_data/widgets.html)
* [Microsoft: Audit event 4624](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4624)
