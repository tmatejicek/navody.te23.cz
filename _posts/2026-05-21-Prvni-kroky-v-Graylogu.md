---
title: "První kroky v Graylogu"
date: 2026-05-21
tags: [Graylog, Logging, Monitoring, Windows]
layout: post
---

## První kroky v Graylogu

Tento článek je určený pro techniky, kteří už mají Graylog v organizaci nasazený a potřebují se rychle zorientovat v běžné práci s logy. Neřešíme zde instalaci, ale hlavně to, **kde hledat**, **jak psát dotazy** a **jak z výsledků vytěžit užitečné informace**.

Pokud Graylog teprve nasazujete, navazuje tento článek na [Instalaci Graylog Open na Debianu 12]({% post_url 2026-04-15-Instalace-Graylog-Open-na-Debianu-12-dvouserverove-reseni %}) a na [nastavení Winlogbeatu pro odesílání logů do Graylogu]({% post_url 2026-04-15-Nastaveni-Winlogbeat-pro-odesilani-Windows-event-logu-do-Graylogu %}).

📌 V příkladech níže předpokládáme, že v Graylogu používáme **výchozí prefixování polí** v Beats inputu. Proto pracujeme s poli jako `winlogbeat_winlog_computer_name`, `winlogbeat_winlog_channel` nebo `winlogbeat_event_code`.

---

### 1. K čemu Graylog slouží

Graylog funguje jako centrální místo pro:

* vyhledávání Windows Event Logů
* analýzu Sysmon událostí
* dohled nad PowerShell aktivitou
* hledání LDAP událostí z doménových řadičů
* rychlé dohledání problémů podle času, počítače nebo typu události

V praxi to znamená, že když řešíme incident, problém s přihlášením, podezřelou aktivitu nebo chybu na konkrétním serveru, první místo, kam se díváme, bývá často právě **Graylog Search**.

---

### 2. Kde začít ve webovém rozhraní

Po přihlášení do Graylogu nás obvykle nejvíc zajímají tyto části:

* **Search** - hlavní místo pro hledání a analýzu logů
* **Streams** - logické rozdělení dat do skupin
* **Dashboards** - uložené přehledy a widgety
* **Alerts / Events** - navazující automatizace a upozornění

Pro běžného technika je nejdůležitější záložka **Search**. Právě tam probíhá většina každodenní práce.

---

### 3. Základní pracovní postup v Graylogu

Při hledání v Graylogu je dobré držet jednoduchý postup:

1. Nejprve nastavíme správné **časové okno**.
2. Potom případně omezíme hledání na vhodný **stream**.
3. Teprve potom píšeme vlastní **query**.
4. Z výsledků si otevřeme konkrétní zprávu a pokračujeme podle jejích polí.

📌 Pokud hledání nevrací očekávaná data, první dvě věci ke kontrole jsou téměř vždy:

* špatně zvolený časový rozsah
* zapomenutý nebo nevhodný filtr na stream

---

### 4. První pravidlo: časové okno

Graylog vždy hledá jen v časovém rozsahu, který je aktuálně nastavený. To je nejčastější důvod, proč technik „nic nevidí“, i když se událost skutečně stala.

Doporučení pro praxi:

* při právě probíhajícím problému začneme například na **Last 15 minutes** nebo **Last 1 hour**
* při dohledávání starší události použijeme **Absolute time range**
* pokud si nejsme jistí, raději časové okno dočasně rozšíříme

📌 Když hledáme například neúspěšná přihlášení nebo PowerShell aktivitu, je časové omezení často důležitější než samotná query syntaxe.

---

### 5. Druhé pravidlo: stream

Graylog standardně hledá ve všech datech, ke kterým má uživatel přístup. To ale nemusí být vždy výhodné.

Proto je dobré hledání podle potřeby omezit na konkrétní **stream**:

* například jen na Windows logy
* jen na bezpečnostní logy
* nebo jen na konkrétní skupinu zařízení

Pokud si nejsme jistí, začneme ve výchozím streamu **All messages** a teprve potom hledání zužujeme.

📌 Stream není totéž co query. Stream pomáhá zmenšit objem dat ještě předtím, než začneme pracovat s konkrétními poli a hodnotami.

---

### 6. Základní syntaxe dotazů

Graylog používá vyhledávací syntaxi podobnou **Lucene query syntax**.

Nejčastější základy:

* hledání textu:

```text
error
```

* hledání podle pole:

```text
winlogbeat_winlog_computer_name:SERVER01
```

* přesná fráze v uvozovkách:

```text
winlogbeat_winlog_channel:"Directory Service"
```

* kombinace podmínek:

```text
winlogbeat_winlog_computer_name:SERVER01 AND winlogbeat_event_code:4625
```

* více možností:

```text
winlogbeat_event_code:(4624 OR 4625)
```

* vyloučení:

```text
winlogbeat_winlog_channel:"Security" NOT winlogbeat_winlog_computer_name:TESTPC01
```

📌 Operátory `AND`, `OR` a `NOT` je vhodné psát velkými písmeny.  
📌 Pokud hodnota obsahuje mezeru nebo lomítko, používáme uvozovky.

