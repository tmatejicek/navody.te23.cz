---
title: "Praktické dashboardy a uložená hledání v Graylogu"
date: 2026-05-21
tags: [Graylog, Dashboards, Monitoring, Windows, Networking]
layout: post
---

## Praktické dashboardy a uložená hledání v Graylogu

Jakmile už do Graylogu posíláme logy z Windows serverů, stanic a síťových prvků, další logický krok je udělat si z něj nástroj pro každodenní provoz. K tomu nejlépe slouží **uložená hledání** a **dashboardy**.

Tento článek je zaměřený prakticky. Představuje menší organizaci, kde máme například:

* Active Directory
* Hyper-V hosty
* MikroTik router
* UniFi gateway, switche a access pointy

Cílem není vytvořit „krásný“ dashboard, ale dashboard, který technikovi ráno během pár minut ukáže, jestli se v infrastruktuře děje něco podezřelého nebo rozbitého.

Navazuje na článek [První kroky v Graylogu]({% post_url 2026-05-21-Prvni-kroky-v-Graylogu %}) a na [nastavení Winlogbeatu pro odesílání logů do Graylogu]({% post_url 2026-04-15-Nastaveni-Winlogbeat-pro-odesilani-Windows-event-logu-do-Graylogu %}).

---

### 1. Uložené hledání vs. dashboard

Graylog rozlišuje dvě věci:

* **Saved search** - uloží si dotaz, časové okno, streamy a případné filtry
* **Dashboard** - zobrazí vybraná data pomocí widgetů

Prakticky:

* **saved search** používáme pro opakované šetření
* **dashboard** používáme pro rychlý přehled

Dobrá praxe je začít vždy nejprve saved searchem. Až když se dotaz osvědčí, uděláme z něj widget nebo celý dashboard.

📌 Graylog dokumentuje, že saved search může uchovávat nejen samotný query string, ale i časové okno, filtry a parametry. To je důvod, proč je lepší stavět dashboardy právě nad promyšlenými hledáními a ne všechno klikat od nuly.

---

### 2. Doporučený pracovní postup

Praktický a dobře udržitelný postup bývá tento:

1. Nejprve si v **Search** ověříme, že dotaz opravdu vrací užitečná data.
2. Hledání si uložíme jako **saved search**.
3. Ze saved search nebo přímo ze Search stránky vytvoříme widget.
4. Widget přidáme do nového nebo existujícího dashboardu.
5. Dashboard doplníme o krátký textový widget s vysvětlením, co má technik sledovat.

Tento postup je lepší než budovat dashboard rovnou, protože:

* dotazy se lépe ladí v Search stránce
* stejné hledání můžeme použít na více dashboardech
* změna dotazu je pak jednodušší a přehlednější

---

### 3. Jak vytvářet saved searches

Ze Search stránky:

1. nastavíme query
2. zvolíme správné časové okno
3. případně omezíme stream
4. klikneme na **Save**
5. uložíme hledání pod srozumitelným názvem

Dobré názvy saved searches jsou například:

* `Neúspěšná přihlášení za 24h`
* `PowerShell 4104 za 24h`
* `LDAP 2889 za 24h`
* `MikroTik chyby a přihlášení`
* `UniFi události za 24h`
* `Hyper-V hosty - chyby a varování`

📌 Graylog si pamatuje i historii dotazů, ale to nestačí jako náhrada za saved searches. Pro provozní použití se vyplatí ukládat jen ta hledání, která se budou skutečně opakovat.

---

### 4. Praktická uložená hledání

Níže jsou příklady saved searches, které dávají smysl pro menší infrastrukturu.

#### 4.1 Neúspěšná přihlášení do Windows

Doporučený název saved search:

```text
Neúspěšná přihlášení za 24h
```

```text
winlogbeat_winlog_channel:"Security" AND winlogbeat_event_code:4625
```

Doporučené časové okno:

* `Last 24 hours`

K čemu se hodí:

* ranní kontrola přihlašovacích problémů
* odhalení bruteforce nebo špatně uložených hesel ve službách

#### 4.2 PowerShell script block logging

Doporučený název saved search:

```text
PowerShell 4104 za 24h
```

```text
winlogbeat_winlog_channel:"Microsoft-Windows-PowerShell/Operational" AND winlogbeat_event_code:4104
```

K čemu se hodí:

* dohledání spuštěných PowerShell skriptů
* rychlá kontrola podezřelé aktivity na serverech

#### 4.3 Nešifrované LDAP dotazy

Doporučený název saved search:

```text
LDAP 2889 za 24h
```

```text
winlogbeat_winlog_channel:"Directory Service" AND winlogbeat_event_code:2889
```

K čemu se hodí:

* dohledání zařízení nebo aplikací, které stále používají nešifrovaný LDAP

Navazuje na [článek o detekci nešifrovaného LDAPu]({% post_url 2025-05-15-Detekce-nesifrovanych-LDAP-dotazu-v-prostredi-Active-Directory %}).

