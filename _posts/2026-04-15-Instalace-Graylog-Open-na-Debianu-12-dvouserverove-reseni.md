---
title: "Instalace Graylog Open na Debianu 12 (dvouserverové řešení)"
date: 2026-04-15
tags: [Graylog, Debian, MongoDB, Data Node]
layout: post
---

## Instalace Graylog Open na Debianu 12 (dvouserverové řešení)

Tento návod instaluje **Graylog Open 7.1** na Debian 12 v architektuře **Core cluster**:

- `graylog01` (`192.168.10.11`): Graylog server a MongoDB,
- `graylog02` (`192.168.10.12`): Graylog Data Node s vestavěným OpenSearch.

⚠️ **Jde o dvouserverové rozdělení rolí, nikoli o vysokou dostupnost.** Výpadek kteréhokoli serveru omezí nebo zastaví službu.

📌 V této architektuře **neinstalujeme OpenSearch ručně**. Jeho životní cyklus spravuje Graylog Data Node.

---

### 1. Příprava serverů

Na obou serverech nastavíme stabilní FQDN/DNS, časovou synchronizaci a UTC:

```bash
sudo timedatectl set-timezone UTC
timedatectl status
getent hosts graylog01
getent hosts graylog02
```

Pokud nepoužíváme DNS, doplníme na oba servery `/etc/hosts`:

```text
192.168.10.11 graylog01
192.168.10.12 graylog02
```

#### 1.1 CPU, RAM a disky

Pro základní produkční Core deployment použijeme jako výchozí sizing:

| Server | vCPU | RAM | Doporučené disky |
|---|---:|---:|---|
| `graylog01` | 8 | 16 GB | OS 40 GB, MongoDB 20 až 50 GB, journal podle ingestu |
| `graylog02` | 8 | 24 GB | OS 40 GB, samostatný datový disk podle ingestu a retence |

Datové disky mají být SSD/NVMe; pro journal a Data Node preferujeme XFS. Datový disk Data Node připojíme pod `/var/lib/datanode`, journal pod `/var/lib/graylog-server/journal` a nastavíme vlastníka podle účtu služby po instalaci balíčků.

Velikost navrhneme z reálné denní ingestace:

- journal: 3 až 5násobek denního ingestu, nejméně kapacita pro doporučených 72 hodin,
- Data Node: `denní ingest × počet dní retence × 1,2`,
- na každém souborovém systému ponecháme alespoň 20 % volného místa.

Při 20 GB denně a 30 dnech retence vychází:

- journal přibližně 60 až 100 GB,
- Data Node přibližně 720 GB.

📌 Jde o počáteční odhad. Komprese, počet polí, shardů a způsob vyhledávání mají velký vliv, proto po pilotu porovnáme skutečný přírůstek dat. Výpočet musí vycházet z nejdelší požadované retence, ne jen z výchozích 30 dnů.

#### 1.2 Firewall

Nepovolujeme interní porty z internetu ani z celé uživatelské sítě.

| Zdroj | Cíl | Port | Účel |
|---|---|---:|---|
| administrační síť | `graylog01` | TCP/9000 | dočasné HTTP pro preflight; po nasazení TLS nahradit HTTPS |
| `graylog01` | `graylog02` | TCP/8999 | Data Node API a správa |
| `graylog01` | `graylog02` | TCP/9200 | vyhledávání a indexace |
| `graylog02` | `graylog01` | TCP/27017 | MongoDB metadata |
| zdroje logů | `graylog01` | podle inputu | například TCP/5044 pro Beats |

📌 Port `9300/TCP` je transport mezi Data Nodes a v této jednouzlové Data Node variantě jej přes síť nepovolujeme. Pravidla realizujeme na síťovém i hostitelském firewallu podle standardu organizace.

#### 1.3 Tajné hodnoty

Na bezpečném administračním systému vygenerujeme `password_secret` o délce alespoň 64 znaků:

```bash
openssl rand -hex 32
```

Stejnou hodnotu později vložíme do `datanode.conf` i `server.conf`. Uložíme ji do správce hesel; po instalaci ji neměníme, protože chrání šifrované hodnoty uložené v MongoDB.

