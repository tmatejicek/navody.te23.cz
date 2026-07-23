---
title: "První kroky v Graylogu"
date: 2026-05-21
tags: [Graylog, Logging, Monitoring, Windows]
layout: post
---

## První kroky v Graylogu

Článek popisuje běžnou práci technika s již nasazeným Graylogem: výběr dat, psaní dotazů a kontrolu výsledků. Příklady předpokládají výchozí prefixování polí z Beats inputu.

📌 Pokud se pole v konkrétní instalaci jmenují jinak, vždy použijeme názvy z detailu skutečné zprávy.

Související postupy:

* [Instalace Graylog Open na Debianu 12]({% post_url 2026-04-15-Instalace-Graylog-Open-na-Debianu-12-dvouserverove-reseni %})
* [Nastavení Winlogbeatu]({% post_url 2026-04-15-Nastaveni-Winlogbeat-pro-odesilani-Windows-event-logu-do-Graylogu %})
* [Praktické dashboardy a uložená hledání]({% post_url 2026-05-21-Prakticke-dashboardy-a-ulozena-hledani-v-Graylogu %})

---

### 1. K čemu Graylog slouží

Graylog centralizuje události z Windows, serverů, aplikací a síťových prvků. Technik v něm může podle času, zdroje a polí dohledat například:

* chybu na konkrétním serveru
* úspěšná a neúspěšná přihlášení
* PowerShell a Sysmon události
* nezabezpečené LDAP vazby
* výpadky nebo změny na síťových prvcích

Pro každodenní práci jsou nejdůležitější:

* **Search**: hledání a analýza zpráv
* **Streams**: předem vymezené skupiny dat
* **Dashboards**: provozní přehledy
* **Alerts / Events**: automatická detekce a upozornění

---

### 2. Správné pořadí hledání

1. Nastavíme časový rozsah.
2. Podle potřeby vybereme jeden nebo více streamů.
3. Spustíme jednoduchý dotaz.
4. Otevřeme detail relevantní zprávy.
5. Další dotaz sestavíme z polí této zprávy.

💡 Při aktuálním problému začneme rozsahem `Last 15 minutes` nebo `Last 1 hour`. Pro událost se známým časem použijeme **Absolute time range**. Příliš široký rozsah zpomaluje dotaz a často přidá nerelevantní výsledky.

Pokud nevíme, ve kterém streamu zpráva je, neomezujeme nejprve žádný stream. Výběr streamu je filtr dat, nikoli součást textu dotazu.

---

### 3. Základní syntaxe

Hledání textu ve výchozích textových polích:

```text
error
```

Hledání podle pole:

```text
winlogbeat_winlog_computer_name:SERVER01
```

Přesná hodnota obsahující mezeru:

```text
winlogbeat_winlog_channel:"Directory Service"
```

Kombinace podmínek:

```text
winlogbeat_winlog_computer_name:SERVER01 AND winlogbeat_event_code:4625
```

Více hodnot stejného pole:

```text
winlogbeat_event_code:(4624 OR 4625)
```

Vyloučení:

```text
winlogbeat_winlog_channel:"Security" AND NOT winlogbeat_winlog_computer_name:TESTPC01
```

Existence pole:

```text
_exists_:winlogbeat_winlog_computer_name
```

📌 Operátory `AND`, `OR` a `NOT` píšeme velkými písmeny a složitější části uzavíráme do závorek.

❌ Samotné `*` ani wildcard začínající `*` nebo `?` tento parser nepřijme; pro ověření pole použijeme `_exists_:nazev_pole`. Názvy polí kopírujeme z detailu zprávy, protože jsou citlivé na přesný zápis.

---

### 4. Praktické dotazy

#### Zprávy z konkrétního počítače

```text
winlogbeat_winlog_computer_name:SERVER01
```

Hodnotu `SERVER01` nahradíme přesnou hodnotou z události. Winlogbeat může posílat krátké jméno nebo FQDN.

#### Neúspěšná přihlášení

```text
winlogbeat_winlog_channel:"Security" AND winlogbeat_event_code:4625
```

#### Úspěšná a neúspěšná přihlášení

```text
winlogbeat_winlog_channel:"Security" AND winlogbeat_event_code:(4624 OR 4625)
```

📌 Událost `4624` neznamená automaticky přihlášení člověka k ploše. Zahrnuje různé typy přihlášení včetně síťových a servisních. Dotaz na interaktivní uživatele je uveden v článku o [praktických dashboardech]({% post_url 2026-05-21-Prakticke-dashboardy-a-ulozena-hledani-v-Graylogu %}).

