# Päev 2: Zabbixist jälgitavuseni

**Kursus:** IT-monitooring ja jälgitavus Zabbixi abil
**Kestus:** ~päev
**Tase:** Edasijõudnud — eeldame eilset päeva ja Zabbixi igapäevast kasutust

---

## 🎯 Õpiväljundid

Pärast seda päeva oskad:

- Eristada mõõdikut ja sündmust ning selgitada, miks häiret ei panda mõõdikule, vaid tingimusele
- Põhjendada, miks percentiilid (p95/p99) on SLO jaoks paremad kui keskmine, ja eristada RED- ja USE-meetodit
- Kirjeldada, mis on jälg, span ja konteksti levitamine, ning selgitada, miks Zabbix ei ole APM
- Valida sämplimisstrateegia (head- vs tail-based) ja põhjendada selle kulu-mõju
- Selgitada OpenTelemetry rolli liimina ja kavandada instrumenteerimise järjekorda
- Kavandada jälgitavuse arhitektuuri kihtidena ning eristada tsentraalset ja föderaalset kogumist
- Valida turbemehhanismid (PSK/TLS, RBAC, secret-haldus) vastavalt keskkonna suurusele
- Kirjeldada intsidendi elukaart SLI-st postmortemini ja otsustada, mida automatiseerida

---

## 1. Kus me eile jäime

Eile rääkisime mõttemudelist: seire versus jälgitavus, kolm sammast, teenusmõtlemine. Panime paika SLI, SLO ja veaeelarve, vaatasime SLA-d, häireväsimust ja toili. Zabbix joonistus välja tugeva mõõdikupõhjana — BSM, SLA, mallid, LLD.

Täna läheme kapoti alla. Kõigepealt mõõdikud sügavamalt: percentiilid, RED/USE, anti-mustrid. Siis lisame puuduvad sambad: jäljed, logid, OpenTelemetry. Lõpuks paneme pildi kokku — Zabbix koos LGTM-i, OTel-i, turbe ja automatiseerimisega.

Eile oli küsimus "kuidas mõelda". Täna on küsimus "kuidas ehitada".

---

## 2. Mõõdikud sügavuti

### Mõõdik ja sündmus pole sama asi

Zabbixi item on mõõdik — pidev mõõtmine ajas, sama mis Prometheuse või OpenTelemetry metric (time series koos siltidega). Mõõdik on dashboardide, SLI-de ja häirereeglite alus.

Sündmus on midagi muud. Zabbixi event tekib triggeri olekumuutusest; Prometheuses tekib alert event siis, kui alerting rule aktiveerub. Sündmus on mõõdikust **tuletatud** seisundimuutus, mitte mõõdik ise.

See vahe on praktiline: sa ei pane häiret mõõdikule otse, vaid tingimusele, ja sündmus tekib siis, kui see tingimus oleku muudab. Ahel on mõõdik (item) → trigger (reegel) → sündmus (event) → häire/intsident.

### Item key vs sildid

Siin lahkneb Zabbix ja Prometheuse/OTel maailm. Zabbixis on dimensioonid item key sees — `net.if.in[eth0]`, `net.if.in[eth1]` — ja iga uus dimensioon tähendab uut item key'd. Host on peamine objekt.

Prometheuse ja OTel mudelis on dimensioonid siltides:

```
net_in_bytes{host="srv01", iface="eth0"}
```

Sildid on paindlikud metaandmed, mille järgi on lihtne filtreerida ja agregeerida — `sum(net_in_bytes) by (host)`. Sama mõõdik, palju lõikeid, ilma uut item key'd loomata. See on hea teada, kui ehitad Zabbixi kõrvale Prometheuse-põhist kihti.

### Push vs pull

Kaks viisi telemeetriat koguda. **Pull**: seiresüsteem läheb ise andmed võtma — Prometheus scrape'ib `/metrics` endpoint'i, otsustab ise, millal koguda, ja teenuse-avastus on sisse ehitatud. **Push**: andmed saadetakse seiresüsteemi — Zabbixi aktiivne agent ja OTLP töötavad nii. Push sobib NAT-i ja tulemüüride taha ning lühiajaliste tööde (batch, serverless) jaoks.

Zabbix oskab mõlemat. Eile nägime passiivset (pull) ja aktiivset (push) agenti — sama jaotus kehtib kogu telemeetriamaailmas.

### Percentiilid — keskmine valetab

