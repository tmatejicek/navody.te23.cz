---
title: "Základní nastavení Graylogu po instalaci"
date: 2026-04-15
tags: [Graylog, Logging, Monitoring]
layout: post
---

## Základní nastavení Graylogu po instalaci

Samotná instalace Graylogu ještě neznamená, že je prostředí připravené pro běžný provoz. V tomto článku si ukážeme, **co přesně po instalaci nastavit**, **kde to v aktuálním Graylogu najít** a **co zkontrolovat**, aby se později nemusely předělávat inputy, streamy, retence nebo dashboardy.

Tento článek navazuje na [Instalaci Graylog Open na Debianu 12]({% post_url 2026-04-15-Instalace-Graylog-Open-na-Debianu-12-dvouserverove-reseni %}).

---

### 1. Nastavení veřejné URL Graylogu v `server.conf`

První praktická věc po instalaci je správně nastavit veřejnou URL Graylogu. Používá se ve webovém rozhraní i v odkazech, které Graylog generuje například v e-mailových notifikacích.

Na `graylog01` upravíme soubor:

```text
/etc/graylog/server/server.conf
```

a nastavíme:

```ini
http_external_uri = https://logs.example.cz/
```

Pokud Graylog publikujeme přes reverzní proxy nebo certifikát s veřejným názvem, musí zde být právě tato výsledná adresa, nikoliv interní hostname serveru.

Pak restartujeme službu:

```bash
sudo systemctl restart graylog-server
```

Ověření:

1. otevřeme Graylog v prohlížeči přes cílovou URL
2. zkontrolujeme, že se přesměrování nebo odkazy nevracejí na interní jméno serveru

📌 Pokud `http_external_uri` nenastavíme správně, budou některé odkazy z Graylogu směřovat na špatnou adresu.

---

### 2. Nastavení retence a rotace v index setech

Graylog ukládá zprávy do **index setů**. Právě zde určujeme, jak budou data rotovat a jak dlouho zůstanou dostupná.

V prostředí, které máš na screenshotu, je praktická cesta:

```text
System / Indices
```

Tady:

1. otevřeme existující `Default index set`
2. nebo klikneme na `Create index set`
3. nastavíme hodnoty přímo na konkrétním index setu

Pro menší nebo střední on-premise prostředí v Graylog Open dává smysl zkontrolovat nebo nastavit:

* **Index Analyzer**: `standard`
* **Shards per Index**: `1`
* **Replicas**: `0`
* **Max. Number of Segments**: `1`
* **Field type refresh interval**: `5 minutes`

V části **Rotation & Retention** zkontrolujeme, jaká strategie je nastavená.

Prakticky nastavíme:

* **Rotation strategy**: `Index Time Size Optimizing`
* **Retention strategy**: `Delete`

Ve formuláři pak pracujeme hlavně s položkami:

* **Min. in storage**
* **Max. in storage**

Pokud používáme `Index Time Size Optimizing`, je praktičtější nenechávat obě hodnoty stejné. Mezi `Min. in storage` a `Max. in storage` necháme rezervu, aby Graylog mohl rotaci posunout podle velikosti shardů. Výchozí hodnoty `30 days / 40 days` jsou pro pochopení principu dobrý příklad.

Po úpravě změny uložíme.

📌 V prostředí s několika málo streamy je často praktičtější vytvořit si správně hned konkrétní index sety přímo v `System / Indices`.

#### 2.1 Co nastavit prakticky

Pokud organizace **nespadá pod ZoKB**:

* `Default index set`: `Min. in storage = 30 dní`, `Max. in storage = 40 dní`
* `server-logs`: `Min. in storage = 90 dní`, `Max. in storage = 100 dní`, pokud chceme delší retenci serverů

Pokud organizace spadá do **nižšího režimu** podle ZoKB:

* `Default index set`: `Min. in storage = 30 dní`, `Max. in storage = 40 dní`
* `server-logs`: `Min. in storage = 365 dní`, `Max. in storage = 395 dní`

Pokud organizace spadá do **vyššího režimu** podle ZoKB:

* `Default index set`: `Min. in storage = 30 dní`, `Max. in storage = 40 dní`
* `server-logs`: `Min. in storage = 548 dní`, `Max. in storage = 578 dní`

---

### 3. Vytvoření samostatného index setu pro serverové logy

Pokud mají logy koncových stanic i síťové syslogy stejnou retenci jako výchozí nastavení Graylogu, nemá smysl pro ně zakládat další index sety. V takovém případě necháme:

* `Default index set` pro běžné endpointy a síťové logy
* samostatný `server-logs` jen pro servery, AD a bezpečnostní logy s delší retencí

V aktuálním UI otevřeme:

```text
System / Indices
```

Klikneme na:

```text
Create index set
```

A v tomto modelu vytvoříme jen:

* `server-logs`

Praktický základ:

* **Title**: `Server Logs`
* **Index prefix**: `server-logs`
* **Analyzer**: `standard`
* **Index shards**: `1`
* **Index replicas**: `0`

V části **Rotation & Retention** pak nastavíme retenci podle typu dat.

Praktické mapování:

* logy koncových stanic -> `Default index set`
* síťové syslogy -> `Default index set`
* logy serverů včetně AD a bezpečnostních událostí -> index set `server-logs`

📌 Pokud organizace spadá do režimu vyšších povinností, je samostatný index set pro serverové a bezpečnostní logy prakticky nutnost.  
📌 Pokud mají dva typy logů stejnou retenci, samostatný stream stačí a další index set jen zbytečně přidává správu navíc.

---

### 4. Založení prvních inputů a routování do správných streamů

Po instalaci Graylogu ještě žádná data sama netečou. Další krok je vytvoření inputů.

V menší organizaci bývá praktický začátek:

* **Beats input** pro Winlogbeat
* **Syslog UDP** nebo **Syslog TCP** pro síťové prvky

Inputy otevřeme v:

```text
System / Inputs
```

#### 4.1 Beats input pro Winlogbeat

1. V `System / Inputs` v poli **Select input** vybereme `Beats`.
2. Klikneme na **Launch new input**.
3. Vyplníme minimálně:
   * **Title**: `Winlogbeat`
   * **Bind address**: `0.0.0.0`
   * **Port**: `5044`
4. Pokud nemáme důvod input vázat jen na jeden uzel, zvolíme **Global**.
5. Klikneme na **Launch input**.

Pak se otevře **Input Setup Wizard**. Ten je potřeba dokončit, jinak input ještě neběží.

Praktický postup v průvodci:

1. zvolíme **Skip Illuminate**
2. v části routování vybereme **Create Stream**
3. jako název streamu zadáme například `Windows Endpoints`
4. jako **Index Set** ponecháme `Default index set`
5. dokončíme průvodce přes **Start Input**

#### 4.2 Syslog input pro síťové prvky

Stejný princip použijeme pro MikroTik a UniFi:

1. v `System / Inputs` vybereme `Syslog UDP` nebo `Syslog TCP`
2. klikneme na **Launch new input**
3. nastavíme port podle návrhu prostředí
4. zvolíme **Global**
5. po spuštění dokončíme **Input Setup Wizard**

V routování nastavíme například:

* stream `Network Syslog`
* index set `Default index set`

Praktická pravidla:

* pokud není důvod k izolaci na konkrétní uzel, použijeme **Global input**
* pokud to zdroj podporuje, je spolehlivější **TCP** než **UDP**
* input bez dokončeného **Input Setup Wizard** ještě není v plném provozu
* stream a index set je lepší určit hned při vzniku inputu
* serverové a bezpečnostní logy je vhodné směrovat do samostatného streamu s delší retencí

#### 4.3 Jak oddělit koncové stanice a servery

Pokud chceme mít logy koncových stanic na `30 dní`, ale logy serverů na `12` nebo `18 měsíců`, nestačí jen jeden společný stream.

Praktický postup:

1. vytvoříme stream `Windows Endpoints` s index setem `Default index set`
2. vytvoříme druhý stream `Server Logs`
3. tomuto streamu přiřadíme index set `server-logs`
4. do streamu `Server Logs` přidáme pravidla například pro:
   * hostname serverů a doménových řadičů
   * `winlogbeat_winlog_channel:"Security"`
   * `winlogbeat_winlog_channel:"Microsoft-Windows-PowerShell/Operational"`
   * `winlogbeat_winlog_channel:"Microsoft-Windows-Sysmon/Operational"`
   * `winlogbeat_winlog_channel:"Directory Service"`

Stream vytvoříme v:

```text
Streams
```

Tím získáme:

* kratší retenci pro koncové stanice a síťové logy v `Default index set`
* delší retenci pro serverové a bezpečnostní logy
* jednodušší dashboardy, alerty a oprávnění

---

### 5. První uživatelé a přístupy v Graylog Open

Graylog používá role i sdílení entit. V Open edici je ale potřeba počítat s tím, že možnosti sdílení jsou jednodušší než v Enterprise.

V aktuálním UI otevřeme:

```text
System / Users and Teams
```

Na kartě **Users** klikneme na:

```text
Create user
```

Praktický základ:

1. vytvořit osobní administrátorský účet pro správce
2. vestavěný účet `admin` ponechat spíš jako nouzový účet
3. běžným technikům dát roli `Reader`
4. jen uživatelům, kteří mají spravovat Graylog, dát `Admin`

Praktická poznámka pro Open edici:

* dokumentace uvádí, že **Graylog Open** umí sdílení jen přes **Share with Everyone**
* pokud tedy chceme, aby běžní technici viděli společné dashboardy nebo saved searches, je v Open edici nejjednodušší tyto entity sdílet právě takto

Prakticky to znamená:

1. otevřeme například dashboard nebo saved search
2. klikneme na **Share**
3. zvolíme **Share with Everyone**
4. nastavíme úroveň přístupu, typicky `Viewer`

Pokud chceme zkontrolovat dostupné role, otevřeme:

```text
System / Roles
```

📌 Každý uživatel musí mít alespoň roli `Reader` nebo `Admin`.  
📌 Samotná role nestačí ke všemu. U entit, jako jsou dashboardy nebo saved searches, stále hraje roli sdílení.

---

### 6. SMTP pro budoucí e-mailové alerty

Pokud chceme v budoucnu používat e-mailové alerty, je rozumné SMTP nastavit hned na začátku.

Na `graylog01` upravíme:

```text
/etc/graylog/server/server.conf
```

a doplníme například:

```ini
transport_email_enabled = true
transport_email_hostname = smtp.example.cz
transport_email_port = 587
transport_email_use_auth = true
transport_email_auth_username = graylog@example.cz
transport_email_auth_password = <HESLO>
transport_email_use_tls = true
transport_email_from_email = graylog@example.cz
transport_email_web_interface_url = https://logs.example.cz/
```

Poté restartujeme Graylog server:

```bash
sudo systemctl restart graylog-server
```

Praktický test:

1. otevřeme `Alerts / Notifications`
2. klikneme na **Create notification**
3. jako typ vybereme **Email Notification**
4. použijeme **Execute Test Notification**

Tím si ověříme, že SMTP funguje ještě předtím, než začneme vytvářet alerty.

📌 `transport_email_web_interface_url` by měla odpovídat stejné veřejné adrese jako `http_external_uri`.

---

### 7. Kontrola po prvním ingestu

Jakmile do Graylogu začnou téct první data, uděláme krátkou provozní kontrolu.

#### 7.1 Ověření inputů

Otevřeme:

```text
System / Inputs
```

Zkontrolujeme:

* že inputy nejsou jen v setup módu
* že přibývají zprávy
* že nejsou vidět chyby při spuštění inputu

#### 7.2 Ověření streamů a index setů

Při vytváření inputů zkontrolujeme, že:

* logy koncových stanic tečou do streamu `Windows Endpoints` a používají `Default index set`
* logy serverů tečou do streamu `Server Logs` a používají `server-logs`
* síťové logy tečou do streamu `Network Syslog` a používají `Default index set`

#### 7.3 Ověření polí na reálné zprávě

V `Search` si otevřeme jednu reálnou Windows zprávu a zkontrolujeme pole:

* `winlogbeat_winlog_computer_name`
* `winlogbeat_winlog_channel`
* `winlogbeat_event_code`

Pokud tato pole nevidíme tak, jak očekáváme, je lepší problém vyřešit hned na začátku, než později opravovat dashboardy, saved searches nebo alerty.

#### 7.4 Ověření Data Node a journalu

Nakonec zkontrolujeme:

* že je Data Node připojený
* že journal neroste neobvykle rychle
* že se v prostředí neobjevují dlouhodobé backlogy
* že zdroje mají správně synchronizovaný čas

To je důležité hlavně v prvních dnech po nasazení. U režimu vyšších povinností je navíc nepřetržitá synchronizace jednotného času technických aktiv výslovný požadavek vyhlášky.

---

## Shrnutí

✅ Po instalaci je vhodné nejdřív nastavit `http_external_uri` a ověřit výslednou URL  
✅ Retenci a rotaci nastavujeme přímo přes `System / Indices`  
✅ `Default index set` může zůstat pro běžné endpointy a síťové logy  
✅ Pro ZoKB dává smysl mít delší retenci v index setu `server-logs`  
✅ Každý input je potřeba nejen spustit, ale i dokončit v `Input Setup Wizard`  
✅ V Graylog Open je praktické počítat s jednoduchým modelem rolí a sdílení  
✅ SMTP a první provozní kontrolu je lepší udělat hned na začátku