#### PowerShell Script Block Logging

```text
winlogbeat_winlog_channel:"Microsoft-Windows-PowerShell/Operational" AND winlogbeat_event_code:4104
```

📌 Události vznikají pouze tehdy, když je zapnutý odpovídající PowerShell logging.

#### Sysmon Process Create

```text
winlogbeat_winlog_channel:"Microsoft-Windows-Sysmon/Operational" AND winlogbeat_event_code:1
```

📌 Dotaz funguje pouze na počítačích s nainstalovaným a správně nakonfigurovaným Sysmonem.

#### Nezabezpečené LDAP vazby

```text
winlogbeat_winlog_channel:"Directory Service" AND winlogbeat_event_code:2889
```

Událost `2889` vzniká na doménovém řadiči po zapnutí diagnostiky **16 LDAP Interface Events** na úroveň `2`. Neznamená každý provoz na portu `389`, ale nezabezpečené LDAP vazby popsané v [samostatném článku]({% post_url 2025-05-15-Detekce-nesifrovanych-LDAP-dotazu-v-prostredi-Active-Directory %}).

#### Události Microsoft DHCP

```text
winlogbeat_winlog_computer_name:DHCP01 AND (winlogbeat_winlog_channel:"Microsoft-Windows-DHCP Server Events/Admin" OR winlogbeat_winlog_channel:"Microsoft-Windows-DHCP Server Events/Operational")
```

Kanály musí být přidané v konfiguraci Winlogbeatu na DHCP serveru.

---

### 5. Jak číst jednu zprávu

V detailu zprávy kontrolujeme:

* `timestamp`
* zdroj nebo `winlogbeat_winlog_computer_name`
* `winlogbeat_winlog_channel`
* `winlogbeat_event_code`
* `message`
* pole v `winlogbeat_winlog_event_data_*`

💡 Z názvu pole lze v Search vytvořit další podmínku. Hodnotu ale interpretujeme podle dokumentace daného poskytovatele události. Stejné event ID může mít význam pouze v kombinaci s konkrétním kanálem nebo providerem.

---

### 6. Když hledání nefunguje

#### Query parsing error

❌ Zkontrolujeme závorky, uvozovky, operátory a wildcard. Pro všechny zprávy s určitým polem nepoužijeme `field:*`, ale:

```text
_exists_:field
```

#### Unknown field

⚠️ Pole není známé v aktuálně prohledávaných indexech. Postup:

1. rozšíříme časový rozsah
2. zrušíme omezení na stream
3. najdeme jednu očekávanou zprávu podle textu nebo `source`
4. z detailu zkopírujeme skutečný název pole

❌ Nepoužíváme názvy jako `winlogbeat_source`, pokud je reálná zpráva neobsahuje.

#### Dotaz je validní, ale nic nevrací

Ověříme:

* zda událost ve Windows skutečně vznikla
* zda Winlogbeat daný kanál čte
* zda Beats input přijímá data
* zda stream událost nezpřístupňuje jen jiné skupině uživatelů
* zda timestamp leží ve vybraném rozsahu

---

### 7. Uložené hledání a dashboard

Opakovaný dotaz uložíme v `Search` přes **Save**. Saved search uchová dotaz, čas, streamy a filtry a hodí se pro další šetření.

Pro průběžný přehled vytvoříme widget přes **Create +** nebo hledání exportujeme do dashboardu. Každý widget má vlastní dotaz a časový rozsah; podrobný postup je v článku [Praktické dashboardy a uložená hledání v Graylogu]({% post_url 2026-05-21-Prakticke-dashboardy-a-ulozena-hledani-v-Graylogu %}).

---

### Shrnutí

✅ Nejdřív nastavíme časové okno, potom stream a až poté zpřesňujeme dotaz.  
✅ Názvy polí kopírujeme z detailu skutečné zprávy.  
✅ Pro ověření existence pole používáme `_exists_:nazev_pole`.  
✅ Opakované dotazy ukládáme jako saved searches a provozní přehledy jako widgety dashboardu.  
⚠️ Význam události vždy posuzujeme společně s kanálem, providerem a obsahem zprávy.

---

### Zdroje

* [Graylog: Search Your Log Data](https://go2docs.graylog.org/current/making_sense_of_your_log_data/search_your_log_data.html)
* [Graylog: Saved Searches](https://go2docs.graylog.org/current/making_sense_of_your_log_data/saved_searches.html)
* [Graylog: Dashboards](https://go2docs.graylog.org/current/interacting_with_your_log_data/dashboards.html)
