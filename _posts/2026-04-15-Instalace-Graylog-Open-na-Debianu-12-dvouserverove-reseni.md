---
title: "Instalace Graylog Open na Debianu 12 (dvouserverové řešení)"
date: 2026-04-15
tags: [Graylog, Debian, MongoDB, "Data Node"]
layout: post
---

## Instalace Graylog Open na Debianu 12 (dvouserverové řešení)

V tomto článku si ukážeme instalaci **Graylog Open** podle oficiální dokumentace **Graylog 7.0 pro Debian 12**. Použijeme doporučené dvouserverové rozdělení rolí: na prvním serveru poběží **Graylog server a MongoDB**, na druhém **Graylog Data Node**.

Tento postup odpovídá základnímu produkčnímu nasazení typu **core cluster**, nikoliv vysoce dostupnému řešení. Je tedy vhodný tam, kde chceme oddělit vyhledávací vrstvu od aplikační vrstvy, ale nepotřebujeme plnou redundanci.

---

### 1. Příprava prostředí

Než začneme, připravíme si dva servery s **Debianem 12**:

* `graylog01` - Graylog server + MongoDB
* `graylog02` - Graylog Data Node

Je vhodné mít správně nastavené:

* DNS nebo alespoň záznamy v `/etc/hosts`
* synchronizaci času pomocí NTP
* firewall mezi servery

V minimální variantě budeme potřebovat povolit tyto porty:

* z administrátorského počítače na `graylog01`: `TCP/9000`
* z `graylog02` na `graylog01`: `TCP/27017`
* z `graylog01` na `graylog02`: `TCP/8999`
* z `graylog01` na `graylog02`: `TCP/9200`

📌 V této architektuře **neinstalujeme OpenSearch ručně**. O vyhledávací vrstvu se stará **Graylog Data Node**.

#### 1.1 Vygenerování sdíleného `password_secret`

Na jednom ze serverů si vygenerujeme tajný řetězec, který použijeme **ve stejné podobě** v `server.conf` i `datanode.conf`:

```bash
openssl rand -hex 32
```

Výstup si uložíme, například:

```text
5dbd6f7a6f8d5f9cc1f1d0f4cb34b04fa5cbb6fcb2a80e4228f84c7f0c40d41d
```

#### 1.2 Vygenerování hashovaného hesla správce

Na `graylog01` si připravíme hash hesla pro účet `admin`:

```bash
echo -n 'SilneAdminHeslo' | sha256sum | cut -d' ' -f1
```

Tento hash použijeme později jako hodnotu `root_password_sha2`.

---

### 2. Instalace MongoDB na `graylog01`

Graylog 7.0 v tomto scénáři používá **MongoDB** pro metadata a konfiguraci. Na serveru `graylog01` nejprve přidáme repozitář MongoDB 8.0:

```bash
wget -qO - https://www.mongodb.org/static/pgp/server-8.0.asc | sudo tee /etc/apt/trusted.gpg.d/mongodb-server-8.0.asc
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/debian bookworm/mongodb-org/8.0 main" | sudo tee /etc/apt/sources.list.d/mongodb-org-8.0.list
sudo apt-get update
sudo apt-get install -y mongodb-org
```

Poté upravíme vazbu služby tak, aby byla dostupná i z druhého serveru:

```yaml
# /etc/mongod.conf
net:
  port: 27017
  bindIp: 127.0.0.1,192.168.10.11
```

V ukázce je použita IP adresa `192.168.10.11`; nahradíme ji skutečnou adresou serveru `graylog01`.

Službu spustíme a povolíme po startu systému:

```bash
sudo systemctl daemon-reload
sudo systemctl enable mongod.service
sudo systemctl restart mongod.service
sudo systemctl status mongod.service
```

Nakonec je vhodné balíčky MongoDB podržet, aby se neaktualizovaly neplánovaně:

```bash
sudo apt-mark hold mongodb-org
```

---

### 3. Instalace Graylog Data Node na `graylog02`

Graylog Data Node vyžaduje správně nastavený parametr `vm.max_map_count`:

```bash
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

Pak přidáme Graylog repozitář a nainstalujeme balíček:

```bash
wget https://packages.graylog2.org/repo/packages/graylog-7.0-repository_latest.deb
sudo dpkg -i graylog-7.0-repository_latest.deb
sudo apt-get update
sudo apt-get install graylog-datanode
```

#### 3.1 Konfigurace `datanode.conf`

Do souboru `/etc/graylog/datanode/datanode.conf` doplníme minimálně tyto hodnoty:

```ini
password_secret = <STEJNA_HODNOTA_Z_OPENSSL_RAND>
mongodb_uri = mongodb://graylog01:27017/graylog
opensearch_heap = 4g
```

📌 `mongodb_uri` upravíme podle skutečného hostname nebo IP serveru s MongoDB.  
📌 `password_secret` musí být stejný jako na Graylog serveru.  
📌 `opensearch_heap` nastavíme přibližně na polovinu RAM serveru, maximálně však podle sizingu výrobce.

Službu následně zapneme:

```bash
sudo systemctl daemon-reload
sudo systemctl enable graylog-datanode.service
sudo systemctl restart graylog-datanode.service
sudo systemctl status graylog-datanode.service
```

---

### 4. Instalace Graylog serveru na `graylog01`

Na serveru `graylog01` přidáme stejný Graylog repozitář i pro aplikační část:

```bash
wget https://packages.graylog2.org/repo/packages/graylog-7.0-repository_latest.deb
sudo dpkg -i graylog-7.0-repository_latest.deb
sudo apt-get update
sudo apt-get install graylog-server
```

#### 4.1 Nastavení Java heapu

V souboru `/etc/default/graylog-server` upravíme velikost heapu podle dostupné RAM. Příklad pro server s menší zátěží:

```ini
GRAYLOG_SERVER_JAVA_OPTS="-Xms1g -Xmx1g -server -XX:+UseG1GC -XX:-OmitStackTraceInFastThrow"
```

📌 Obvykle volíme přibližně **50 % fyzické RAM**, maximálně však podle doporučení Graylogu.

#### 4.2 Konfigurace `server.conf`

V souboru `/etc/graylog/server/server.conf` nastavíme hlavní parametry:

```ini
password_secret = <STEJNA_HODNOTA_Z_OPENSSL_RAND>
root_password_sha2 = <HASH_ADMIN_HESLA_ZE_SHA256SUM>
http_bind_address = 0.0.0.0:9000
message_journal_max_age = 72h
message_journal_max_size = 90gb
```

Význam nejdůležitějších voleb:

* `password_secret` je sdílené tajemství používané Graylogem
* `root_password_sha2` je SHA-256 hash hesla účtu `admin`
* `http_bind_address` zpřístupní webové rozhraní na portu `9000`
* `message_journal_*` určují velikost a dobu držení lokální fronty zpráv

Pokud MongoDB běží lokálně a výchozí `mongodb_uri` vám vyhovuje, není nutné ho měnit. V opačném případě ho nastavíme explicitně.

Graylog server spustíme:

```bash
sudo systemctl daemon-reload
sudo systemctl enable graylog-server.service
sudo systemctl restart graylog-server.service
sudo systemctl status graylog-server.service
```

---

### 5. Dokončení instalace přes webové rozhraní

Po startu Graylog serveru sledujeme log:

```bash
sudo tail -f /var/log/graylog-server/server.log
```

V logu se po startu objeví jednorázové přihlašovací údaje pro úvodní **preflight** konfiguraci. Poté otevřeme v prohlížeči:

```text
https://graylog01:9000/
```

V preflight kroku provedeme:

* přihlášení jednorázovými údaji z logu
* potvrzení nebo vytvoření certifikátů pro Data Node
* přiřazení Data Node k instalaci
* dokončení inicializace clusteru

Po dokončení preflightu se do Graylogu přihlašujeme standardně jako:

* uživatel: `admin`
* heslo: původní heslo, ze kterého jsme vytvořili `root_password_sha2`

⚠ `password_secret` **není** heslo pro přihlášení do webového rozhraní.

---

### 6. Ověření funkčnosti

Po dokončení instalace ověříme, že běží všechny tři klíčové služby:

```bash
sudo systemctl status mongod
sudo systemctl status graylog-datanode
sudo systemctl status graylog-server
```

Zároveň zkontrolujeme:

* zda se Graylog web otevře na `https://graylog01:9000`
* zda je v rozhraní vidět připojený Data Node
* zda nevznikají chyby v `/var/log/graylog-server/server.log`
* zda je možné vytvořit první input a index set

Pokud se Data Node nepřipojí, nejčastější příčinou bývá:

* chyba v `mongodb_uri`
* blokovaný port `27017`, `8999` nebo `9200`
* rozdílná hodnota `password_secret`
* chybějící nebo špatně nastavený `vm.max_map_count`

---

## Shrnutí

✅ Dvouserverová instalace Graylog Open v základní podobě používá `graylog01` pro Graylog + MongoDB a `graylog02` pro Data Node  
✅ MongoDB musí být z Data Node dostupné přes `TCP/27017`  
✅ `password_secret` musí být stejný v `server.conf` i `datanode.conf`  
✅ Data Node vyžaduje správně nastavený `vm.max_map_count`  
✅ Dokončení instalace probíhá přes webový preflight na `https://graylog01:9000`
