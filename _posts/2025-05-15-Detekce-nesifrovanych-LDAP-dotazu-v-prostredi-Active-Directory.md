---
title: "Detekce nezabezpečených LDAP vazeb v Active Directory"
date: 2025-05-15
tags: [LDAP, Active Directory, Security, Graylog]
layout: post
---

## Detekce nezabezpečených LDAP vazeb v Active Directory

Události `2886` až `2889` pomáhají najít klienty, kteří používají:

- SASL LDAP bind (Negotiate, Kerberos, NTLM nebo Digest) bez podpisu,
- simple bind přes spojení bez SSL/TLS.

📌 **Nejde o obecný detektor veškerého provozu na portu `389`.** LDAP na `389` může použít StartTLS nebo ochranu SASL a být bezpečný; naopak simple bind bez TLS přenáší přihlašovací údaje po síti bez odpovídající ochrany.

---

### 1. Význam událostí

Události najdeme na každém řadiči domény v logu **Directory Service**.

| Event ID | Význam |
|---|---|
| `2886` | Po startu AD DS upozorní, že řadič domény nevyžaduje LDAP signing. Neříká, že signing nepodporuje. |
| `2887` | Jednou za 24 hodin shrne povolené unsigned SASL bindy a simple bindy bez TLS. Neobsahuje seznam klientů. |
| `2888` | Jednou za 24 hodin shrne stejné typy bindů, které řadič domény odmítl, protože signing vyžaduje. |
| `2889` | Detailní událost pro jednotlivého klienta: IP adresa, identita a typ bindu. Vyžaduje diagnostickou úroveň `2`. |

📌 Event ID `2890` do této sady LDAP signing událostí nepatří.

---

### 2. Zapnutí detailního logování

Nastavení provedeme na všech zapisovatelných řadičích domény. Hodnota se použije okamžitě, restart není nutný.

```powershell
reg add "HKLM\SYSTEM\CurrentControlSet\Services\NTDS\Diagnostics" /v "16 LDAP Interface Events" /t REG_DWORD /d 2 /f
```

Pro více DC použijeme Group Policy Preferences:

1. V GPO aplikovaném pouze na OU **Domain Controllers** otevřeme **Computer Configuration > Preferences > Windows Settings > Registry**.
2. Vytvoříme nebo aktualizujeme hodnotu `16 LDAP Interface Events` v klíči `HKLM\SYSTEM\CurrentControlSet\Services\NTDS\Diagnostics`.
3. Typ nastavíme na `REG_DWORD` a hodnotu na `2`.

⚠️ Úroveň `2` zvyšuje množství událostí. Po dokončení inventury ji můžeme vrátit na `0`, případně upravit nebo odstranit příslušnou GPO Preference:

```powershell
reg add "HKLM\SYSTEM\CurrentControlSet\Services\NTDS\Diagnostics" /v "16 LDAP Interface Events" /t REG_DWORD /d 0 /f
```

---

### 3. Vyhodnocení událostí 2889

Následující skript zobrazí jedinečné kombinace IP adresy, účtu a typu bindu za posledních 24 hodin:

```powershell
$ComputerName = 'dc1.firma.cz'
$StartTime = (Get-Date).AddHours(-24)

$Rows = foreach ($Event in Get-WinEvent -ComputerName $ComputerName -FilterHashtable @{
    LogName  = 'Directory Service'
    Id       = 2889
    StartTime = $StartTime
}) {
    $Xml = [xml]$Event.ToXml()
    $Data = @($Xml.Event.EventData.Data | ForEach-Object { [string]$_.'#text' })

    $Endpoint = $Data[0]
    if ($Endpoint -match '^\[(?<ip>.+)\]:(?<port>\d+)$' -or
        $Endpoint -match '^(?<ip>.+):(?<port>\d+)$') {
        $IpAddress = $Matches.ip
        $SourcePort = [int]$Matches.port
    }
    else {
        $IpAddress = $Endpoint
        $SourcePort = $null
    }

    $BindType = switch ([int]$Data[2]) {
        0 { 'SASL bez podpisu' }
        1 { 'Simple bind bez TLS' }
        default { "Neznámý typ ($($Data[2]))" }
    }

    [pscustomobject]@{
        DomainController = $ComputerName
        TimeCreated      = $Event.TimeCreated
        IPAddress        = $IpAddress
        SourcePort       = $SourcePort
        Identity         = $Data[1]
        BindType         = $BindType
    }
}

$Rows |
    Sort-Object IPAddress, Identity, BindType -Unique |
    Format-Table DomainController, IPAddress, Identity, BindType -AutoSize
```

