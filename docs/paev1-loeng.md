# Päev 1: Mõtteviis Zabbixi näitel

**Kursus:** IT-monitooring ja jälgitavus Zabbixi abil
**Kestus:** ~päev (loeng vahelduvad demoga)
**Tase:** Edasijõudnud — eeldame, et Zabbix on teil juba töös ja igapäevane

---

## 🎯 Õpiväljundid

Pärast seda päeva oskad:

- Eristada seiret ja jälgitavust ning selgitada, miks jälgitavus on seire ülemhulk
- Kirjeldada kolme sammast (mõõdikud, logid, jäljed) ja näidata, kus need Zabbixis on ja kus veel pole
- Eristada black-box ja white-box lähenemist ning põhjendada, miks mõlemat on vaja
- Valida nelja kuldsignaali abil, mida üldse monitoorida
- Selgitada SLI/SLO/veaeelarve mõtet ja siduda see Zabbixi SLA-raportiga
- Hinnata häiret selle järgi, kas see paneb inimese tegutsema või toodab müra
- Põhjendada, miks mallid, sõltuvused ja toili-kaotus on suure halduskoormuse juures kõige olulisemad

---

## 1. Kaks päeva, kaks küsimust

Need kaks päeva jagunevad selgelt. Täna räägime **miks** — kuidas monitooringule ja jälgitavusele mõelda. Homme tuleb **kuidas** — kapoti alla ja Zabbixist väljapoole: mõõdikud sügavuti, tracing ja OpenTelemetry, arhitektuurid, full-stack, turve ja automatiseerimine.

Te oskate Zabbixit juba. Te ei tulnud siia õppima, kuidas hosti lisada. Täna lisame mõttemudeli, mille sees Zabbix on üks tugev tükk — ja vaatame iga mõiste juures, kus ta täpselt teie Zabbixis elab.

Tööriista paigaldad päevaga. Mõtteviisi muutmine võtab kuid. Seesama mõtteviis eristab meeskonda, kes upub häiretesse, sellest, kes magab öösel.

---

## 2. Zabbix kui ühine sõnavara

Enne kui läheme mõistetele, paneme ühise sõnavara paika. See pole õpetus — pigem kaart, mille peale me kogu päeva mõisteid asetame.

### Liides ja arhitektuur

Zabbixi liides jaguneb kolmeks loogiliseks osaks. **Monitoring** näitab, mida sa praegu näed — problems, latest data. **Data collection** määrab, mida ja kuidas kogud — hosts, templates. **Alerts**, **Reports** ja **Users** katavad teavitamise, raportid ja ligipääsu.

Arhitektuuri saab kokku võtta nelja rolliga: server on aju, andmebaas on ajalugu, proxy kogub eemalt, agent istub jälgitaval masinal. Andmed liiguvad agentidelt ja SNMP-st läbi proxy serverisse, sealt andmebaasi ja frontendi.

Agent suhtleb serveriga kahel viisil. **Passiivses** režiimis server küsib ja agent vastab — sobib lähivõrku. **Aktiivses** režiimis agent ühendub ise serveriga — sobib tulemüüri või NAT-i taha. Server majas tähendab tavaliselt passiivset, server kaugel või DMZ-s aktiivset.

### Mõisted, mida kogu päeva kasutame

**Host** on jälgitav üksus — server, switch, teenus. **Host group** grupeerib hoste õiguste ja mallide jaoks. **Item** on üks mõõdik; tüübid on agent, SNMP, HTTP, arvutatud.

Latest data näitab värskeid väärtusi ja seda, kas item üldse kogub. Problems on koht, kus probleemid näha — mis, millal, kui tõsine — ning sa filtreerid neid tagide ja raskusastme järgi.

Probleemi peale käivitub **action** (teade, eskaleerimine, skript), teade läheb **media type**'i kaudu kuhugi (e-post, Slack, Teams), ja sündmus laheneb kas automaatselt, käsitsi sulgemisega või acknowledge'iga.

Ülejäänu on teil samuti tuttav: maintenance summutab probleemid hooldusaknal, template on korduvkasutatav itemite ja triggerite komplekt, dashboard koondab widgetid ühele paneelile, map näitab topoloogiat host-staatusega. UserParameter ja Agent 2 pluginad lubavad oma skripte ja valmis-integratsioone (Docker, andmebaasid). Network discovery skaneerib võrku, auto-registration laseb aktiivsel agendil end ise registreerida.