SLO jaoks mõõda saba, mitte keskmist. Keskmine varjab probleeme: kui enamik päringuid on kiired ja väike osa väga aeglane, jääb keskmine ilus, aga osa kasutajaid kannatab. p95 ja p99 näitavad neid aeglasemaid päringuid.

Tail latency määrab kasutajakogemuse. Wise'i makse-checkouti puhul ei loe, et "keskmine vastusaeg on 200 ms" — loeb, et iga sajas klient ootab kolm sekundit. p95/p99 annavad varajase hoiatuse süsteemi probleemidest, enne kui keskmine üldse liigub.

### RED ja USE

Kaks vaatenurka, mis täiendavad eile õpitud nelja kuldsignaali.

**USE** kirjeldab ressursse: Utilization (kui hõivatud), Saturation (kas piiri lähedal), Errors (kas esineb vigu), pluss kestus. See aitab leida **põhjuse** ressursi tasandil.

**RED** kirjeldab teenust kasutaja vaatest: Rate (kui palju päringuid), Errors (kui palju vigu), Duration (kui kaua võtavad). See näitab **mõju** teenuse tasandil.

Mõlemat koos: kõrge CPU kasutus (USE) → pikk järjekord kettal (USE) → pikk vastusaeg (RED) → kasutajad kogevad viga (RED). USE leiab juurpõhjuse, RED näitab, mis kasutajani jõuab.

### Exemplars — sild mõõdikult jäljele

Mõõdik ütleb, et midagi on aeglane, aga ei ütle, miks. Exemplar lahendab selle: histogrammi punkt kannab kaasas konkreetse `trace_id`. Näed graafikul spike'i → klikid punktil → exemplar sisaldab trace ID-d → hüppad otse sellesse jälge → näed täpselt, mis päringut aeglustas. See on otsetee mõõdikute maailmast jälgede maailma, ja me tuleme selle juurde tracingu osas tagasi.

### Zabbix mõõdikuallikana

Küsimus pole "kas Zabbix või uus stack". Zabbixi mõõdikud lähevad Grafanasse datasource'ina või Prometheuse exporteri kaudu, ja siis on infra-mõõdikud samas vaates app-metrics'iga. Zabbix ei pea olema eraldi maailm — ta on üks mõõdikuallikas ühises vaates Prometheuse, OTel-i ja rakenduse mõõdikute kõrval.

!!! warning "Metrics anti-mustrid"
    **Counter ilma `rate()` või `increase()`-ta** näitab ainult kasvavat joont — mõttetu graafik. **Latentsus ainult keskmisena** peidab kasutajate halbu kogemusi. **Kõrge kardinaalsusega sildid** (`user_id`, `session_id`, IP) tapavad mõõdiku-andmebaasi salvestuskulu.

    Disaini tasandil: mõõdik ilma kasutajata ("kes seda vaatab?"), häire ilma tegevuseta ("mida ma nüüd teen?"), dashboard ilma küsimuseta ("millisele küsimusele see vastab?"). Iga mõõdik peaks vastama küsimusele. Kui küsimust pole, pole mõõdikut tõenäoliselt vaja.

---

## 3. Tracing ja APM

### Mis on APM

Application Performance Monitoring näitab kogu päringu teekonda ja kasutajakogemust — miks kasutaja ootab. Neli tükki: jaotatud tracing (päringu tee läbi teenuste), transaction breakdown (kuhu aeg rakenduse sees kulub — kood, väliskutsed, SQL, cache), RUM (päris kasutaja brauserist — laadimisaeg, Core Web Vitals, JS-vead) ja service map (teenuste topoloogia, automaatselt jälgedest koostatud).

### Kas Zabbix on APM? Aus vastus: ei

Zabbixil **on** web scenarios (synthetic), browser item (7.0+), JMX/Java gateway, HTTP agent items. Zabbixil **ei ole** jaotatud tracingut, koodi-instrumenteerimist, transaktsiooni breakdowni, RUM-i ega automaatset topoloogiat. Zabbix katab infra ja synthetic-kontrolli. Päris APM on OTel pluss tracing-backend. See on aus piir, mitte puudus — Zabbix lihtsalt teeb teist tööd.

### Jälg ja span

Jälg on ühe päringu "elulugu" teenuste vahel — `O→O→O···→O`. Span on selle ajaosa: üks tükk ajatelg jälje sees. Mõõdik ütleb "checkout võttis 4 sekundit"; jälg näitab, kus need 4 sekundit kulusid — kas gateways, order-service'is, makses või andmebaasis.

