---
title: "Základní nastavení Graylogu po instalaci"
date: 2026-04-15
tags: [Graylog, Logging, Monitoring]
layout: post
---

## Základní nastavení Graylogu po instalaci

Postup navazuje na [Instalaci Graylog Open na Debianu 12]({% post_url 2026-04-15-Instalace-Graylog-Open-na-Debianu-12-dvouserverove-reseni %}) a používá názvy položek z Graylogu 7.1.

📌 Nastavení níže provádíme před připojením většího počtu produkčních zdrojů logů.

---

### 1. Veřejná URL a HTTPS

Na `graylog01` upravíme:

```text
/etc/graylog/server/server.conf
```

```ini
http_external_uri = https://logs.example.cz/
```

📌 Hodnota musí být URL, kterou používají uživatelé. Pokud je před Graylogem reverzní proxy, uvedeme její výslednou HTTPS adresu. Potom:

```bash
sudo systemctl restart graylog-server
sudo systemctl --no-pager --full status graylog-server
```

V prohlížeči ověříme platný certifikát, přihlášení a to, že odkazy generované Graylogem neobsahují interní hostname ani HTTP.

---

### 2. Retence v index setech

Otevřeme:

```text
System / Indices
```

U každého index setu použijeme **Edit** a v části **Rotation & Retention** ponecháme doporučenou volbu **Data Tiering**. V Graylogu Open ponecháme **Warm Tier** vypnutý; warm tier vyžaduje Enterprise licenci. Volba **Legacy (Deprecated)** a strategie **Index Time Size Optimizing** nejsou pro nové nastavení doporučené.

Nejdůležitější položky jsou:

* **Min. in storage**: nejkratší doba, po kterou mají data zůstat uložená
* **Max. in storage**: horní hranice, po jejímž dosažení Graylog data odstraní

📌 **Hodnoty Min. a Max. in storage nenastavujeme stejně.** Rozdíl dává Graylogu prostor pro práci s celými indexy; výchozích `30 days / 40 days` je vhodný základ pro běžná data bez delšího retenčního požadavku.

#### Praktické hodnoty

| Použití | Min. in storage | Max. in storage |
|---|---:|---:|
| `Default index set` pro běžné endpointy a síťové logy | 30 dní | 40 dní |
| `server-logs` mimo regulovaný požadavek | 90 dní | 100 dní |
| `server-logs` jako interně schválený základ pro nižší režim | 365 dní | 395 dní |
| vybrané události ve vyšším režimu | 550 dní | 580 dní |

📌 U **nižšího režimu** vyhláška č. 410/2025 Sb. neurčuje pevný počet dnů. Dobu uchování stanovíme podle bezpečnostních potřeb, hodnocení rizik, smluvních požadavků a interní směrnice; `365/395` je pouze praktický výchozí návrh.

⚠️ U **vyššího režimu** vyhláška č. 409/2025 Sb. vyžaduje uchovat určené bezpečnostní a relevantní provozní události nejméně 18 měsíců. Hodnoty `550/580` poskytují rezervu vůči rozdílné délce kalendářních měsíců. Nestačí ale pouze nastavit retenci: organizace musí určit sledované události a zajistit, aby všechny skončily v odpovídajícím index setu. Může jít i o vybrané události z koncových stanic nebo síťových prvků.

Před prodloužením retence spočítáme kapacitu z reálného denního ingestu:

```text
potřebná kapacita ≈ průměrný denní přírůstek × počet dnů × 1,2
```

⚠️ Rezerva `20 %` je minimum pro růst a provozní špičky. Retenci nezvyšujeme, pokud by tím mohl Data Node zaplnit disk.

Vzorec předpokládá `0` replik a nezahrnuje snapshoty ani zálohy. Po přidání replik nebo lokálního snapshot repository jejich kapacitu přičteme zvlášť.

---

### 3. Index set `server-logs`

Koncové stanice a síťové syslogy se stejnou retencí ponecháme v `Default index set`. Samostatný index set vytvoříme jen pro data s odlišnou retencí:

```text
System / Indices / Create index set
```

Nastavíme:

* **Title**: `Server Logs`
* **Description**: `Logy serverů, AD a vybrané bezpečnostní události`
* **Index prefix**: `server-logs`
* **Analyzer**: `standard`
* **Index shards**: `1`
* **Index replicas**: `0`
* **Rotation & Retention**: `Data Tiering`
* **Warm Tier**: vypnuto
* **Min./Max. in storage**: podle předchozí tabulky

📌 Hodnoty `1 shard / 0 replicas` odpovídají instalaci s jediným Data Node. Replika na stejném Data Node nezvyšuje odolnost a zbytečně spotřebuje místo. Po přidání druhého Data Node lze nastavit jednu repliku, ale až po ověření kapacity.

---

### 4. Inputy

Inputy vytváříme v:

```text
System / Inputs
```

Pro menší prostředí obvykle stačí:

* `Beats` na `TCP/5044` pro Winlogbeat
* `Syslog TCP` pro zařízení, která TCP podporují
* `Syslog UDP` jen pro zdroje, které TCP neumějí

📌 Po **Launch input** dokončíme **Input Setup Wizard**. V Graylogu 7.1 input v setup módu neběží. V edici Open obvykle zvolíme **Skip Illuminate** a jako existující cílový stream vybereme `All messages`, který používá `Default index set`.

Input označíme jako **Global** jen tehdy, když má poslouchat na každém Graylog serveru. V popisovaném dvouserverovém řešení běží Graylog Server pouze na `graylog01`, takže může být input svázán s tímto uzlem.

Porty inputů na firewallu povolíme jen ze zdrojových sítí. Beats input a Winlogbeat je vhodné zabezpečit TLS podle [samostatného článku]({% post_url 2026-04-15-Nastaveni-Winlogbeat-pro-odesilani-Windows-event-logu-do-Graylogu %}).

---

### 5. Stream `Server Logs` bez duplicit

⚠️ Zpráva uložená současně přes streamy s různými index sety může zabírat místo v obou index setech. Serverové zprávy proto nesmějí zůstat zároveň v `Default index set`.

Otevřeme:

```text
Streams / Create stream
```

Nastavíme:

* **Title**: `Server Logs`
* **Index Set**: `Server Logs`
* **Remove matches from 'Default Stream'**: zapnout

Potom přidáme pravidla podle **skutečných hostname serverů**. Například pro pole `winlogbeat_winlog_computer_name` vytvoříme jednu podmínku pro každý server a nastavíme, že zpráva musí splnit alespoň jedno pravidlo. Pokud máme spolehlivou jmennou konvenci, lze použít regulární výraz, například:

```text
^(DC|SRV|HV|DHCP)[0-9]+(\..+)?$
```

Před použitím regulárního výrazu jej ověříme nad reálnými hodnotami pole. Doménový řadič může například posílat FQDN místo krátkého jména.

Stream následně spustíme. V `Search` omezíme hledání na `Server Logs` a ověříme nové zprávy. Změna streamu nepřesune starší zprávy; pravidla se použijí až na nově přijatá data.

❌ Serverové logy **nerozpoznáváme pouze podle kanálu** `Security`, `System`, `Application`, PowerShell nebo Sysmon. Stejné kanály existují i na koncových stanicích a takové pravidlo by do dlouhé retence poslalo i jejich zprávy.

---

### 6. Uživatelé a oprávnění

Otevřeme:

```text
System / Users and Teams / Users
```

1. Vytvoříme osobní účet každému správci.
2. Vestavěný účet `admin` ponecháme jako nouzový a jeho heslo uložíme bezpečně.
3. Běžným technikům přidělíme pouze potřebné role, typicky `Reader`.
4. Přístup ke streamům, dashboardům a uloženým hledáním udělíme přes jejich nabídku **Share**.
5. U sdílených entit nastavíme nejnižší potřebnou úroveň, typicky `Viewer`.

📌 Samotná role `Reader` automaticky neznamená přístup ke každému streamu nebo dashboardu. Dostupné možnosti sdílení se mohou lišit podle licence a zapnutých funkcí, proto se řídíme volbami zobrazenými v konkrétní instalaci.

---

### 7. SMTP

V `/etc/graylog/server/server.conf` nastavíme například SMTP se STARTTLS:

```ini
transport_email_enabled = true
transport_email_hostname = smtp.example.cz
transport_email_port = 587
transport_email_use_auth = true
transport_email_auth_username = graylog@example.cz
transport_email_auth_password = <HESLO>
transport_email_use_tls = true
transport_email_use_ssl = false
transport_email_from_email = graylog@example.cz
transport_email_web_interface_url = https://logs.example.cz/
```

⚠️ Soubor obsahuje heslo, proto ověříme oprávnění:

```bash
sudo chown root:graylog /etc/graylog/server/server.conf
sudo chmod 640 /etc/graylog/server/server.conf
sudo systemctl restart graylog-server
```

Test provedeme přes:

```text
Alerts / Notifications / Create notification / Email Notification
```

V editoru použijeme **Execute Test Notification**. Pokud SMTP relay podporuje omezení podle IP adresy, je bezpečnější vytvořit samostatný účet nebo relay pravidlo pouze pro `graylog01`.

---

### 8. Provozní kontrola

Po prvním ingestu zkontrolujeme:

1. `System / Inputs`: input je spuštěný, není v setup módu a nemá chyby v **Input Diagnosis**.
2. `Streams`: nové serverové zprávy jsou v `Server Logs` a nejsou v `All messages`/defaultním index setu.
3. `System / Indices`: indexy jsou zdravé a roste očekávaný index set.
4. `Search`: na reálné zprávě existují očekávaná pole a timestamp odpovídá času události.
5. Disk Data Node: je dostatek volného místa a indexová data rostou podle očekávání.
6. Journal na Graylog Serveru: fronta se za běžného provozu nezvětšuje a disk má dostatečnou rezervu.
7. Čas: Graylog, Data Node, MongoDB i zdroje používají synchronizovaný čas.

U Winlogbeatu při výchozím prefixování ověříme hlavně:

```text
winlogbeat_winlog_computer_name
winlogbeat_winlog_channel
winlogbeat_event_code
```

---

### 9. Zálohy a obnova

⚠️ Záloha pouze celé VM za běhu není dostatečný plán obnovy. Zálohujeme odděleně:

* MongoDB, která obsahuje konfiguraci Graylogu, uživatele, streamy a dashboardy
* konfigurace a certifikáty z `/etc/graylog/` a reverzní proxy
* hodnoty `password_secret` a `root_password_sha2`
* indexová data pomocí podporovaných snapshotů Data Node/OpenSearch, pokud je vyžadována jejich obnova

❌ Nekopírujeme pouze živý adresář Data Node jako konzistentní zálohu.

✅ Nejméně jednou za rok provedeme test obnovy do odděleného prostředí a zapíšeme skutečný čas obnovy.

---

### Shrnutí

✅ Nastavíme správnou veřejnou HTTPS adresu a ověříme generované odkazy.  
✅ Běžná data ponecháme v `Default index set`; samostatný index set vytváříme jen pro odlišnou retenci.  
✅ Serverový stream odebere odpovídající zprávy z výchozího streamu, aby nevznikaly duplicity.  
✅ Inputy dokončíme v **Input Setup Wizard** a porty povolíme jen ze zdrojových sítí.  
✅ Nastavíme osobní účty, SMTP, provozní kontrolu a otestovaný plán obnovy.

---

### Zdroje

* [Graylog: Index Model](https://go2docs.graylog.org/current/setting_up_graylog/index_model.html)
* [Graylog: Data Tiering](https://go2docs.graylog.org/current/setting_up_graylog/data_tiering.htm)
* [Graylog: Set Up an Input](https://go2docs.graylog.org/current/getting_in_log_data/setup_an_input.htm)
* [Graylog: Streams](https://go2docs.graylog.org/current/making_sense_of_your_log_data/streams.html)
* [Graylog: Backup and Restore Best Practices](https://go2docs.graylog.org/current/setting_up_graylog/backup_considerations.htm)
* [Vyhláška č. 409/2025 Sb. pro vyšší režim](https://www.zakonyprolidi.cz/cs/2025-409)
* [Vyhláška č. 410/2025 Sb. pro nižší režim](https://www.zakonyprolidi.cz/cs/2025-410)
