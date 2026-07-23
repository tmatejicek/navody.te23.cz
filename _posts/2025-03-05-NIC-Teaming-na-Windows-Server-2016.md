---
title: "Jak nastavit NIC Teaming na Windows Server 2016"
date: "2025-03-05"
tags: [NIC Teaming, Windows Server, Networking, LBFO]
layout: post
---

## Jak nastavit NIC Teaming na Windows Server 2016

NIC Teaming (též známý jako Load Balancing and Failover, LBFO) spojí více síťových adaptérů do jednoho logického rozhraní. Zvyšuje dostupnost a umožňuje rozložit více souběžných síťových toků mezi členy týmu. Jeden TCP tok ale obvykle nepřekročí rychlost jedné fyzické karty.

📌 **Jeden TCP tok obvykle nepřekročí rychlost jedné fyzické síťové karty.** Vyšší agregovaná propustnost se projeví až u více souběžných spojení.

⚠️ Tento postup je určený pro klasický **LBFO tým**, například pro management nebo aplikační provoz fyzického serveru. Pro nový externí virtuální přepínač Hyper-V je od Windows Serveru 2016 k dispozici **Switch Embedded Teaming (SET)**, který tým vytváří přímo ve vSwitchi. SET a LBFO nekombinujeme. Pokud potřebujeme RDMA/SMB Direct nebo nasazujeme SDN, použijeme SET; klasický LBFO tým RDMA vypíná.

---
## 1. Požadavky

- Windows Server 2016
- Minimálně dvě síťové karty stejné rychlosti; kombinace různých rychlostí není podporovaná pro správné rozkládání provozu
- Administrátorský přístup k serveru
- U režimů Static nebo LACP odpovídající konfigurace portů na fyzickém přepínači

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
     - **Switch Independent** (Nezávislý na přepínači) – bezpečná výchozí volba; nevyžaduje konfiguraci přepínače a členové mohou být zapojeni do různých přepínačů.
     - **Static Teaming** – vyžaduje shodnou statickou agregaci na přepínači.
     - **LACP (Link Aggregation Control Protocol)** – vyžaduje shodně nastavenou LACP skupinu na přepínači.
   - **Load Balancing Mode**:
     - **Dynamic** – doporučená obecná volba pro Windows Server 2016; rozkládá odchozí toky podle adres a portů a podle vytížení je přesouvá mezi členy.
     - **Hyper-V Port** – vhodný pro některé Hyper-V scénáře, protože přiřazuje virtuální porty jednotlivým členům týmu.
     - **Address Hash** – rozděluje toky podle hash hodnoty adres a případně transportních portů.
   - **Standby Adapter**: Můžete určit adaptér jako záložní (volitelné).
5. Klikněte na **OK** a počkejte na vytvoření týmu.

---
## 4. Ověření funkčnosti

1. Po vytvoření týmu přejděte do **Centra síťových připojení**:
   - Otevřete **Ovládací panely** → **Síť a internet** → **Centrum síťových připojení a sdílení**.
   - Klikněte na **Změnit nastavení adaptéru**.
2. Měli byste vidět nový virtuální síťový adaptér odpovídající vytvořenému týmu.
3. IP adresu, výchozí bránu a DNS nastavte na novém týmovém rozhraní. Na fyzických členech týmu IP konfiguraci nenastavujte.
4. Pro ověření dostupnosti můžete použít příkazový řádek:

   ```powershell
   Get-NetLbfoTeam
   Get-NetLbfoTeamMember
   ```

   Tento příkaz zobrazí stav a konfiguraci NIC Teamingu.
5. Zkuste provést **ping** na jiný server v síti a sledujte odezvu.

---
## 5. Testování odolnosti proti výpadku

⚠️ Test provádějte z lokální konzole nebo z relace s jinou síťovou cestou, abyste si při chybné konfiguraci neodpojili vzdálenou správu.

1. Z konzole serveru nebo z relace s alternativní cestou odpojte jednu ze síťových karet a ověřte, zda připojení stále funguje.
2. Připojte síťovou kartu zpět a ujistěte se, že se adaptér automaticky znovu připojí.

Tým lze vytvořit také reprodukovatelně pomocí PowerShellu:

```powershell
New-NetLbfoTeam -Name "Team1" `
  -TeamMembers "Ethernet 1","Ethernet 2" `
  -TeamingMode SwitchIndependent `
  -LoadBalancingAlgorithm Dynamic
```

---
## Závěr

✅ NIC Teaming zvyšuje dostupnost a agregovanou propustnost více souběžných spojení.  
✅ Pro běžný LBFO tým je bezpečnou výchozí volbou **Switch Independent** a **Dynamic**.  
✅ IP konfiguraci nastavujeme na týmovém rozhraní, nikoli na fyzických členech.  
✅ Konfiguraci vždy sladíme s fyzickým přepínačem a otestujeme výpadek každého člena týmu.  
❌ Pro nový Hyper-V vSwitch, RDMA nebo SDN nekombinujeme LBFO a SET.

Další informace: [New-NetLbfoTeam](https://learn.microsoft.com/en-us/powershell/module/netlbfo/new-netlbfoteam) a [Switch Embedded Teaming](https://learn.microsoft.com/en-us/windows-server/networking/technologies/hpn/hpn-software-only-features#switch-embedded-teaming-set)