### Konteksti levitamine

Kuidas eraldi teenuste span'id üheks jäljeks liimitakse? W3C `traceparent` standard: versioon, trace-id, span-id, flags. Iga teenusekõne kannab seda HTTP päises või metaandmetes. Ilma selleta jäävad span'id orvuks ega seo end üheks jäljeks. Kui teenus A kutsub teenust B, peab A oma trace-konteksti B-le edasi andma — muidu paistab B töö eraldi looga.

### Span'i osad

Span sisaldab atribuute (`http.method`, `db.statement`, `user.tier`), sündmusi (ajatemplitud märked, nt exception või retry) ja staatust (OK või Error). Oluline erinevus mõõdikust: span'is **on** kõrge kardinaalsus lubatud. `user_id` mõõdikusildis on lõks; span-atribuudis on ta täpselt see, mida tahad — sest jälge ei agregeerita time series'iks.

### Sämplimine — kõike ei salvestata

Jälgi on liiga palju, et kõiki hoida. Kaks lähenemist:

| | Head-based | Tail-based |
|---|---|---|
| Otsus | trace'i alguses | trace'i lõpus |
| Kulu | odav, kiire | kallim, vajab rohkem ressurssi |
| Näeb | otsus enne tulemust | terve trace enne otsust |
| Risk | võib probleemse trace'i vahele jätta | näeb kõik vead ja aeglased |

Head-based küsib "kas salvestame selle?", tail-based küsib "kas see trace oli huvitav?". Tail-based hoiab täpselt need jäljed, mida vaja — vead ja aeglased — aga maksab rohkem, sest peab kõik span'id otsuse hetkeni puhvris hoidma.

### RED-mõõdikud jälgedest

Span-metrics tähendab, et tracingu andmetest genereeritakse automaatselt RED-mõõdikud (Rate, Errors, Duration). Ühest instrumenteerimisest saad nii jäljed kui mõõdikud: rakendus → span'id → OTel Collector → trace'id Temposse, RED-mõõdikud Prometheusesse/Grafanasse. Sa ei pea RED-i eraldi instrumenteerima, kui jäljed on juba olemas.

### Kriitiline tee

Kõige aeglasem span ei pruugi olla probleem. Oluline on span, mis määrab päringu **kogukestuse** — kriitiline tee. Mõni span võib olla aeglane, aga jooksta paralleelselt ega lükka koguaega edasi. Jälg aitab kriitilise tee leida. Optimeeri mõju, mitte värvi: paranda seda, mis päriselt kogukestust määrab, mitte iga punast tulpa.

### Korrelatsioon: logid + trace_id

Kui logirida kannab `trace_id`, seob ta logi konkreetse jäljega:

```
2026-06-03 14:22:01 ERROR checkout failed
  trace_id=4bf92f3577b34da6 user=tier_gold
```

Nüüd liiguvad kolm vaadet kokku. Mõõdik ütleb, et probleem on olemas (p99 hüppas). Jälg näitab, kus probleem tekkis (Order Service 1,85 s kriitilisel teel). Logi ütleb, mis täpselt juhtus (deadlock detected, stacktrace). Logist hüppad jälge, jäljest logisse, trace ID seob kõik kokku: **metrics → trace → logs → root cause**.

### Kuhu jäljed lähevad

Tracing-backend'id lahendavad sama ülesannet — salvestavad, indekseerivad ja analüüsivad jälgi. Klassikalised open-source: Zipkin (vanim, lihtne), Jaeger (CNCF, levinud), Tempo (Grafana, objektisalvestus, LGTM osa). APM-platvormid oma backendiga: Elastic APM, Datadog, New Relic, Dynatrace, AppDynamics. Cloud-native: AWS X-Ray, Google Cloud Trace, Azure Application Insights.

---

## 4. OpenTelemetry

### OTel kui liim

OpenTelemetry on ühtne viis telemeetriat koguda ja edastada. Allikad saadavad andmed OTel Collectorisse, mis sämplib, rikastab ja marsruudib need edasi — jäljed Temposse, logid Lokisse, mõõdikud Mimirisse. Üks formaat sisse, mitu sihtkohta välja.

### SDK: auto vs manuaalne

