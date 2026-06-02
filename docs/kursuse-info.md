# IT-süsteemide monitooringu ja jälgitavuse alused Zabbixi abil

Täienduskoolituse õppekava  
Haapsalu Kutsehariduskeskus  
Ehitajate tee 3, Uuemõisa, Haapsalu, 90401

---

## Üldinfo

**Õppekava nimetus**  
IT-süsteemide monitooringu ja jälgitavuse alused Zabbixi abil

**Õppekavarühm**  
Andmebaaside ja võrgu disain ning haldus

**Õppekava koostamise alus**  
Haapsalu Kutsehariduskeskuse IT-süsteemide nooremspetsialist õppekava (EHIS kood 215398).

**Õppekava kogumaht ja ülesehitus**

- 7 akadeemilist tundi
  - 6 akadeemilist tundi kontaktõpet
  - 1 akadeemiline tund iseseisvat tööd

---

## Sihtgrupp ja õppega alustamise tingimused

**Sihtgrupp**

- IT-administraatorid (nooremad, kesktasemel).

**Õppega alustamise tingimused**

- Osalejal on kogemus Zabbixiga ja ta on tegelenud IT-administreerimisega.

---

## Õpisündmuse eesmärk

Koolituse tulemusel on õppija valmis kasutama Zabbixi võimalusi IT-teenuste monitooringu ja jälgitavuse kavandamisel ning esmase praktilise seadistuse tegemisel.

---

## Õpiväljundid

Koolituse läbinu:

1. selgitab monitooringu ja jälgitavuse erinevust ning mõõdikute, logide ja jälgede rolli IT-teenuse toimivuse hindamisel;
2. seostab IT-teenuse komponentide sõltuvusi SLI, SLO ja veaeelarve mõistetega teenusepõhise vaate alusel;
3. kavandab näidisjuhtumi põhjal nutika teavitamise loogika, arvestades sõltuvusi, hüstereesi, eskaleerimist ja teavituste ülekülluse vähendamise põhimõtteid;
4. seadistab juhendatud praktilises ülesandes vähemalt ühe Zabbixi põhifunktsiooni, näiteks teenusevaate, automaattuvastuse või kohandatud kontrolli;
5. selgitab OpenTelemetry, Zabbix API ja Ansible kasutusvõimalusi Zabbixi-põhise monitooringu laiendamisel ja automatiseerimisel.

---

## Õppe sisu

### IT-teenuste jälgitavuse mõttemudel ja aluspõhimõtted (3 ak tundi)

- Observability vs monitoring  
- Kolm sammast (metrics, logs, traces)  
- Service thinking ja sõltuvused  
- SLI / SLO / Error Budget  
- Alerting filosoofia ja alert fatigue  
- Intsidentide haldus, postmortem, Google SRE põhimõtted  
- OpenTelemetry kui 2026 standard  
- Zabbix 8 LTS lühiülevaade

### Zabbixi nutikas seadistamine ja automatiseerimine (3 ak tundi)

- Zabbix Services moodul ja SLA  
- Smart alerting (dependencies, hysteresis, escalation)  
- LLD ja custom checks (UserParameter, HTTP, Script items)  
- Agendid (active/passive), proxy, turvalisus (PSK/TLS, audit log)  
- Automaatika: Zabbix API + Ansible, GitOps  
- Zabbix + Prometheus + Grafana koos  
- Vabatahtlik lab-kodutöö juhend

### Iseseisev töö – laboratoorne töö (1 ak t)

- Laboratoorne iseseisev töö kursuse repo põhjal püstitatud keskkonnas: näidis-IT-teenuse monitooringu kavandamine ja seadistamine.

---

## Õppemeetodid

- Lühi-loengud  
- Selgitus ja demonstratsioon  
- Interaktiivne arutelu osalejate kogemuse põhjal  
- Reflektiivsed küsimused  
- Kodutöö / laboratoorne töö

---

## Õppekeskkond

Õpe toimub Haapsalu Kutsehariduskeskuse arvutiklassis, tellija ruumides või veebis.

- Kohapealses õpperuumis on tagatud õppija individuaalne töökoht, koolitaja arvuti, projektor, tahvel jm vajalikud õppevahendid.  
- Veebiõppes osalemiseks peab õppijal olema toimiv internetiühendus, mikrofon ja kaamera ning valmidus oma kohalolekut autentida.  
- Esitlus toimub pakkuja keskkonnas.
- Õppe- ja praktikamaterjalid on kättesaadavad kursuse Giti-repos; iseseisva töö jaoks tõstab õppija repo põhjal vajaliku keskkonna oma arvutis üles (eeldus: Docker ja Git).

---

## Nõuded õpingute lõpetamiseks

Õppija:

- osaleb kontaktõppes vähemalt 75%;  
- on esitanud laboratoorse iseseisva töö, mis vastab hindamiskriteeriumidele.

**Hindamismeetod**

- Laboratoorne töö.

**Hindamiskriteeriumid**

Õppija esitab laboratoorse töö, mis koosneb praktilisest seadistusest ja lühikesest kirjalikust põhjendusest ning tõendab kõiki õpiväljundeid:

- seadistab vähemalt ühe Zabbixi põhifunktsiooni (teenusevaade, automaattuvastus või kohandatud kontroll) — õpiväljund 4;  
- selgitab monitooringu ja jälgitavuse erinevust ning mõõdikute, logide ja jälgede rolli valitud teenuse näitel — õpiväljund 1;  
- seostab komponentide sõltuvused SLI, SLO ja veaeelarvega (õpiväljund 2) ning kavandab teavitamise loogika, arvestades sõltuvusi, hüstereesi ja eskaleerimist (õpiväljund 3);  
- kirjeldab, kuidas lahendust saaks laiendada ja automatiseerida OpenTelemetry, Zabbix API ja Ansible abil — õpiväljund 5.

---

## Väljastatav dokument

Koolituse lõppedes väljastatakse õppijale:

- Haapsalu KHK täienduskoolituse tunnistus, kui õpingute lõpetamise nõuded on täidetud;  
- tõend, kui õpitulemusi ei saavutatud, kuid õppija võttis osa õppetööst (tõend väljastatakse vastavalt osaletud kontakttundide arvule ja läbitud teemadele).

---

## Koolitaja

Koolitajal on IT-alane kõrgharidus või vähemalt 3-aastane töökogemus IT-infrastruktuuri ja monitooringu valdkonnas ning täiskasvanute koolitamise kompetentsid.

**Õppekava koostaja(d)**  
Maria Talvik, Haapsalu Kutsehariduskeskuse IT kutseõpetaja ja koolitaja  
<maria.talvik@hkhk.edu.ee>