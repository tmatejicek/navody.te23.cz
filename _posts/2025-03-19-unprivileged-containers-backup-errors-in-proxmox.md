---
title: "Jak vyřešit problém se zálohováním neprivilegovaných LXC kontejnerů na Proxmoxu"
date: 2025-03-19
tags: [Proxmox, LXC, Backup, Troubleshooting]
---

## Jak vyřešit problém se zálohováním neprivilegovaných LXC kontejnerů na Proxmoxu

Při zálohování neprivilegovaného LXC kontejneru na NFS nebo CIFS se může objevit například:

```text
INFO: tar: /mnt/pve/NAS02/dump/vzdump-lxc-104-2025_03_16-01_00_05.tmp: Cannot open: Permission denied
```

`vzdump` běží na Proxmox hostiteli. Při záloze LXC ale používá mapované UID/GID neprivilegovaného kontejneru. Síťové úložiště nemusí tyto vlastníky, ACL nebo rozšířené atributy správně podporovat, a vytvoření dočasné pracovní oblasti proto selže.

📌 **Řešením je použít lokální pracovní adresář na Proxmox hostiteli a výsledný archiv dále ukládat na síťové úložiště.**

---

### 1. Ověření příčiny a volného místa

Na Proxmox hostiteli zkontrolujeme konfiguraci a volné místo v lokálním dočasném adresáři:

```bash
grep -E '^[[:space:]]*tmpdir:' /etc/vzdump.conf
df -h /var/tmp
stat -c '%a %U:%G %n' /var/tmp
```

Standardní `/var/tmp` má obvykle oprávnění `1777 root:root`. Lokální disk musí mít dostatek volného místa. U LXC může dočasná pracovní kopie, zejména v režimu `suspend`, potřebovat prostor blížící se velikosti použitých dat kontejneru.

---

### 2. Nejdřív jednorázový test

Než změníme globální konfiguraci, otestujeme jeden kontejner. Nahradíme `104` a `NAS02` skutečným CT ID a ID úložiště:

```bash
vzdump 104 --storage NAS02 --mode snapshot --compress zstd --tmpdir /var/tmp
```

Pokud záloha proběhne, příčinou byla dočasná oblast na síťovém úložišti.

---

### 3. Trvalé nastavení

Otevřeme globální konfiguraci na každém Proxmox uzlu, kde se záloha může spustit:

```bash
nano /etc/vzdump.conf
```

Přidáme nebo upravíme jediný aktivní řádek:

```text
tmpdir: /var/tmp
```

Změna se použije při dalším spuštění `vzdump`; restart hostitele není potřeba. Nastavení ověříme:

```bash
grep -E '^[[:space:]]*tmpdir:' /etc/vzdump.conf
```

⚠️ **`tmpdir` musí být na rychlém lokálním souborovém systému s dostatečnou kapacitou.** Nepoužívejte RAM disk ani malý kořenový oddíl, který by záloha mohla zaplnit. Pokud `/var/tmp` kapacitně nestačí, vytvořte samostatný lokální filesystem a nastavte `tmpdir` na jeho adresář.

---

### 4. Ověření výsledku

Po změně spustíme běžný naplánovaný job a zkontrolujeme jeho log. Záloha je použitelná až po ověření obnovy, proto alespoň u testovacího kontejneru provedeme restore pod novým CT ID a ověříme start a data aplikace.

Pokud chyba zůstává i s lokálním `tmpdir`, zkontrolujeme v konfiguraci kontejneru bind mounty a device mounty. Jejich obsah se standardně nezálohuje a pro spravované mount pointy musí být nastaveno `backup=1`:

```bash
pct config 104
```

---

### Shrnutí

✅ Dočasnou oblast měníme na Proxmox hostiteli, ne uvnitř kontejneru.  
✅ Nejdřív změnu ověříme pro jeden job pomocí `--tmpdir`.  
✅ Před globálním nastavením zkontrolujeme kapacitu lokálního disku.  
✅ Po opravě ověříme nejen dokončení zálohy, ale také testovací obnovu.

Další informace: [Proxmox VE Administration Guide](https://pve.proxmox.com/pve-docs/pve-admin-guide.html#vzdump_configuration)