---

### 7. Praktické dotazy

Níže jsou příklady dotazů, které dávají smysl v prostředí, kde do Graylogu posíláme Windows logy přes Winlogbeat.

#### 7.1 Logy z konkrétního počítače

```text
winlogbeat_winlog_computer_name:SERVER01
```

Použijeme, když řešíme konkrétní server nebo stanici.

#### 7.2 Neúspěšná přihlášení ve Security logu

```text
winlogbeat_winlog_channel:"Security" AND winlogbeat_event_code:4625
```

Vhodné pro rychlou kontrolu neúspěšných přihlášení.

#### 7.3 Úspěšná a neúspěšná přihlášení dohromady

```text
winlogbeat_winlog_channel:"Security" AND winlogbeat_event_code:(4624 OR 4625)
```

Dobré pro základní přehled přihlašovací aktivity.

#### 7.4 PowerShell script block logging

```text
winlogbeat_winlog_channel:"Microsoft-Windows-PowerShell/Operational" AND winlogbeat_event_code:4104
```

Použijeme při kontrole spuštěných PowerShell skriptů.

#### 7.5 Sysmon Process Create

```text
winlogbeat_winlog_channel:"Microsoft-Windows-Sysmon/Operational" AND winlogbeat_event_code:1
```

Praktické při hledání spuštěných procesů.

#### 7.6 LDAP události z doménového řadiče

```text
winlogbeat_winlog_channel:"Directory Service" AND winlogbeat_event_code:2889
```

Tento dotaz je vhodný například pro dohledání nešifrovaných LDAP dotazů. Navazuje na [článek o detekci nešifrovaného LDAPu]({% post_url 2025-05-15-Detekce-nesifrovanych-LDAP-dotazu-v-prostredi-Active-Directory %}).

#### 7.7 Kombinace počítače a event ID

```text
winlogbeat_winlog_computer_name:DC01 AND winlogbeat_winlog_channel:"Directory Service" AND winlogbeat_event_code:2889
```

Tohle je typický příklad dotazu, který už řeší velmi konkrétní situaci.

---

### 8. Jak číst detail jedné zprávy

Když najdeme zajímavý záznam, otevřeme si jeho detail. Tam bývá nejvíc užitečných informací.

Sledujeme hlavně:

* **čas události**
* **zdrojový počítač**
* **název logu / channel**
* **event code**
* **text zprávy**
* další pole, která Winlogbeat nebo Graylog z události vytáhl

Praktický postup:

* najdeme jednu relevantní zprávu
* z jejího detailu zkopírujeme zajímavou hodnotu
* z této hodnoty postavíme další dotaz

Například:

* z `winlogbeat_winlog_computer_name` dohledáme další události z téhož serveru
* z `winlogbeat_event_code` rozšíříme hledání na stejný typ událostí
* z textu zprávy nebo z uživatele dohledáme související aktivitu

📌 Často je rychlejší začít jednou konkrétní zprávou než se snažit hned napsat „dokonalý“ dotaz.

---

### 9. Uložení hledání a dashboardy

Pokud se k nějakému hledání vracíme opakovaně, má smysl si ho uložit.

**Saved search** je vhodný, když:

* používáme stejný dotaz opakovaně
* chceme mít připravenou šablonu hledání
* potřebujeme rychle navázat na předchozí analýzu

**Dashboard** je vhodný, když:

* chceme data sledovat průběžně
* potřebujeme přehled pro tým nebo vedoucího
* potřebujeme mít na jedné obrazovce více widgetů najednou

Graylog umožňuje převést hledání do dashboardu přímo ze Search stránky. To se hodí zejména u dotazů, které se opakují každý den.

---

### 10. Nejčastější chyby při práci s Graylogem

Pokud hledání nefunguje podle očekávání, nejčastější příčiny bývají:

* časové okno je příliš úzké nebo úplně špatně
* hledáme ve špatném streamu
* je překlep v názvu fieldu
* pole obsahuje mezery nebo speciální znaky a chybí uvozovky
* hledaná data se do Graylogu vůbec neposílají

Příklady typických problémů:

* `winlogbeat_event_code:2889` nic nevrací, protože log `Directory Service` nebyl ve Winlogbeatu povolen
* `4104` nic nevrací, protože není zapnuté PowerShell logging policy
* Sysmon dotazy nic nevrací, protože Sysmon na stanici není nainstalovaný

📌 Když dotaz nic nevrací, není to vždy chyba v Graylogu. Velmi často je problém už na straně zdrojového systému nebo ingestu.

---

## Shrnutí

✅ Pro běžného technika je nejdůležitější část Graylogu záložka **Search**  
✅ Nejdřív vždy kontrolujeme **časové okno**, potom **stream** a teprve pak samotný dotaz  
✅ Při práci s Windows logy se vyplatí znát hlavně pole `winlogbeat_winlog_computer_name`, `winlogbeat_winlog_channel` a `winlogbeat_event_code`  
✅ Praktická práce v Graylogu stojí hlavně na jednoduchých opakovatelných dotazech  
✅ Detail jedné zprávy bývá nejlepší výchozí bod pro další analýzu