See on teie igapäev. Edasi vaatame, millise mõttemudeli sisse see kõik mahub.

---

## 3. Jälgitavus on mõtteviis, mitte tööriist

Kujuta ette tüüpilist olukorda. Teie Zabbixis on sadu hoste, kõik üleval, CPU ja mälu normis, triggerid korras. Siis helistab keegi: teenus on aeglane.

Zabbix vastas täpselt sellele küsimusele, mille te talle ette seadsite — "kas hostid on üleval, kas ressursid on normis". Aga kasutaja küsimus oli teine. See vahe ongi tänase päeva sisu.

!!! info "Jälgitavuse Venn-diagramm"
    Seire ja jälgitavus kattuvad — mõlemad jälgivad mõõdikuid ja saadavad häireid. Erinevus on servades: seire jälgib **teadaolevaid** probleeme reeglite alusel, jälgitavus aitab tuvastada ka neid, mida sa ette ei näinud — "unknown-unknowns", probleeme, mille olemasolu sa ei osanud aimata.

---

## 4. Seire vs jälgitavus

Seire on eelnevalt määratud küsimuste jälgimine: lävi → trigger. Sa tead ette, mida jälgida.

Jälgitavus lubab tagantjärele küsida ka seda, mida ette ei näinud. Ta ei ole seire vastand — ta on tema ülemhulk. Seire on jälgitavuse sees, mitte tema kõrval.

Zabbixis on seire väga selgelt nähtav. Trigger nagu `last(/host/system.cpu.util) > 5` on küsimus, mille sa **enne** püstitasid. Aga küsimust "miks see endpoint on aeglane ainult ühele kliendile teisipäeviti?" sa triggerina kirja ei pannud — sa ei teadnud, et seda vaja läheb. Jälgitavus on suutlikkus sellisele küsimusele tagantjärele vastata.

---

## 5. Kolm sammast: mõõdikud, logid, jäljed

Jälgitavus toetub kolmele sambale. **Mõõdikud** ütlevad, kui palju — arv ajas. **Logid** ütlevad, mis täpselt juhtus — detail. **Jäljed** ütlevad, kuhu aeg kulus — päring läbi teenuste.

Zabbixis on need kolm sammast erineval küpsusastmel:

| Sammas | Olukord Zabbixis |
|---|---|
| Mõõdikud | Zabbixi kodu — itemid, mallid, SNMP |
| Logid | log-item teeb mustri-tuvastust, mitte täismahulist logihaldust |
| Jäljed | täna väljas; 8.0 LTS toob OpenTelemetri sisse |

Mõõdikutes on Zabbix tugev. Logide poolel ta tunneb mustreid, aga ei asenda logihalduse tööriista. Jäljed tulevad alles — see on hea teada, kui plaanid arhitektuuri paari aasta peale.

---

## 6. Black-box vs white-box

**Black-box** testib süsteemi väljast, nagu kasutaja: kas teenus vastab? **White-box** mõõdab seest: CPU, järjekord, rakenduse enda mõõdikud.

Black-box ütleb "katki", white-box ütleb "miks". Mõlemat on vaja. Ainult white-box jätab lõksu: kõik sisemised mõõdikud rohelised, aga kasutaja ikka ligi ei pääse, sest probleem on kuskil vahepeal — DNS, load balancer, sertifikaat.

Zabbixis teete te juba mõlemat. Black-box on web scenario, simple check (ping, port), HTTP agent. White-box on agent item, SNMP, Agent 2 pluginad. Uus pole tegevus, vaid nimi ja teadlikkus, et need on kaks erinevat vaadet samale teenusele.

---

## 7. Mida monitoorida — neli kuldsignaali

Kui ei tea, kust alustada, alusta Google SRE neljast kuldsignaalist:

- **Latentsus** — kui kaua vastus võtab
- **Liiklus** — kui palju nõudlust on
- **Vead** — kui suur osa päringutest ebaõnnestub
- **Küllastus** — kui täis ressurss on

Need neli katavad enamiku, mida kasutaja päriselt tunneb. Zabbixis on igaühel oma kodu: latentsus tuleb web scenario vastusajast, liiklus päringud-sekundis itemist, vead HTTP 5xx loendurist või triggerist, küllastus CPU/mälu/järjekorra/ketta mõõdikutest.

---

## 8. Sildid, dimensioonid ja kardinaalsus