📌 Skript spustíme pro každý DC. Klient se může připojovat k různým řadičům, takže kontrola jediného serveru není úplná.

---

### 4. Vyhodnocení v Graylogu

Pokud Beats input používá výchozí prefixování polí, přehled relevantních událostí získáme dotazem:

```text
winlogbeat_winlog_channel:"Directory Service" AND winlogbeat_event_code:(2886 OR 2887 OR 2888 OR 2889)
```

Pro hledání konkrétních klientů použijeme pouze detailní události:

```text
winlogbeat_winlog_channel:"Directory Service" AND winlogbeat_event_code:2889
```

💡 Ve výsledcích otevřeme zprávu a ověříme, do kterých polí Winlogbeat uložil IP adresu a identitu. Podle verze Windows a renderování události mohou být údaje také součástí pole `message`; názvy polí proto před vytvořením dashboardu ověříme na skutečné zprávě.

#### 4.1 Konfigurace Winlogbeatu

Na doménových řadičích musí Winlogbeat číst log `Directory Service`:

```yaml
winlogbeat.event_logs:
  - name: Directory Service
    event_id: 2886, 2887, 2888, 2889
```

Po změně ověříme konfiguraci a restartujeme službu podle [návodu k Winlogbeatu a Graylogu]({% post_url 2026-04-15-Nastaveni-Winlogbeat-pro-odesilani-Windows-event-logu-do-Graylogu %}).

📌 Událost `2889` vyžaduje diagnostické logování z kapitoly 2. Souhrnné události `2887` a `2888` na tomto nastavení nezávisí.

---

### 5. Náprava a vynucení LDAP signing

Nejprve odstraníme všechny závislosti nalezené v událostech `2889`:

- simple bind převedeme na LDAPS (`636`) nebo LDAP s StartTLS a ověříme certifikační řetězec i jméno serveru,
- SASL klienta nastavíme tak, aby vyžadoval signing,
- aplikace s pevně uloženým heslem přesuneme na servisní účet s minimálními oprávněními a zabezpečené spojení.

Po období bez událostí `2887` nastavíme v GPO pro řadiče domény:

**Computer Configuration > Policies > Windows Settings > Security Settings > Local Policies > Security Options > Domain controller: LDAP server signing requirements > Require signing**

Pro klienty ověříme nastavení:

**Network security: LDAP client signing requirements > Negotiate signing**

⚠️ Po otestování kompatibility lze na vybraných systémech přejít na `Require signing`. Změnu nejprve pilotujeme; nekompatibilní LDAP aplikace mohou po vynucení přestat fungovat.

Funkci ověříme pomocí `ldp.exe`: simple bind na portu `389` bez TLS má po vynucení skončit chybou **Strong Authentication Required**, zatímco LDAPS nebo StartTLS musí nadále fungovat. LDAP channel binding je samostatné bezpečnostní nastavení a vyhodnocuje se jinými událostmi.

---

### Shrnutí

✅ Událost `2887` ukazuje denní souhrn povolených nezabezpečených vazeb.  
✅ Událost `2889` identifikuje konkrétní klienty a vyžaduje diagnostickou úroveň `2`.  
✅ Winlogbeat musí z doménových řadičů sbírat log **Directory Service**.  
✅ Nejdřív opravíme klienty a teprve potom vynutíme LDAP signing.  
⚠️ Vynucení vždy pilotujeme, protože nekompatibilní aplikace mohou přestat fungovat.

### Zdroje

- [How to enable LDAP signing in Windows Server](https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/enable-ldap-signing-in-windows-server)
- [LDAP signing for Active Directory Domain Services](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/ldap-signing)
