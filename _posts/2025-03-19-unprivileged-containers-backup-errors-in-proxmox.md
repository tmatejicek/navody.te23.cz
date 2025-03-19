---
title: "Jak vyřešit problém se zálohováním neprivilegovaných LXC kontejnerů na Proxmoxu"
date: 2025-03-19
tags: [Proxmox, LXC, Backup, Troubleshooting]
---

## Jak vyřešit problém se zálohováním neprivilegovaných LXC kontejnerů na Proxmoxu

Při zálohování LXC kontejnerů na Proxmoxu se můžete setkat s chybou typu:

```
INFO: tar: /mnt/pve/NAS02/dump/vzdump-lxc-104-2025_03_16-01_00_05.tmp: Cannot open: Permission denied
```

Tento problém se obvykle týká neprivilegovaných LXC kontejnerů a jejich interakce se síťovým úložištěm. Důvodem je, že vzdump (nástroj na zálohování) využívá dočasný soubor ve složce úložiště, kam neprivilegovaný kontejner nemusí mít přístup. 

## **Řešení**: Změna dočasného adresáře pro vzdump

Řešení spočívá v úpravě konfiguračního souboru `/etc/vzdump.conf`, kde nastavíme dočasnou složku na místní disk. 

### **Postup opravy**
1. **Otevřete konfiguraci vzdump:**
   ```bash
   nano /etc/vzdump.conf
   ```

2. **Přidejte nebo upravte řádek:**
   ```bash
   tmpdir: /var/tmp
   ```
   Tento řádek zajistí, že dočasné soubory budou ukládány do `/var/tmp`, což je lokální úložiště, které není ovlivněno oprávněními na síťových úložištích.

3. **Uložte změny a zavřete soubor** (v nano stiskněte `CTRL + X`, pak `Y` a potvrďte `Enter`).

4. **Spusťte zálohování znovu.**

### **Proč to funguje?**
- `/var/tmp` je lokální složka, kde nemají síťová oprávnění vliv.
- Vyhnete se problémům s právy na síťových úložištích (NFS, CIFS).
- Zálohování bude probíhat bez chyb a neprivilegované kontejnery nebudou mít problém s přístupem k dočasným souborům.

## **Shrnutí**
Pokud narazíte na problém se zálohováním neprivilegovaných LXC kontejnerů na Proxmoxu, přidejte do `/etc/vzdump.conf` řádek:

```bash
tmpdir: /var/tmp
```

Tato jednoduchá úprava zabrání problémům s oprávněními a umožní bezproblémové zálohování neprivilegovaných kontejnerů.