Silt on mõõtmik, mille järgi andmeid lõigata — teenus, keskkond, regioon. Idee on universaalne: Zabbix tags, Prometheus labels ja OpenTelemetry attributes on sama mõte eri nimedega. Sama sõnavara tähendab, et hüppad ühest tööriistast teise ilma uut mudelit õppimata.

Zabbixis seovad tagid hoste, triggereid ja sündmusi, ja sa filtreerid nendega probleemivaadet.

!!! warning "Kardinaalsuse lõks"
    Kõrge kardinaalsus — silt, millel on väga palju erinevaid väärtusi, nagu `user_id` või IP-aadress — plahvatab mõõdiku-andmebaasi. Iga unikaalne väärtus loob uue aegrea. Kõrge kardinaalsus kuulub logidesse ja jälgedesse, mitte mõõdikusiltidesse.

Praktikas: kasuta siltides asju, mida on lõplik hulk (keskkond, teenus, regioon). Asju, mida on lõpmatu hulk (kasutaja, sessioon, päringu ID), hoia logides.

---

## 9. Teenusmõtlemine ja sõltuvused

Host on detail. Teenus on lubadus kasutajale. Kasutaja ei hooli serverist number 17 — ta hoolib, kas makse toimib.

Rikked levivad sõltuvusgraafi mööda: sümptom on üleval, põhjus all. Jagatud sõltuvus — andmebaas, DNS, autentimine — annab suurima plahvatusraadiuse, sest selle alla jääb korraga palju teenuseid.

Zabbixis on selle jaoks **business service monitoring**. Defineerid teenuse, seod ta alusosadega (olemasolevate triggeritega), ja Zabbix arvutab teenuse oleku ja SLA. Host → teenus annab teenuse-puu, mis toimib sillana CMDB poole. See on samm hostide loendist teenuste kaardi juurde.

---

## 10. SLI / SLO / veaeelarve

**SLI** on kasutajakogemuse mõõt — edukate päringute protsent, p99 latentsus — mitte CPU. **SLO** on selle siht, näiteks 99,9% kuus. **Veaeelarve** on 1 − SLO.

100% on vale eesmärk. Ta on liiga kallis ja kasutaja ei märkagi vahet 99,9% ja 100% vahel. Veaeelarve on kulutatav ressurss: kui seda on järel, võid riskida ja muudatusi teha; kui ta on otsas, stabiliseerid.

Zabbixis on SLA-raport see mõõtmiskoht. Seod teenuse SLA-eesmärgiga, ja Zabbix arvutab kättesaadavuse: uptime, allajäämine, kasutatud veaeelarve. 99,9% kuus tähendab umbes 43 minutit lubatud katkestust. See teeb arutelu konkreetseks — number otsustab, mitte valjeim hääl koosolekul.

---

## 11. Toil — korduv käsitsitöö

SRE-s on **toil** korduv, käsitsi tehtav, automatiseeritav töö ilma püsiva väärtuseta. Tema halb omadus on, et ta skaleerub süsteemide arvuga: kaks korda rohkem süsteeme tähendab kaks korda rohkem toili.

Suure halduskoormuse — kümnete administraatorite ja sadade hostide — juures on toil teie suurim varjatud kulu. SRE eesmärk pole toili taluda, vaid teda mõõta ja automatiseerides kaotada.

Zabbixis on toili-kaotus juba sisse ehitatud. **Mall** standardiseerib: üks muudatus mallis jõuab kõigi seotud hostideni. **Makro** lubab host-erandi ilma kloonimata. **Low-level discovery** automatiseerib: leiab kettad, liidesed, konteinerid ise ja loob nende kohta itemid. Standardiseerimine pluss automatiseerimine — see ongi toili kadumine praktikas.

---

## 12. Häired: tegutsetav vs müra

Häire eesmärk on panna inimene tegutsema. Kui häire ei nõua tegevust, pole see häire — see kuulub dashboardile või logisse.

Häireväsimus on tegelik oht: müra koolitab inimese häireid eirama, ja siis jääb märkamata ka õige signaal. Lihtne test iga häire kohta: kui see käivitub kell kolm öösel, mida inimene teeb? Kui vastus on "ei midagi" või "vaatab hommikul", pole see öine häire.