**Auto-instrumenteerimine**: agent teeb töö sinu eest, HTTP, SQL ja gRPC leitakse automaatselt, kiire alustada. **Manuaalne**: lisad äriloogika ise — checkout, makse, tellimus — rohkem konteksti. Auto vastab küsimusele "mis süsteemis juhtus?", manuaalne "mida kasutaja tegi?". Praktikas alustad autoga ja lisad manuaalseid span'e seal, kus äriloogika on oluline.

### Collector

OTel Collector on üks vastuvõtupunkt: sämplimine ja rikastamine, andmete marsruutimine, mitu sihtkohta sama pipeline'iga. SDK-d, Kubernetes, compute ja API-d saadavad kõik OTLP-d collectorisse, sealt läheb edasi observability-platvormidele, salvestusele ja analüütikale.

### Instrumenteerimise järjekord

Kui hakkad OTel-i kasutusele võtma, siis sellises järjekorras: kõigepealt auto-instrumenteeri teenused (HTTP, SQL, gRPC, messaging), siis lisa `trace_id` logidesse (et logid ja jäljed kokku saaks), siis lisa käsitsi ärilised span'id (checkout, makse), ja sea sämplimine varakult, et kulud kontrolli all hoida. Sämplimist hiljem lisada on valus, sest seni oled kogunud rohkem, kui vaja.

### Üks instrumenteerimine, mitu sihti

Ilma OTel-ita on igal tööriistal oma agent, oma formaat, vendor lock-in, ja migratsioon tähendab koodimuudatusi. OTel-iga instrumenteerid üks kord, telemeetria formaat on standardne, backend on vaba valik, ja backendi vahetus ei nõua koodimuudatust. Instrumenteeri üks kord, vali backend hiljem.

### OTel ja Zabbix

Zabbix jääb infra- ja teenusemonitooringu kihiks. OTel katab tracingu, APM-i ja kõrge kardinaalsusega telemeetria. Zabbix näitab, et midagi on aeglane; OTel aitab leida, miks. Praktikas: Zabbix annab häire, seejärel vaatad Grafana dashboardi, kontrollid CPU-d ja mälu, vaatad andmebaasi mõõdikuid, otsid logidest vigu, võrdled kellaaegu ja vajadusel kaasad arendaja. OTel täiendab Zabbixit, ei asenda.

!!! note "OTel praktilised väljakutsed"
    Collector on uus komponent, mida tuleb ise monitoorida. Sämplimine on teadlik kompromiss kulude ja nähtavuse vahel. Logide ökosüsteem areneb kiiresti. Auto-instrumenteerimine ei kata kogu äriloogikat. Ja instrumenteerimine vajab hooldust ning standardeid — see pole "pane peale ja unusta".

---

## 5. Arhitektuurid

### Jälgitavuse kihid

Kogu pilt mahub kuude kihti, alt üles: **süsteemid** (serverid, võrk, Kubernetes, rakendused) → **agendid ja kogumine** (Zabbix Agent/SNMP, node_exporter, logishipper, OTel SDK) → **pipeline** (OTel Collector, vajadusel Kafka) → **backend'id** (mõõdikud Prometheus/Mimir/TimescaleDB, logid Loki/OpenSearch, jäljed Tempo/Jaeger) → **visualiseerimine** (Grafana, Zabbix UI, Kibana/OpenSearch Dashboards) → **äri** (SLA/SLO raportid, äridashboardid). Iga kiht teeb ühte tööd ja annab järgmisele edasi.

### Kus telemeetria kogutakse

| Muster | Kus | Milleks |
|---|---|---|
| Agent | server/VM | hosti mõõdikud |
| Sidecar | Kubernetes pod | rakenduse lähedal kogumine |
| Gateway | keskne sissepääs | agregeerimine, filtreerimine |
| Collector | pipeline | töötlus ja edastamine |

Andmeid võib koguda mitmes kohas, aga OTel Collector on tavaliselt keskne kogumis- ja töötluspunkt.

### Tsentraalne vs föderaalne

**Tsentraalne**: üks server kogub kõik — lihtne, aga pudelikael, sobib väikesele/keskmisele keskkonnale. **Föderaalne**: kohalikud kogujad pluss ülemine kiht — skaleerub, keerukam, sobib suurele ja mitme asukohaga keskkonnale. Zabbix proxy on föderaalse kogumise klassika: te teete seda juba, kui proxyd koguvad eemalt ja edastavad serverisse.

