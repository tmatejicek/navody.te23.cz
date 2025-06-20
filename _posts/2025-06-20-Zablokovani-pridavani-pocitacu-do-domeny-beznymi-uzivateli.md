---
title: "Zablokování přidávání počítačů do domény běžnými uživateli"
date: 2025-06-20
tags: [Active Directory, Security, ms-DS-MachineAccountQuota]
layout: post
---

## Zablokování přidávání počítačů do domény běžnými uživateli

V tomto článku popisujeme, proč a jak nastavit atribut `ms-DS-MachineAccountQuota` na hodnotu 0 v prostředí Active Directory. Cílem je omezit, kdo smí přidávat počítače do domény, a tím výrazně zvýšit zabezpečení doménového prostředí.

---

### 1. Co je ms-DS-MachineAccountQuota

Atribut `ms-DS-MachineAccountQuota` určuje, kolik počítačových účtů může běžný (neprivilegovaný) uživatel vytvořit v doméně bez speciálních oprávnění.

* Výchozí hodnota je `10`
* Každý uživatel může vytvořit až 10 počítačových objektů
* Omezení se **nevztahuje na administrátory**

⚠ To znamená, že kdokoliv s doménovým účtem může bez dalšího oprávnění připojit až 10 zařízení do domény.

---

### 2. Proč změnit hodnotu na 0

#### 2.1 Zabezpečení proti zneužití

* Útočník s běžným účtem může přidat počítač do domény a eskalovat oprávnění

📌 Omezení možnosti přidávat zařízení chrání doménu před neautorizovaným rozšiřováním

#### 2.2 Delegace správy

* Ve většině organizací přidávají zařízení správci
* Delegace práv na konkrétní OU je bezpečnější a lépe kontrolovatelná
* Uživatelé, kteří potřebují přidávat zařízení, mohou mít delegovaná práva na konkrétní organizační jednotky (OU)
---

### 3. Jak změnit ms-DS-MachineAccountQuota

#### 3.1 Zjištění aktuální hodnoty

```powershell
Get-ADObject ((Get-ADDomain).distinguishedname) -Properties ms-DS-MachineAccountQuota | Select-Object ms-DS-MachineAccountQuota
```

#### 3.2 Změna na hodnotu 0

```powershell
Set-ADDomain (Get-ADDomain).distinguishedname -Replace @{"ms-ds-MachineAccountQuota"="0"}
```

---

### 4. Doporučení pro praxi

* Po nastavení na 0 otestujeme přidání zařízení jako běžný uživatel
* Delegujme práva na vybrané OU (např. přes konzoli Active Directory Users and Computers)
* Dokumentujme změnu v bezpečnostní politice organizace

---

## Shrnutí

✅ Atribut `ms-DS-MachineAccountQuota` omezuje počet zařízení, které může uživatel přidat do domény  
✅ Výchozí hodnota `10` představuje bezpečnostní riziko  
✅ Nastavení na `0` zakáže běžným uživatelům přidávání zařízení bez oprávnění  
✅ Administrátoři nebo delegovaní uživatelé mohou zařízení přidávat i nadále  
✅ Změna zvyšuje kontrolu nad prostředím a zabraňuje neautorizovanému připojení strojů
