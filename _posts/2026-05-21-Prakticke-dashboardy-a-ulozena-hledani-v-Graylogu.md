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
winlogbeat_winlog_computer_name:(HV01 OR HV02) AND (winlogbeat_winlog_channel:"System" OR winlogbeat_winlog_channel:"Application") AND (error OR fail* OR critical)
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

#### 4.7 Microsoft DHCP - administrace a provoz

Doporučený název saved search:

```text
Microsoft DHCP události za 24h
```

Pokud máme DHCP server například `DHCP01`, může být praktické toto hledání:

```text
winlogbeat_winlog_computer_name:DHCP01 AND (winlogbeat_winlog_channel:"Microsoft-Windows-DHCP Server Events/Admin" OR winlogbeat_winlog_channel:"Microsoft-Windows-DHCP Server Events/Operational")
```

K čemu se hodí:

* změny scope, option a dalších DHCP nastavení
* kontrola startu a chyb DHCP služby
* dohledání stavových změn u DHCP failoveru

📌 Na DHCP serverech bývají pro běžný provoz nejzajímavější kanály `Microsoft-Windows-DHCP Server Events/Admin` a `Microsoft-Windows-DHCP Server Events/Operational`.  
📌 Pokud tyto události v Graylogu nevidíme, je potřeba je přidat do sběru ve Winlogbeatu na DHCP serveru.

---

### 5. Jak z hledání udělat widget

Po provedení hledání klikneme v Search stránce na **Create +** a vybereme vhodný typ widgetu.

V praxi se nejčastěji hodí:

* **Aggregation widget** - graf, single number nebo tabulka nad agregovanými daty
* **Message table widget** - tabulka konkrétních zpráv
* **Text (Markdown) widget** - vysvětlení, co dashboard sleduje

Doporučení:

* pro „kolik toho bylo“ použijeme **Single Number**
* pro vývoj v čase použijeme **Line Chart** nebo **Bar Chart**
* pro přehled top hodnot použijeme **Data Table**
* pro konkrétní poslední události použijeme **Message Table**

📌 V aktuálním Graylogu není `Time Series` samostatný typ widgetu. Pro trend v čase vytvoříme **Aggregation widget** a jako vizualizaci zvolíme typicky `Line Chart`.  
📌 Akce `Show Top Values` není samostatný typ widgetu. Je to jen rychlá cesta k vytvoření **Data Table** s nejčastějšími hodnotami pole.  
📌 Graylog uvádí, že dashboard má vlastní search bar, ale ten pouze **dočasně přepisuje** dotazy widgetů. Nemění trvale jejich uloženou logiku. Proto je důležité mít widgety správně nastavené už při jejich vytvoření.

---

### 6. Praktický dashboard č. 1 - Ranní kontrola infrastruktury

Časové okno:

* `Last 24 hours`

Na tento dashboard dává smysl dát jen několik widgetů, které technik zkontroluje během pár minut.

#### 6.1 Počítače, které poslaly logy

Dotaz:

```text
_exists_:winlogbeat_winlog_computer_name
```

Postup:

1. V `Search` vložíme dotaz výše.
2. Klikneme na `Create +` a vybereme `Aggregation`.
3. V `Visualization` zvolíme `Data Table`.
4. V `Group by Row` vybereme `winlogbeat_winlog_computer_name`.
5. V `Metrics` přidáme `Count` a pole necháme prázdné.
6. Volitelně ponecháme i automaticky přidané `Percentage`, pokud chceme vidět podíl jednotlivých počítačů.
7. Widget uložíme do dashboardu.

Výsledek:

* seznam počítačů a počet zpráv od každého z nich
* rychlá kontrola, jestli některý server nebo stanice nepřestaly posílat logy

Pokud chceme místo seznamu jen číslo, vytvoříme druhý widget `Single Number` a jako metriku zvolíme funkci pro unikátní počet hodnot nad `winlogbeat_winlog_computer_name`.

#### 6.2 Uživatelé s úspěšným přihlášením

Dotaz:

```text
winlogbeat_winlog_channel:"Security" AND winlogbeat_event_code:4624
```

Postup:

1. V `Search` vložíme dotaz výše.
2. Klikneme na `Create +` a vybereme `Aggregation`.
3. V `Visualization` zvolíme `Data Table`.
4. V `Group by Row` vybereme `winlogbeat_winlog_event_data_TargetUserName`.
5. V `Metrics` přidáme `Count` a pole necháme prázdné.
6. Volitelně ponecháme i `Percentage`, pokud chceme vidět podíl jednotlivých uživatelů.
7. Widget uložíme do dashboardu.

Výsledek:

* seznam uživatelů a počet odpovídajících logon událostí
* rychlý přehled přihlašovací aktivity za posledních 24 hodin

V praxi bývá vhodné odfiltrovat technické účty, `ANONYMOUS LOGON` a počítačové účty končící znakem `$`. Pokud chceme místo seznamu jen číslo, vytvoříme druhý widget `Single Number` a zvolíme metriku pro unikátní počet hodnot nad `winlogbeat_winlog_event_data_TargetUserName`.

#### 6.3 Počet neúspěšných přihlášení

Dotaz:

```text
winlogbeat_winlog_channel:"Security" AND winlogbeat_event_code:4625
```

Postup:

1. V `Search` vložíme dotaz výše.
2. Klikneme na `Create +` a vybereme `Aggregation`.
3. V `Visualization` zvolíme `Single Number`.
4. V `Metric` nastavíme `count()`.
5. Widget uložíme do dashboardu.

Výsledek:

* jedno číslo, které rychle ukáže, jestli v posledních 24 hodinách nevyskočil počet chybných přihlášení

#### 6.4 Trend neúspěšných přihlášení

Dotaz:

```text
winlogbeat_winlog_channel:"Security" AND winlogbeat_event_code:4625
```

Postup:

1. V `Search` vložíme dotaz výše.
2. Klikneme na `Create +` a vybereme `Aggregation`.
3. V `Visualization` zvolíme `Line Chart`.
4. V `Group by Row` vybereme `timestamp`.
5. V `Metric` nastavíme `count()`.
6. Widget uložíme do dashboardu.

Výsledek:

* graf, který ukáže, jestli jde o ojedinělý problém, nebo o soustavný nárůst

#### 6.5 Poslední PowerShell 4104 události

Dotaz:

```text
winlogbeat_winlog_channel:"Microsoft-Windows-PowerShell/Operational" AND winlogbeat_event_code:4104
```

Postup:

1. V `Search` vložíme dotaz výše.
2. Klikneme na `Create +` a vybereme `Message Table`.
3. V tabulce ponecháme nebo doplníme sloupce `timestamp`, `winlogbeat_winlog_computer_name`, `winlogbeat_event_code` a `message`.
4. Widget uložíme do dashboardu.

Výsledek:

* poslední PowerShell script block události na jednom místě

#### 6.6 LDAP 2889 za posledních 24 hodin

Dotaz:

```text
winlogbeat_winlog_channel:"Directory Service" AND winlogbeat_event_code:2889
```

Postup pro rychlý přehled:

1. V `Search` vložíme dotaz výše.
2. Klikneme na `Create +` a vybereme `Aggregation`.
3. V `Visualization` zvolíme `Single Number`.
4. V `Metric` nastavíme `count()`.
5. Widget uložíme do dashboardu.

Postup pro detail:

1. Nad stejným dotazem klikneme na `Create +` a vybereme `Message Table`.
2. V tabulce ponecháme hlavně `timestamp`, `winlogbeat_winlog_computer_name` a `message`.
3. Widget uložíme do dashboardu.

Výsledek:

* jedno číslo pro rychlou kontrolu
* jedna tabulka pro dohledání konkrétních LDAP klientů

#### 6.7 Poslední důležité síťové události

Dotaz:

```text
(source:mikrotik01 OR source:unifi-gateway01 OR source:unifi-switch01 OR source:unifi-ap01) AND (error OR critical OR disconnect* OR wan OR uplink)
```

Postup:

1. V `Search` vložíme dotaz výše.
2. Klikneme na `Create +` a vybereme `Message Table`.
3. V tabulce ponecháme hlavně `timestamp`, `source` a `message`.
4. Widget uložíme do dashboardu.

Výsledek:

* rychlý přehled síťových problémů bez nutnosti ručně procházet každý zdroj zvlášť

---

### 7. Praktický dashboard č. 2 - Active Directory a bezpečnost

Tento dashboard se hodí, pokud chceme mít bokem samostatný pohled na doménové řadiče a bezpečnostní události.

Časové okno:

* `Last 24 hours`

Doporučené widgety:

* **Data Table** seskupená podle `winlogbeat_winlog_computer_name` pro event `4625`
* **Message Table** pro `2889`
* **Line Chart** pro `4104`
* **Message Table** pro logy z doménových řadičů s filtrem na konkrétní servery

Příklad dotazu:

```text
winlogbeat_winlog_computer_name:(DC01 OR DC02) AND (winlogbeat_event_code:4625 OR winlogbeat_event_code:2889 OR winlogbeat_event_code:4104)
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
* **Single Number** nebo **Line Chart** pro Hyper-V chyby
* **Message Table** pro poslední Microsoft DHCP události
* **Data Table** seskupená podle `winlogbeat_event_code` pro Microsoft DHCP
* **Data Table** seskupená podle `source`, pokud chceme vidět, které zařízení generuje nejvíc událostí

Příklad pro Hyper-V:

```text
winlogbeat_winlog_computer_name:(HV01 OR HV02) AND (winlogbeat_winlog_channel:"System" OR winlogbeat_winlog_channel:"Application") AND (error OR fail* OR critical)
```

Příklad pro síť:

```text
source:(mikrotik01 OR unifi-gateway01 OR unifi-switch01 OR unifi-ap01) AND (error OR critical OR warning OR disconnect* OR uplink OR wan)
```

Příklad pro Microsoft DHCP:

```text
winlogbeat_winlog_computer_name:DHCP01 AND (winlogbeat_winlog_channel:"Microsoft-Windows-DHCP Server Events/Admin" OR winlogbeat_winlog_channel:"Microsoft-Windows-DHCP Server Events/Operational")
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
✅ U Windows logů se opíráme hlavně o `winlogbeat_winlog_computer_name`, `winlogbeat_winlog_channel` a `winlogbeat_event_code`  
✅ U MikroTiku a UniFi je praktické začít přes `source` a `message` a query doladit podle reálných logů