Na `graylog01` vygenerujeme hash hesla kořenového účtu Graylogu bez zapsání hesla do historie shellu:

```bash
echo -n "Enter Graylog admin password: "
read -rs GRAYLOG_PASSWORD
echo
printf '%s' "$GRAYLOG_PASSWORD" | sha256sum | cut -d' ' -f1
unset GRAYLOG_PASSWORD
```

✅ Výstup je `root_password_sha2`; původní heslo bezpečně uložíme.

---

### 2. MongoDB na `graylog01`

#### 2.1 Instalace MongoDB 8.0

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg

curl -fsSL https://www.mongodb.org/static/pgp/server-8.0.asc | \
  sudo gpg --dearmor -o /usr/share/keyrings/mongodb-server-8.0.gpg

echo "deb [ signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/debian bookworm/mongodb-org/8.0 main" | \
  sudo tee /etc/apt/sources.list.d/mongodb-org-8.0.list

sudo apt-get update
sudo apt-get install -y mongodb-org
sudo apt-mark hold mongodb-org
sudo systemctl enable --now mongod.service
sudo systemctl status mongod.service
```

⚠️ Kompatibilitu nové hlavní verze MongoDB vždy ověříme v matici Graylogu. Balíček je záměrně podržený proti neřízenému upgradu. Při plánované bezpečnostní aktualizaci použijeme `apt-mark unhold mongodb-org`, provedeme podporovaný upgrade v servisním okně a potom balíček znovu podržíme.

#### 2.2 Účty a autentizace MongoDB

Výchozí instalace nemá zapnutou autentizaci. Než MongoDB zpřístupníme Data Node, vytvoříme administrační účet a účet `graylog`. Hesla zadáváme interaktivně a uložíme do správce hesel:

```bash
mongosh
```

V konzoli `mongosh` provedeme:

```javascript
use admin
db.createUser({
  user: "mongoAdmin",
  pwd: passwordPrompt(),
  roles: [{ role: "root", db: "admin" }]
})

use graylog
db.createUser({
  user: "graylog",
  pwd: passwordPrompt(),
  roles: [
    { role: "readWrite", db: "graylog" },
    { role: "dbAdmin", db: "graylog" },
    { role: "clusterMonitor", db: "admin" }
  ]
})

exit
```

V `/etc/mongod.conf` nastavíme pouze loopback a interní IP `graylog01` a zapneme autorizaci:

```yaml
net:
  port: 27017
  bindIp: 127.0.0.1,192.168.10.11

security:
  authorization: enabled
```

```bash
sudo systemctl restart mongod.service
sudo systemctl status mongod.service

mongosh --host 127.0.0.1 --username graylog --password \
  --authenticationDatabase graylog
```

Heslo v MongoDB URI musí být URL-encoded. Pokud používáme náhodné heslo pouze z písmen, číslic a znaků `-._~`, další kódování není nutné. URI bude mít tvar:

```text
mongodb://graylog:<URL_ENCODED_PASSWORD>@graylog01:27017/graylog?authSource=graylog
```

⚠️ MongoDB obsahuje uživatele, role, dashboardy a konfiguraci Graylogu. Port `27017` proto povolíme pouze z `graylog02`; na `graylog01` se Graylog připojí přes `127.0.0.1`.

---

### 3. Data Node na `graylog02`

#### 3.1 Repozitář a balíček

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates wget
wget https://packages.graylog2.org/repo/packages/graylog-7.1-repository_latest.deb
sudo dpkg -i graylog-7.1-repository_latest.deb
sudo apt-get update
sudo apt-get install -y graylog-datanode
```

`vm.max_map_count` nastavíme v samostatném sysctl souboru:

```bash
echo 'vm.max_map_count=262144' | \
  sudo tee /etc/sysctl.d/99-graylog-datanode.conf
sudo sysctl --system
cat /proc/sys/vm/max_map_count
```

✅ Výsledek musí být nejméně `262144`.