### Salvestamine andmetüübi järgi

Erinevad andmetüübid nõuavad erinevat salvestusstrateegiat. Mõõdikud lähevad time series -andmebaasi (Prometheus, Mimir, InfluxDB) — optimeeritud kiireks kirjutamiseks ja agregeerimiseks. Logid lähevad logiindeksisse (OpenSearch, Elastic) — täistekstotsing suurel mahul. Jäljed lähevad objektisalvestusse (Tempo, Loki taga S3/MinIO/Azure Blob/GCS) — odav suure mahu jaoks. Zabbixi enda ajaseeria läheb PostgreSQL + TimescaleDB hüpertabelitesse: automaatne ajapõhine partitsioneerimine, kompressioon kuni ~90%, chunk-retention vanade andmete kustutamiseks.

### Kafka kui puhver

Kafka neelab tippkoormuse, kui backend ei jõua. Producer'id (agendid, collectorid) ei pea teadma backendi elusolekut. Kui backend on lühiajaliselt maas, ei kao andmed — nad järjekordistatakse Kafkasse ja ootavad. See lahutab kogumise ja salvestamise üksteisest, nii et üks aeglane backend ei suru kogu pipeline'i kinni.

### HA ja andmevoog

Single point of failure on jälgitavuse vaenlane — eriti, sest jälgitavussüsteem peab töötama just siis, kui muu on katki. Zabbix serveril on HA (node-cluster, `ha_node`), proxydel HA (proxy-grupid, mitu teed), collector ja backend dubleeritud mitmes tsoonis. Andmevoog ei tohi sõltuda ühest sõlmest ega ühest ühendusest.

### Full-stack: Zabbix + LGTM + OTel

Mitte kas-kas, vaid kumb mille jaoks. Üks katus (Grafana) neljal sambal: Zabbix DB/Timescale (infra-mõõdikud, probleemid, triggerid), Mimir (mõõdikud skaalas), Loki (logid), Tempo (jäljed). All agendid ja kogujad: Zabbix Agent serveritele, node_exporter, OTel Collector rakendustele ja K8s-le.

---

## 6. Integratsioon ja full-stack

### Zabbix + Grafana

Grafana ühendab Zabbixi ja ülejäänud stacki üheks vaateks: Zabbixi mõõdikud ja probleemid, Prometheuse mõõdikud, Loki logid, Tempo jäljed — kõik samal dashboardil. See on koht, kus eile õpitud teenusvaade ja tänane telemeetria kokku saavad.

### LGTM stack

Grafana ökosüsteem kolmele sambale: **L**oki (logid), **G**rafana (visualiseerimine), **T**empo (jäljed), **M**imir (mõõdikud skaalas). Üks ökosüsteem, Grafana liidesena. Kui Zabbix on teil tugev infra-pool, siis LGTM lisab app-poole sama katuse alla.

### Prometheus ↔ Zabbix mõlemas suunas

Zabbix → Prometheus: Zabbixi andmed Grafanasse datasource'i või exporti kaudu, ühtne vaade. Prometheus → Zabbix: Zabbix scrapib ise `/metrics` endpoint'i (HTTP agent + preprocessing), ja app-metrics tulevad sisse Zabbixi itemitena. Paljud ei tea, et Zabbix oskab Prometheuse endpoint'i ise scrape'ida — see avab tee app-mõõdikuteni ilma eraldi Prometheuse-serverit püstitamata.

### Keskne logimine: Loki vs OpenSearch

**Loki** indekseerib ainult silte — odav, väike maht, Grafana-native. **OpenSearch/Elastic** indekseerib kogu sisu — võimas otsing, suurem kulu, Kibana/OS Dashboards. Loki, kui maht peab olema odav; OpenSearch, kui vajad sügavat otsingut ja analüütikat. See on sama kompromiss, mille kohtad eile mainitud kardinaalsuse juures: mida rohkem indekseerid, seda kallim.

### Zabbix Cloud

Zabbixi-hallatav SaaS (AWS/Azure/GCP regioonid). Vähem taristuhaldust, kiire start, hea kui pole oma DB/HA hooldamise ressurssi. Ei sobi, kui on range andmeasukoht või sügav on-prem integratsioon.

### Hübriidarhitektuur ja otsustusraamistik