Zabbixis on selle jaoks kaks tööriista. **Hoiata sümptomitest, mitte põhjustest** — kasutaja-mõjust, mitte iga sisemise detaili pärast. **Trigger-sõltuvus** tähendab, et kui andmebaas kukub, saad ühe häire, mitte viiskümmend. **Eskalatsioon** laseb öösel pageda ainult siis, kui on kasutaja-mõju.

---

## 13. Tegutse-kiht — auto-remediation

Monitooring ei pea lõppema häirega — ta võib reageerida. Lihtsad korduvad juhud sobivad automaatseks lahendamiseks: teenuse restart, ketta puhastus.

Ettevaatlikult: logi alati, mida automaatika tegi, ja ära peida juurpõhjust. Auto-remediation, mis vaikselt teenust taaskäivitab, võib varjata probleemi, mis vajaks päris parandust.

Zabbixis viib selle ellu action operation. Trigger läheb PROBLEM-olekusse → action → remote command või skript. Teenus maas → restart; ketas täis → puhastus. Hoia see lihtsate, hästi mõistetud juhtude jaoks ja logi iga tegevus.

Sellega saab kogu ahel kokku: **instrumenteeri → kogu → ... → hoiata → tegutse**.

---

## 14. Kokkuvõte — mis on täna tähtis

**Seire vastab teadaolevatele küsimustele; jälgitavus lubab esitada uusi.** Trigger on küsimus, mille sa enne püstitasid.

**Kolm sammast: mõõdikud, logid, jäljed.** Zabbix on tugev mõõdikutes; 8.0 toob jäljed (OpenTelemetry) sisse.

**Mõõda kasutajakogemust, mitte ainult masinat.** SLI/SLO/veaeelarve panevad kasutaja vaate numbri taha.

**Mõtle teenustes, mitte hostides.** Kasutaja hoolib teenusest; business service monitoring seob hostid teenuse-puusse.

**Hoiata sümptomitest, mitte põhjustest.** Üks häire andmebaasi kukkumisel, mitte viiskümmend.

**Toil on suure halduskoormuse suurim varjatud kulu.** Mallid, makrod ja LLD kaotavad teda.

**Häire peab panema inimese tegutsema.** Kell-kolm-öösel test eraldab signaali mürast.

Iga mõiste on seotud teie Zabbixiga. Homme viime need teostuseni.

---

## Allikad

### Ametlik dokumentatsioon
| Allikas | URL |
|---|---|
| Zabbix items | https://www.zabbix.com/documentation/current/en/manual/config/items/item |
| Zabbix triggers | https://www.zabbix.com/documentation/8.0/en/manual/config/triggers/trigger |
| Zabbix IT services (BSM) | https://www.zabbix.com/documentation/current/en/manual/it_services |
| Zabbix SLA report | https://www.zabbix.com/documentation/current/en/manual/it_services/sla |
| Trigger dependencies | https://www.zabbix.com/documentation/current/en/manual/config/triggers/dependencies |
| Low-level discovery | https://www.zabbix.com/documentation/current/en/manual/web_interface/frontend_sections/data_collection/hosts/discovery |
| Remote command operation | https://www.zabbix.com/documentation/7.4/en/manual/config/notifications/action/operation/remote_command |
| Zabbix roadmap (jäljed / OTel) | https://www.zabbix.com/roadmap |

### Teooria ja kontekst
| Allikas | URL |
|---|---|
| Google SRE Workbook | https://sre.google/workbook/table-of-contents/ |
| Eliminating Toil (Google SRE) | https://sre.google/sre-book/eliminating-toil/ |
| IBM — observability vs monitoring | https://www.ibm.com/think/topics/observability-vs-monitoring |
| IBM — observability pillars | https://www.ibm.com/think/insights/observability-pillars |
| Zabbix tags — usage ja guidelines | https://blog.zabbix.com/tags-in-zabbix-6-0-lts-usage-subfilters-and-guidelines/19565/ |

### Praktiline
| Allikas | URL |
|---|---|
| The Zabbix Book — frontend | https://www.thezabbixbook.com/ch02-zabbix-installation/frontend/ |
| Golden signals Zabbixis (Squadcast) | https://medium.com/@squadcast/golden-signals-monitoring-from-fundamental-principles-for-zabbix-and-nagios-users-8ff90f5b89ba |

---

*Järgmine — Päev 2: Zabbixist jälgitavuseni (mõõdikud sügavuti, tracing, OpenTelemetry, arhitektuurid, full-stack, turve, automatiseerimine).*