Pokud je `/var/lib/datanode` samostatný připojený disk, nastavíme vlastníka po instalaci balíčku:

```bash
sudo chown -R graylog-datanode:graylog-datanode /var/lib/datanode
```

#### 3.2 `datanode.conf`

V `/etc/graylog/datanode/datanode.conf` nastavíme:

```ini
password_secret = <STEJNA_HODNOTA_NA_OBOU_SERVERECH>
mongodb_uri = mongodb://graylog:<URL_ENCODED_PASSWORD>@graylog01:27017/graylog?authSource=graylog
bind_address = 192.168.10.12
opensearch_data_location = /var/lib/datanode/data
opensearch_heap = 12g
```

Pro VM s 24 GB RAM nastavujeme OpenSearch heap na polovinu RAM, tedy `12g`. Maximum je 31 GB.

Data Node má ještě vlastní malý JVM proces. V `/etc/graylog/datanode/jvm.options` najdeme existující `-Xms` a `-Xmx` a nastavíme je shodně; nepřidáváme druhou dvojici:

```text
-Xms1g
-Xmx1g
```

Službu spustíme:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now graylog-datanode.service
sudo systemctl status graylog-datanode.service
sudo journalctl -u graylog-datanode -n 100 --no-pager
```

📌 Před preflightem běží Data Node API na `8999`; vestavěný OpenSearch se plně zprovozní až při provisioningu certifikátů.

---

### 4. Graylog server na `graylog01`

#### 4.1 Repozitář a balíček

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates wget
wget https://packages.graylog2.org/repo/packages/graylog-7.1-repository_latest.deb
sudo dpkg -i graylog-7.1-repository_latest.deb
sudo apt-get update
sudo apt-get install -y graylog-server
```

Pokud je journal samostatný disk, po instalaci nastavíme vlastníka:

```bash
sudo chown -R graylog:graylog /var/lib/graylog-server/journal
```

#### 4.2 Heap Graylog serveru

V `/etc/default/graylog-server` upravíme pouze hodnoty `-Xms` a `-Xmx`. Pro VM se 16 GB RAM použijeme 8 GB:

```ini
GRAYLOG_SERVER_JAVA_OPTS="-Xms8g -Xmx8g -server -XX:+UseG1GC -XX:-OmitStackTraceInFastThrow"
```

Minimum a maximum mají být shodné. Graylog serveru nepřidělujeme více než polovinu RAM ani více než 16 GB.

#### 4.3 `server.conf`

V `/etc/graylog/server/server.conf` nastavíme:

```ini
password_secret = <STEJNA_HODNOTA_NA_OBOU_SERVERECH>
root_password_sha2 = <SHA256_HASH_ADMIN_HESLA>
mongodb_uri = mongodb://graylog:<URL_ENCODED_PASSWORD>@127.0.0.1:27017/graylog?authSource=graylog

http_bind_address = 0.0.0.0:9000

message_journal_max_age = 72h
message_journal_max_size = 80gb
```

📌 `80gb` odpovídá ukázkové ingestaci 20 GB denně s rezervou. Pro jiné prostředí hodnotu přepočítáme. Journal zahazuje nejstarší nezpracované zprávy ve chvíli, kdy jako první dosáhne maximálního stáří nebo velikosti, proto musí vyhovovat obě hodnoty.

⚠️ `http_bind_address` zapíná obyčejné **HTTP**, nikoli HTTPS. Port `9000` zatím povolíme pouze z administrační sítě a použijeme jej pro preflight.

Službu spustíme:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now graylog-server.service
sudo systemctl status graylog-server.service
sudo journalctl -u graylog-server -n 200 --no-pager
```

---

### 5. Preflight

V logu Graylog serveru najdeme dočasné preflight přihlašovací údaje. Z administrační sítě otevřeme:

```text
http://graylog01:9000/
```

Přihlásíme se jednorázovými údaji z logu, nikoli heslem odpovídajícím `root_password_sha2`. V průvodci:

1. ověříme nalezený Data Node,
2. vytvoříme interní CA Graylogu nebo použijeme vlastní podporovanou CA,
3. provisionujeme Data Node certifikátem,
4. počkáme, až bude Data Node dostupný a cluster zdravý,
5. dokončíme preflight.

📌 Poté se přihlašujeme jako `admin` původním heslem, z něhož jsme vytvořili `root_password_sha2`. `password_secret` není přihlašovací heslo.

---

### 6. TLS před produkčním provozem

Přístup přes `http://graylog01:9000` používáme jen pro počáteční konfiguraci v izolované administrační síti. Před předáním uživatelům zapneme HTTPS jedním z podporovaných způsobů:

- TLS přímo v Graylogu pomocí `http_enable_tls`, certifikačního řetězce a PKCS#8 privátního klíče,
- reverzní proxy s TLS a správně nastaveným `http_external_uri`.

Po nasazení ověříme, že webové přihlášení, REST API a případné Sidecary používají `https://`, a nešifrovaný port `9000` omezíme jen na proxy nebo administrační síť. Také jednotlivé inputy, například Beats, zabezpečíme TLS.

⚠️ Ukázková MongoDB konfigurace výše používá autentizaci a síťovou izolaci, ale spojení `graylog02 -> graylog01:27017` ještě nešifruje. Před produkčním provozem zapneme TLS také v MongoDB, certifikační autoritu přidáme do Java truststore Graylog Serveru i Data Node a v obou `mongodb_uri` vynutíme TLS podle dokumentace MongoDB. Všechny interní porty ponecháme dostupné pouze mezi konkrétními uzly.

---

### 7. Ověření

Na `graylog01`:

```bash
systemctl is-active mongod graylog-server
ss -lntp | grep -E ':(27017|9000)\b'
sudo journalctl -u graylog-server -n 100 --no-pager
```

Na `graylog02`:

```bash
systemctl is-active graylog-datanode
cat /proc/sys/vm/max_map_count
ss -lntp | grep -E ':(8999|9200)\b'
sudo journalctl -u graylog-datanode -n 100 --no-pager
```

V Graylogu otevřeme **System > Cluster Configuration**, v části Data Nodes ověříme zdravý Data Node a platnost jeho certifikátu. Dále zkontrolujeme volné místo:

```bash
df -h /var/lib/graylog-server/journal
df -h /var/lib/datanode
```

Před příjmem produkčních logů ještě nastavíme retenci, streamy, uživatele, zálohy a monitoring podle navazujícího článku [Základní nastavení Graylogu po instalaci]({% post_url 2026-04-15-Zakladni-nastaveni-Graylogu-po-instalaci %}).

---

### Shrnutí

✅ `graylog01` provozuje Graylog server a MongoDB, `graylog02` Graylog Data Node.  
✅ Velikost journalu a datového disku počítáme z denní ingestace a požadované retence.  
✅ `password_secret` je na obou serverech stejný a ukládáme jej do správce hesel.  
✅ MongoDB používá autentizaci a port `27017` je dostupný jen mezi potřebnými uzly.  
✅ Před produkčním provozem zapneme TLS pro web, inputy i spojení k MongoDB.  
⚠️ Toto dvouserverové řešení neposkytuje vysokou dostupnost.

### Zdroje

- [Graylog 7.1: Debian installation](https://go2docs.graylog.org/current/downloading_and_installing_graylog/debian_installation.htm)
- [Initial configuration settings](https://go2docs.graylog.org/current/setting_up_graylog/initial_configuration_settings.html)
- [Network connectivity and firewall requirements](https://go2docs.graylog.org/current/setting_up_graylog/network_connectivity_and_firewall_requirements.htm)
- [Secure your Graylog environment](https://go2docs.graylog.org/current/planning_your_deployment/secure_your_graylog_environment.htm)
- [MongoDB users and roles required by Graylog](https://go2docs.graylog.org/current/setting_up_graylog/cluster_configuration.htm)
- [Graylog compatibility matrix](https://go2docs.graylog.org/current/downloading_and_installing_graylog/compatibility_matrix.htm)
- [MongoDB: Configure `mongod` with TLS](https://www.mongodb.com/docs/manual/tutorial/configure-ssl/)