#### 4.4 Hyper-V hosty - chyby a varování

Doporučený název saved search:

```text
Hyper-V hosty - chyby a varování
```

Pokud do Graylogu posíláme z Hyper-V hostů alespoň standardní `Application` a `System` logy, může být praktické například toto hledání:

```text
winlogbeat_source:(HV01 OR HV02) AND (winlogbeat_winlog_channel:"System" OR winlogbeat_winlog_channel:"Application") AND (error OR fail* OR critical)
```

K čemu se hodí:

* rychlá kontrola problémů na hostech virtualizace
* dohledání výpadků služeb, storage nebo síťových problémů

📌 Názvy `HV01` a `HV02` upravíme podle skutečných hostname.  
📌 Pokud sbíráme i další Hyper-V specifické logy, můžeme saved search rozšířit o další channel názvy.

#### 4.5 MikroTik - chyby, přihlášení a změny konfigurace

Doporučený název saved search:

```text
MikroTik chyby a přihlášení
```

U síťových prvků bývá nejpraktičtější začít přes pole `source` a `message`, protože přesný formát syslogu se může lišit podle verze a konfigurace.

Například:

```text
source:mikrotik01 AND (error OR critical OR login OR failure OR configuration)
```

K čemu se hodí:

* kontrola přihlášení administrátorů
* podezřelé změny konfigurace
* chybové stavy routeru

#### 4.6 UniFi - důležité infrastrukturní události

Doporučený název saved search:

```text
UniFi události za 24h
```

Například:

```text
source:(unifi-gateway01 OR unifi-switch01 OR unifi-ap01) AND (error OR disconnect* OR uplink OR wan OR dhcp)
```

K čemu se hodí:

* odpojování zařízení
* výpadky uplinku
* problémy s WAN nebo DHCP

📌 U MikroTiku a UniFi je vhodné nejdřív v Search stránce ručně projít několik reálných zpráv a podle nich si doladit konkrétní textové výrazy v query.

---

### 5. Jak z hledání udělat widget

Po provedení hledání klikneme v Search stránce na **Create +** a vybereme vhodný typ widgetu.

V praxi se nejčastěji hodí:

* **Aggregation widget** - graf, single number nebo tabulka nad agregovanými daty
* **Message table widget** - tabulka konkrétních zpráv
* **Text (Markdown) widget** - vysvětlení, co dashboard sleduje

Doporučení:

* pro „kolik toho bylo“ použijeme **Single Number**
* pro vývoj v čase použijeme **Time Series**
* pro přehled top hodnot použijeme **Data Table / Top values**
* pro konkrétní poslední události použijeme **Message Table**

📌 Graylog uvádí, že dashboard má vlastní search bar, ale ten pouze **dočasně přepisuje** dotazy widgetů. Nemění trvale jejich uloženou logiku. Proto je důležité mít widgety správně nastavené už při jejich vytvoření.

---

### 6. Praktický dashboard č. 1 - Ranní kontrola infrastruktury

Tohle je nejdůležitější dashboard pro běžný provoz. Technik si ho otevře ráno a během několika minut ví, jestli se děje něco mimořádného.

Časové okno dashboardu:

* `Last 24 hours`

Právě tady dávají největší smysl widgety typu **aktivní počítače** a **aktivní uživatelé**, protože rychle ukážou, jestli prostředí „žije“ očekávaným způsobem.

Doporučené widgety:

#### 6.1 Aktivní Windows počítače

Zdrojový dotaz pro widget:

```text
winlogbeat_source:*
```

Widget:

* **Top values** nad `winlogbeat_source`
* případně **Single Number** s počtem unikátních počítačů

Smysl:

* rychle ukáže, které počítače v poslední době posílaly logy
* pomůže odhalit neaktivní nebo chybějící zdroje logů

Tento přehled většinou není potřeba ukládat jako samostatný saved search. V praxi bývá praktičtější vytvořit ho rovnou jako widget na dashboardu.

#### 6.2 Aktivní uživatelé

Zdrojový dotaz pro widget:

```text
winlogbeat_winlog_channel:"Security" AND winlogbeat_event_code:4624
```

Widget:

* **Top values** nad `winlogbeat_winlog_event_data_TargetUserName`

Smysl:

* rychlý přehled, kdo se v prostředí v posledních hodinách nebo dnech přihlašoval
* užitečné při ranní kontrole i při dohledávání podezřelé aktivity

📌 V některých prostředích dává smysl z tohoto widgetu odfiltrovat technické účty, `ANONYMOUS LOGON` nebo počítačové účty končící znakem `$`.  
📌 Stejně jako u aktivních počítačů jde obvykle spíš o widget než o saved search.

#### 6.3 Počet neúspěšných přihlášení

Saved search:

```text
winlogbeat_winlog_channel:"Security" AND winlogbeat_event_code:4625
```

Widget:

* **Single Number**
* metrika `count()`