Reaalne keskkond on tavaliselt hübriid: on-prem (Zabbix Proxy + OTel Collector HA-grupis), cloud (OTel Collector AWS/Azure/GCP juures), ja Zabbix Cloud. Kõik voolab läbi Kafka puhvri dubleeritud backendidesse, Grafana koondab. Asukoht ei tohiks jälgitavust mõjutada.

Küsimus pole "milline tööriist on parim", vaid milline kombinatsioon lahendab teie probleemi kõige lihtsamalt ja väiksema kuluga.

---

## 7. Turve

### Krüpteerimine: PSK vs TLS

**PSK** (pre-shared key): jagatud salajane võti, lihtne, väike keskkond, identity ja key ühes. **Sertifikaadipõhine TLS**: CA-allkirjastatud sertifikaadid, skaleerub, tsentraalne usaldus; mTLS-iga on mõlemad pooled tõendatud. Zabbixis on agent ↔ server ↔ proxy kõik krüpteeritavad — väikeses keskkonnas PSK, kasvades sertifikaadid.

### mTLS ja sertifikaatide haldus

OTLP käib üle TLS, collector ↔ backend mTLS. Sertide elutsükkel — väljastamine, uuendamine, tühistamine — on omaette töö. Aegunud sert annab vaikse katkestuse, mis on jälgitavuse jaoks eriti ohtlik: süsteem, mis peaks sind hoiatama, lakkab vaikselt andmeid saamast, ja sa ei pruugi seda märgata enne, kui vajad andmeid kõige rohkem.

### Audit log

Zabbixi audit log salvestab konfiguratsiooni- ja autentimismuudatused. Küsimus "kes selle triggeri eile öösel keelas?" peab olema vastatav. See pole valikuline panga, NIS2, ISO ja siseauditi kontekstis.

### Autentimine

LDAP/AD ettevõtte kasutajatele, SAML/SSO ühe sisselogimise jaoks, MFA teise tegurina adminidele. Internal-autentimine jääb väikese keskkonna varuks või hädalahenduseks, mitte peamiseks viisiks.

### RBAC — õigused skaalal

Kolm tasandit. **User roles** (7.0+) määravad, mida tohib teha — lugemine, kirjutamine, seadistamine, administreerimine, ainult käivitamine — granulaarselt funktsioonidele ja objektidele. **User groups** seovad kasutajad rollide komplekti. **Host groups** määravad, millistele objektidele (ja alamgruppidele) õigus kehtib.

Ahel: kasutaja kuulub gruppi → grupil on rollid → rollid annavad õigused → host group määrab ulatuse → õigused rakenduvad konkreetsetele hostidele ja mõõdikutele. Õige inimene, õiged õigused, õige ulatus. Suure halduskoormuse juures on see see, mis hoiab seitsekümmend administraatorit üksteise jalus astumast.

### API ja secrets

Kolm rolli, mida segi ei aeta. **Secret macro** on saladus, mida ei näidata — väärtus pole loetav, ei logita, ei paista UI-s; `{$DB_PASS}` võib viidata Vaultile. **API token** on see, mida automaatika tohib teha — scoped, piiratud õiguse ja elueaga. **Vault** on koht, kus saladusi hallatakse. Lühidalt: API token = mida tohib teha, secret macro = mida ei näidata, Vault = kus hoitakse.

---

## 8. Intsidendid, postmortem ja automatiseerimine

### Intsident algusest lõpuni

SRE vaatab intsidenti kindla ahelana. SLI rikub eesmärki (checkout p99 ületab sihttaset) → alert käivitub (SLO või alerting policy avab intsidendi) → mõju hinnatakse (kui palju kasutajaid, kui kiiresti veaeelarve kulub) → põhjus leitakse (jäljed, mõõdikud ja logid, mitte oletused) → teenus taastatakse (MTTR minimeeritakse) ja postmortemis õpitakse. **SLI → Alert → Mõju → Juurpõhjus → Taastamine.**

### Postmortem

Timeline tuleb mõõdikutest — millal algas, millal taastati. Root cause tuleb jälgedest ja logidest, mitte oletustest. Parandused puudutavad alerting'ut, SLO-sid, automaatikat ja vajadusel koodi. Kultuur on blameless: eesmärk on õppimine, mitte süüdlase otsimine. Intsidendi eesmärk on teenus taastada; postmortemi eesmärk on vältida sama intsidenti tulevikus.

### Observability → Ansible-remediatsioon

