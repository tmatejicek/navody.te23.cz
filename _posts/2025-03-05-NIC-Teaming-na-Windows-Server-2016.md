---
title: "Jak nastavit NIC Teaming na Windows Server 2016"
date: "2025-03-05"
tags: [NIC Teaming, Windows Server, Networking, LBFO]
layout: post
---

## Jak nastavit NIC Teaming na Windows Server 2016

NIC Teaming (též známý jako Load Balancing and Failover, LBFO) umožňuje spojit více síťových adaptérů do jednoho logického rozhraní, čímž zajišťuje vyšší dostupnost a zlepšuje síťový výkon. Tento návod vás krok za krokem provede nastavením teaming na Windows Server 2016.

---
## 1. Požadavky

- Windows Server 2016
- Minimálně dvě síťové karty
- Administrátorský přístup k serveru

---
## 2. Otevření Správce serveru

1. Přihlaste se na server s administrátorskými oprávněními.
2. Otevřete **Správce serveru** (Server Manager).
3. Klikněte na **Místní server** (Local Server) v levém panelu.
4. V pravé části vyhledejte položku **NIC Teaming** – výchozí stav je „Zakázáno“.
5. Klikněte na **Zakázáno**, čímž se otevře okno pro konfiguraci teaming.

---
## 3. Vytvoření týmu síťových adaptérů

1. V okně **NIC Teaming** klikněte na **Tasks** (Úkoly) v pravém horním rohu a vyberte **New Team** (Nový tým).
2. Zadejte název týmu (např. „Team1“).
3. Ze seznamu dostupných adaptérů vyberte dvě nebo více síťových karet, které chcete zahrnout do týmu.
4. Klikněte na **Additional Properties** (Další vlastnosti) a nastavte:
   - **Teaming mode**: 
     - **Switch Independent** (Nezávislý na přepínači) – doporučené pro většinu případů.
     - **Static Teaming** – vyžaduje konfiguraci na síťovém přepínači.
     - **LACP (Link Aggregation Control Protocol)** – vyžaduje podporu LACP na přepínači.
   - **Load Balancing Mode**:
     - **Address Hash** (Výchozí) – vhodný pro většinu scénářů.
     - **Hyper-V Port** – optimalizováno pro virtuální prostředí Hyper-V.
     - **Dynamic** – doporučený režim kombinující výhody výše uvedených.
   - **Standby Adapter**: Můžete určit adaptér jako záložní (volitelné).
5. Klikněte na **OK** a počkejte na vytvoření týmu.

---
## 4. Ověření funkčnosti

1. Po vytvoření týmu přejděte do **Centra síťových připojení**:
   - Otevřete **Ovládací panely** → **Síť a internet** → **Centrum síťových připojení a sdílení**.
   - Klikněte na **Změnit nastavení adaptéru**.
2. Měli byste vidět nový virtuální síťový adaptér odpovídající vytvořenému týmu.
3. Klikněte pravým tlačítkem na tento adaptér, vyberte **Vlastnosti**, a zkontrolujte, zda je přiřazena správná IP adresa.
4. Pro ověření dostupnosti můžete použít příkazový řádek:

   ```powershell
   Get-NetLbfoTeam
   ```

   Tento příkaz zobrazí stav a konfiguraci NIC Teamingu.
5. Zkuste provést **ping** na jiný server v síti a sledujte odezvu.

---
## 5. Testování odolnosti proti výpadku

1. Odpojte jednu ze síťových karet a ověřte, zda připojení stále funguje.
2. Připojte síťovou kartu zpět a ujistěte se, že se adaptér automaticky znovu připojí.

---
## Závěr

NIC Teaming na Windows Server 2016 je užitečná technologie pro zvýšení dostupnosti a propustnosti sítě. Tento postup vám pomůže správně nakonfigurovat teaming a zajistit stabilní síťové připojení i při výpadku jednoho z adaptérů.