#### 6.4 Trend neúspěšných přihlášení

Stejný dotaz jako výše.

Widget:

* **Time Series**
* metrika `count()`

Smysl:

* ukáže, jestli jde o ojedinělý problém, nebo o soustavný nárůst

#### 6.5 Poslední PowerShell 4104 události

Saved search:

```text
winlogbeat_winlog_channel:"Microsoft-Windows-PowerShell/Operational" AND winlogbeat_event_code:4104
```

Widget:

* **Message Table**

Doporučené sloupce:

* timestamp
* `winlogbeat_source`
* `winlogbeat_event_code`
* message

#### 6.6 LDAP 2889 za posledních 24 hodin

Saved search:

```text
winlogbeat_winlog_channel:"Directory Service" AND winlogbeat_event_code:2889
```

Widget:

* **Single Number**
* nebo **Message Table**

#### 6.7 Poslední důležité síťové události

Saved search:

```text
(source:mikrotik01 OR source:unifi-gateway01 OR source:unifi-switch01 OR source:unifi-ap01) AND (error OR critical OR disconnect* OR wan OR uplink)
```

Widget:

* **Message Table**

Smysl:

* rychlý přehled síťových problémů bez nutnosti ručně procházet každý zdroj zvlášť

---

### 7. Praktický dashboard č. 2 - Active Directory a bezpečnost

Tento dashboard se hodí, pokud chceme mít bokem samostatný pohled na doménové řadiče a bezpečnostní události.

Časové okno:

* `Last 24 hours`

Doporučené widgety:

* **Top values** nad `winlogbeat_source` pro event `4625`
* **Message Table** pro `2889`
* **Time Series** pro `4104`
* **Message Table** pro logy z doménových řadičů s filtrem na konkrétní servery

Příklad dotazu:

```text
winlogbeat_source:(DC01 OR DC02) AND (winlogbeat_event_code:4625 OR winlogbeat_event_code:2889 OR winlogbeat_event_code:4104)
```

Tento dashboard je vhodný, když:

* technik řeší problém s přihlášením
* kontroluje LDAP klienty
* potřebuje rychle dohledat podezřelou PowerShell aktivitu

---

### 8. Praktický dashboard č. 3 - Síť a virtualizace

Třetí dashboard se hodí pro infrastrukturu mimo Active Directory.

Doporučené widgety:

* **Message Table** pro MikroTik chyby a přihlášení
* **Message Table** pro UniFi události
* **Single Number** nebo **Time Series** pro Hyper-V chyby
* **Top values** nad `source`, pokud chceme vidět, které zařízení generuje nejvíc událostí

Příklad pro Hyper-V:

```text
winlogbeat_source:(HV01 OR HV02) AND (winlogbeat_winlog_channel:"System" OR winlogbeat_winlog_channel:"Application") AND (error OR fail* OR critical)
```

Příklad pro síť:

```text
source:(mikrotik01 OR unifi-gateway01 OR unifi-switch01 OR unifi-ap01) AND (error OR critical OR warning OR disconnect* OR uplink OR wan)
```

📌 U menší organizace je často praktičtější mít tento dashboard jednodušší. Není nutné se snažit zobrazit všechno najednou.

---

### 9. Co se na dashboard hodí a co ne

Na dashboard se dobře hodí:

* počet událostí
* trend v čase
* top zdroje
* posledních několik důležitých zpráv

Na dashboard se naopak moc nehodí:

* příliš široké message table bez filtru
* složité jednorázové investigativní dotazy
* widgety, které vrací stovky nerelevantních zpráv

Praktické pravidlo:

* pokud je cílem **rychlý přehled**, patří to na dashboard
* pokud je cílem **hlubší šetření**, patří to spíš do saved search

---

### 10. Doporučení pro udržitelnost

Aby dashboardy zůstaly použitelné i za několik měsíců, je dobré držet několik pravidel:

* dávat saved searchům i dashboardům srozumitelné názvy
* mít jeden dashboard pro ranní kontrolu a další jen pro specializované oblasti
* nepřidávat na jeden dashboard příliš mnoho widgetů
* používat textový widget s krátkým popisem účelu dashboardu
* jednou za čas projít, které widgety už nejsou užitečné

Pokud více techniků sdílí stejný Graylog, dává smysl dashboardy a saved searches i sdílet nebo organizovat do kolekcí.

---

## Shrnutí

✅ Saved searches jsou nejlepší základ pro opakovanou práci v Graylogu  
✅ Dashboard má sloužit hlavně pro rychlou orientaci, ne pro detailní šetření  
✅ Pro menší organizaci obvykle stačí jeden ranní dashboard a několik specializovaných pohledů  
✅ U Windows logů se opíráme hlavně o `winlogbeat_source`, `winlogbeat_winlog_channel` a `winlogbeat_event_code`  
✅ U MikroTiku a UniFi je praktické začít přes `source` a `message` a query doladit podle reálných logů