Automaatika, mis reageerib andmetele. Alert/event (Zabbix, Prometheus, Loki) → event-driven reegel valib sobiva playbook'i → Git hoiab versioonitud playbook'e (`restart_service.yml`, `collect_diagnostics.yml`, `scale_out.yml`) → Ansible Controller/AWX käivitab → mõõdikud ja logid kontrollivad tulemust. Kui SLI taastus, suletakse intsident ja logitakse tulemus; kui ei taastunud, eskaleeritakse või käivitatakse teine playbook. Observability avastab probleemi, Git hoiab paranduse loogikat, Ansible käivitab tegevuse, mõõdikud kontrollivad tulemust.

### Mida automatiseerida, mida mitte

Tee automaatselt: korduvad madala riskiga remediatsioonid (restart, logi puhastus), standardiseeritud diagnostika (kogu logid, system info). Väldi täielikku automaatikat: kõrge riskiga operatsioonid, keerukad migratsioonid, olukorrad, kus äririski peab hindama inimene.

Sama kehtib konfiguratsiooni kohta. Zabbixi konfiguratsioon Gitis (JSON/YAML) → Ansible deploy → Zabbix API → templates, host groups, triggers, users, dashboards. Tulemus: kõik muudatused on versioonitud, auditeeritavad ja korduvad. See on eile mainitud toili-kaotus, viidud konfiguratsiooni tasandile.

---

## 9. Kokkuvõte — Zabbixist jälgitavuseni

Tee Zabbixist täisjälgitavuseni on viiest sammust:

**1. Tugev Zabbix-baas.** Templates, LLD, BSM, SLA — see on teil juba olemas ja jääb infra-kihi vundamendiks.

**2. SLI/SLO pluss percentiilid mõõdikutele.** Mõõda saba (p95/p99), mitte keskmist; mõõda kasutajakogemust, mitte ainult masinat.

**3. Logid Zabbixi kõrvale.** Loki odava mahu jaoks, OpenSearch sügava otsingu jaoks.

**4. App-traces OTel-iga ja korrelatsioon.** Üks instrumenteerimine, mitu backendi; `trace_id` seob mõõdikud, logid ja jäljed üheks intsidendiks.

**5. Event-driven automatiseerimine ja SLO-juhtimine.** Observability avastab, Git hoiab loogikat, Ansible käivitab, mõõdikud kontrollivad; postmortem õpib.

Mõni täiendav punkt, mis kursusest kaasa võtta:

**Zabbix ei ole APM.** Ta katab infra ja synthetic; jaotatud tracing on OTel + tracing-backend.

**Item key vs sildid.** Zabbix paneb dimensioonid key'sse, Prometheus/OTel siltidesse — kardinaalsus on mõlemal piirang.

**Kõrge kardinaalsus kuulub jälgedesse, mitte mõõdikutesse.** Span-atribuudis on `user_id` õige; mõõdikusildis tapab ta andmebaasi.

**Single point of failure on jälgitavuse vaenlane.** HA igal kihil, sest jälgitavus peab töötama just siis, kui muu on katki.

**Küsimus pole "parim tööriist", vaid "õige kombinatsioon".** Kõige lihtsam ja väiksema kuluga lahendus teie probleemile.

---

## Allikad

### Ametlik dokumentatsioon
| Allikas | URL |
|---|---|
| Zabbix events | https://www.zabbix.com/documentation/current/en/manual/config/events |
| Zabbix item key | https://www.zabbix.com/documentation/current/en/manual/config/items/item/key |
| Zabbix encryption | https://www.zabbix.com/documentation/current/en/manual/encryption |
| Zabbix Cloud | https://www.zabbix.com/cloud |
| OpenTelemetry docs | https://opentelemetry.io/docs/ |
| OpenTelemetry Collector | https://opentelemetry.io/docs/collector/ |
| W3C Trace Context | https://www.w3.org/TR/trace-context/ |
| Prometheus histograms | https://prometheus.io/docs/practices/histograms/ |

### Teooria ja kontekst
| Allikas | URL |
|---|---|
| Google SRE | https://sre.google/ |
| TimescaleDB | https://www.timescale.com/ |

### Praktiline
| Allikas | URL |
|---|---|
| Grafana Tempo (jäljed) | https://grafana.com/docs/tempo/latest/ |
| Grafana Loki (logid) | https://grafana.com/docs/loki/latest/ |

---

*Järgmine: praktikum — paneme kihid kokku oma keskkonnas.*