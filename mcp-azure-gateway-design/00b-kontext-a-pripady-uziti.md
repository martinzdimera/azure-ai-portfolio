# 00b — Kontext a případy užití: proč MCP-Azure Gateway (doplněno po nezávislém srovnání návrhu)

**Verze:** 1.0 · **Datum:** 2026-09-17 · **Stav:** Návrh
**Cílový čtenář:** sponzor projektu, vedení, byznysoví stakeholdeři — dokument záměrně nepoužívá technický žargon nad rámec toho nejnutnějšího a vysvětluje ho tam, kde se mu nelze vyhnout.

Tento dokument doplňuje mezeru, kterou zbytek architektonické dokumentace (dokumenty `01`–`09`) neřeší — proč organizace tuto bránu vůbec staví, k čemu ji konkrétní lidé ve firmě skutečně použijí, a co se stane, pokud se nepostaví. Zbytek dokumentace se soustředí na to, *jak* je gateway postavená; tento dokument na to, *proč* na tom záleží někomu mimo technický tým.

---

## 1. Odkud problém přichází

Zaměstnanci organizace čím dál víc používají konverzační AI asistenty — Claude i ChatGPT — jako běžný nástroj pro každodenní práci: hledání informací, sepisování textů, analýzu dat, plánování. Tito asistenti se navíc rychle vyvíjejí od „chatbotu, který odpovídá na otázky" k „agentovi, který umí sám vykonávat kroky" — nikoliv jen radit, jak zjistit stav faktury, ale fakturu skutečně vyhledat; nikoliv jen navrhnout text incidentu, ale incident rovnou založit v systému.

Tato schopnost je adresovaná mechanismem jménem **MCP** (Model Context Protocol) — standardizovaným způsobem, jak AI asistent „vidí" a volá nástroje, které mu organizace zpřístupní. Jak Claude, tak ChatGPT tento mechanismus podporují a umožňují administrátorovi organizace připojit vlastní sadu nástrojů, které se pak objeví přímo v konverzaci uživatele jako přirozená součást asistenta.

Problém je, že tyto nástroje musí něco reálně dělat — číst zákaznická data z CRM, hledat faktury ve finančním systému, zakládat tikety v IT service desku, ověřovat stav žádosti o dovolenou v HR systému. Všechny tyto systémy dnes organizace už má a chrání je vlastními přístupovými pravidly. Otázka, kterou tento projekt řeší, zní: **kdo a jak smí AI asistentovi dovolit, aby jménem konkrétního zaměstnance do těchto systémů sáhl — a jak si být jistí, že sáhne jen tam, kam ten zaměstnanec sám smí, a že se o tom vede spolehlivý záznam?**

MCP-Azure Gateway je odpověď na tuto otázku: jedno centrální, spravované místo, přes které vede *veškerý* přístup AI asistentů (Claude i ChatGPT současně) k interním systémům organizace. Ne desítky ad-hoc integrací budovaných nezávisle jednotlivými týmy, ne přímé sdílení přístupových údajů s AI nástroji, ale jedna brána s jednotnou identitou, jednotným oprávněním a jednotným auditem.

## 2. Jaký problém gateway konkrétně řeší

Bez centrální brány má organizace v zásadě tři špatné možnosti, jakmile zaměstnanci začnou chtít propojit AI asistenty s interními systémy:

1. **Nic nepovolit.** AI asistenti zůstanou omezení na to, co uživatel sám zkopíruje do konverzace — ztrácí se většina produktivní hodnoty, kterou agentní schopnosti nabízejí, a zaměstnanci si najdou cestu kolem zákazu (viz kapitola 6 níže).
2. **Nechat každý tým budovat vlastní integraci.** Finanční tým si udělá vlastní propojení na fakturační systém, IT tým vlastní na service desk, obchodní tým vlastní na CRM — bez jednotného modelu oprávnění, bez jednotného auditu, s duplicitní prací a nekonzistentní úrovní zabezpečení mezi jednotlivými integracemi. Bezpečnostní tým nemá jedno místo, kde by mohl dohledat „kdo se AI asistentem ptal na co", protože každá integrace loguje jinak, nebo neloguje vůbec.
3. **Sdílet přístupové údaje přímo s AI nástrojem bez zprostředkující vrstvy.** Nejrizikovější varianta — ztrácí se rozdíl mezi tím, co smí AI asistent jménem konkrétního uživatele, a co by mohl technicky udělat s plným přístupem, který mu byl přidělen.

Gateway řeší tento problém tím, že je **jediné místo**, kudy provoz prochází, s vlastnostmi, které žádná z ad-hoc variant výše nemá zadarmo: ověří, kdo se ptá (ne jen že se ptá „nějaký AI asistent"), rozhodne, jestli tento konkrétní člověk smí zavolat tento konkrétní nástroj s těmito konkrétními parametry, a zapíše nesmazatelný záznam o tom, co se stalo — ať dopadlo volání úspěchem, nebo bylo zamítnuto.

## 3. Mapa případů užití

Následující příklady jsou reprezentativním výřezem z očekávaného rozsahu — řádově desítek nástrojů napříč doménami CRM, financí, obchodu, HR, IT service desku a interní znalostní báze, pro řádově stovky uživatelů do konce plného nasazení (viz [`01-analyza-zadani-a-plan.md`](./01-analyza-zadani-a-plan.md) §3, předpoklady A09–A10). Každý příklad popisuje, kdo situaci zažívá, co ji spustí, jak probíhá, jakou hodnotu přináší, a jaký druh nástroje na pozadí potřebuje.

**1. Obchodní zástupce se připravuje na schůzku se zákazníkem.** Před telefonátem se v konverzaci s asistentem zeptá na historii účtu — poslední objednávky, otevřené reklamace, kontaktní osoby. Asistent zavolá čtecí nástroj nad CRM a odpověď sestaví do přehledného shrnutí. Hodnota: minuty místo přepínání mezi třemi systémy před každým hovorem. Nástroj: čtení dat účtu z CRM, omezené na účty, ke kterým má daný obchodník přiřazený přístup.

**2. Finanční analytik kontroluje stav neuhrazených faktur.** Na konci týdne se zeptá, které faktury konkrétního zákazníka jsou po splatnosti, a nechá si od asistenta připravit shrnutí pro upomínkový e-mail. Hodnota: rutinní kontrola, která dřív vyžadovala export a filtrování v samostatném systému, proběhne jako součást běžné konverzace. Nástroj: čtení seznamu faktur podle zákazníka a stavu.

**3. Zaměstnanec IT podpory zakládá tiket na základě popisu problému uživatele.** Uživatel popíše asistentovi technický problém, asistent shrne popis do strukturovaného tiketu a s potvrzením uživatele ho založí v systému service desku. Hodnota: rychlejší a konzistentnější zakládání tiketů, méně chybějících informací v prvním kroku. Nástroj: zápisový nástroj zakládající incident — vyžaduje explicitní potvrzení před provedením, protože jde o zápisovou operaci s dopadem na frontu podpory.

**4. Schvalující manažer potvrzuje platbu faktury.** Manažer se zeptá na faktury čekající na jeho schválení, prohlédne si detail a přes konverzaci fakturu schválí — asistent před samotným provedením výslovně vyžádá potvrzení, protože jde o finančně citlivou, těžko vratnou akci. Hodnota: schvalování bez nutnosti přepínat se do samostatné firemní aplikace. Nástroj: zápisový nástroj s vysokou úrovní rizika, vynuceným potvrzením a přísnějším oprávněním než čtecí nástroje výše.

**5. Zaměstnanec HR ověřuje stav žádosti o dovolenou.** Zaměstnanec se zeptá, jestli už byla jeho žádost schválená. Asistent zavolá čtecí nástroj nad HR systémem, omezený jen na data samotného tazatele (ne kolegů) — citlivá osobní data, u kterých je rozsah přístupu obzvlášť přísně vymezený. Hodnota: rychlá odpověď na běžný dotaz bez zatěžování HR oddělení. Nástroj: čtení vlastních HR záznamů, s nejužším možným rozsahem dat na osobu, která se ptá.

**6. Nový zaměstnanec prochází nástupní proces.** V prvním týdnu se ptá asistenta na kroky onboardingu, potřebné formuláře a kontakty — asistent kombinuje odpovědi z interní znalostní báze s kontrolou, jaké kroky už má nový zaměstnanec ve svém HR záznamu hotové. Hodnota: jedno místo pro otázky, které by jinak směřovaly na HR nebo IT podporu opakovaně napříč všemi novými zaměstnanci. Nástroje: čtení interní znalostní báze (veřejná v rámci organizace) a čtení vlastního HR záznamu.

**7. Obchodní analytik připravuje přehled napříč odděleními.** Pro čtvrtletní review se zeptá na zákazníky s otevřenými fakturami po splatnosti *a zároveň* otevřenými servisními incidenty — signál rizikových účtů, které potřebují pozornost. Asistent zavolá čtecí nástroje nad financemi a service deskem a výsledek propojí. Hodnota: přehled, který dřív vyžadoval ruční spojení exportů ze dvou různých systémů, vznikne v jedné konverzaci. Nástroje: čtení faktur po splatnosti a čtení otevřených incidentů, obě omezené na rozsah dat, ke kterým má analytik přístup.

**8. Bezpečnostní/compliance tým šetří podezřelou aktivitu.** Po nahlášení podezření, že se citlivá data mohla dostat mimo organizaci přes AI asistenta, bezpečnostní tým dohledá v nezměnitelném auditním záznamu gateway přesně to, jaký uživatel jaký nástroj s jakými parametry zavolal a s jakým výsledkem, včetně zamítnutých pokusů. Hodnota: bez gateway by takové šetření u ad-hoc integrací (varianta 2 v kapitole 2) buď nebylo možné vůbec, nebo by vyžadovalo rekonstrukci z nesourodých, nekonzistentních logů jednotlivých systémů. Zde slouží samotná existence gateway jako vyšetřovací nástroj, ne jednotlivé volání nástroje.

**9. Vlastník skillu v obchodním oddělení přidává nový firemní postup pro asistenta.** Doménový expert (ne vývojář) popíše nový firemní postup — např. jak správně sestavit nabídku — formou strukturovaného popisu, který projde recenzí a po schválení se objeví jako nová schopnost asistenta pro celou organizaci, bez nutnosti nasazovat nový kód gateway. Hodnota: rozšiřování schopností asistenta je řízený proces s recenzí, ne neřízené přidávání nekontrolovaného kódu, a nevyžaduje zapojení vývojového týmu pro každou drobnou změnu.

Napříč příklady výše je vidět opakující se vzorec: čtecí operace (1, 2, 5, 6, 7) mají nižší riziko a jednodušší schvalovací cestu, zatímco zápisové operace (3, 4) vyžadují explicitní potvrzení a přísnější oprávnění — to není náhoda, ale přímý důsledek bezpečnostního modelu popsaného v [`07-bezpecnost-a-autorizace.md`](./07-bezpecnost-a-autorizace.md), který tento dokument nechce opakovat technicky, jen ukázat, jak se projevuje v běžné práci.

## 4. Přínosy a jak se budou měřit

Přínos gateway se dělí na dvě roviny, které je užitečné rozlišovat, protože se měří jinak.

**Přímý produktivní přínos** — čas ušetřený tím, že zaměstnanci nemusí přepínat mezi systémy a AI asistent za ně vykoná rutinní kroky přímo. Měří se nepřímo přes adopci: podíl aktivních uživatelů z cílové skupiny, kteří nástroj skutečně použili alespoň jednou týdně (sledováno od pilotní fáze, viz [`08-provoz-observabilita-rollout.md`](./08-provoz-observabilita-rollout.md) §4, Dashboard 2 — adopce a využití), a trend počtu volání jednotlivých nástrojů v čase. Cílovou hodnotu podílu aktivních uživatelů pilotní skupiny musí stanovit sponzor projektu — v tuto chvíli je to otevřená otázka (viz exit kritéria Fáze 1 v `08` §9).

**Přínos v podobě rizika, kterému se organizace vyhne** — hůř měřitelný přímo, protože jde o prevenci něčeho, co se díky gateway nestane. Zde jsou nejlepším proxy ukazatelem: (a) počet a povaha zamítnutých volání nástrojů (Dashboard 3 — bezpečnost a odmítnutí, `08` §4) jako signál, že autorizační vrstva skutečně něco zachytává, ne že je jen formalitou; (b) úplnost a ověřitelnost auditní stopy při případném bezpečnostním šetření — zda šetření (viz případ užití 8 výše) skutečně proběhne rychle a spolehlivě, až na to dojde; (c) absence bezpečnostních incidentů vázaných na neoprávněný přístup přes AI asistenta, sledováno jako součást standardního bezpečnostního reportingu organizace.

Obě roviny se promítají do byznysového zdůvodnění projektu odlišně — první ukazuje návratnost investice v podobě ušetřeného času, druhá v podobě sníženého rizika, kterému se nedá snadno přiřadit jednotné číslo dřív, než se realizuje (nebo nerealizuje).

## 5. Zainteresované strany a jejich očekávání

| Strana | Co od gateway očekává | Kde v dokumentaci najde odpověď |
|---|---|---|
| Sponzor projektu / vedení | Viditelný přínos pro produktivitu, kontrolovatelné riziko, jasný postup nasazení bez skokového rizika | Tento dokument, [`01-analyza-zadani-a-plan.md`](./01-analyza-zadani-a-plan.md) §6–§8 |
| Koncoví uživatelé (obchod, finance, HR, IT podpora) | Aby asistent skutečně uměl to, co mají dovoleno dělat, bez zbytečných překážek u běžných čtecích dotazů | Případy užití výše; [`08-provoz-observabilita-rollout.md`](./08-provoz-observabilita-rollout.md) §10 konfigurace připojení |
| Bezpečnostní a compliance tým | Že žádný přístup neobejde autorizaci, že existuje nesmazatelný a ověřitelný záznam každého volání, že zápisové/rizikové operace vyžadují explicitní potvrzení | [`07-bezpecnost-a-autorizace.md`](./07-bezpecnost-a-autorizace.md) celý dokument |
| Vlastníci downstream systémů (CRM, finance, HR, ITSM) | Že gateway nepřidává nekontrolovaný provoz nad jejich API, respektuje existující rate limity a přístupová pravidla, a že mají viditelnost do toho, co se přes gateway děje s jejich daty | [`02-architektura-reseni.md`](./02-architektura-reseni.md), [`05-api-a-protokoly.md`](./05-api-a-protokoly.md) §7 downstream kontrakt |
| Interní IT (provozní tým) | Zvládnutelný provoz s runbooky a jasnou eskalační cestou, bez nutnosti hlubokých znalostí AI/MCP specifik | [`08-provoz-observabilita-rollout.md`](./08-provoz-observabilita-rollout.md) §6 a §11 handover |
| Vývojový/platformní tým | Jasně definovaná hranice mezi tím, co je konfigurace, a co katalog nástrojů, aby přidávání nových schopností nevyžadovalo pokaždé nasazení nového kódu | [`03-kodova-architektura.md`](./03-kodova-architektura.md) §9.3 |

Očekávání jednotlivých stran se místy dostávají do napětí — nejviditelněji mezi koncovými uživateli (chtějí co nejméně překážek) a bezpečnostním týmem (chce co nejpřísnější kontrolu u rizikových operací). Návrh toto napětí neřeší tím, že by stranu jedné straně upřednostnil plošně, ale rozlišením podle rizika konkrétní operace (viz `riskTier` v [`07-bezpecnost-a-autorizace.md`](./07-bezpecnost-a-autorizace.md)) — čtecí dotazy s nízkým rizikem mají minimální tření, zápisové a citlivé operace mají tření vyšší a záměrné.

## 6. Co se stane, když se gateway nepostaví

Poptávka po propojení AI asistentů s interními systémy od zaměstnanců nezmizí jen proto, že jí organizace formálně nevyhoví — spíš se přesune mimo dohled. To je scénář **shadow IT**: jednotlivci nebo týmy si najdou vlastní, neschválenou cestu, jak asistentovi zpřístupnit potřebná data, protože produktivní tlak je reálný a AI asistenty dnes běžně používají i bez firemního schválení.

V praxi to znamená kombinace, kterou už bezpečnostní model tohoto návrhu identifikuje jako jedno z hlavních rizik (viz [`07-bezpecnost-a-autorizace.md`](./07-bezpecnost-a-autorizace.md) §2, hrozba „exfiltrace dat kombinací nástrojů"): zaměstnanec zkopíruje citlivá data ze zákaznického systému, HR záznamu nebo finančního reportu přímo do konverzace s veřejně dostupnou verzí AI asistenta — bez jakéhokoli záznamu o tom, že k tomu došlo, bez kontroly nad tím, kam data ve skutečnosti šla (u některých veřejných nasazení mohou být data použita jinak, než zaměstnanec předpokládá), a bez možnosti tento tok zpětně omezit nebo prošetřit. Neexistuje audit, protože neexistuje jednotné místo, kudy by provoz musel projít — každý si integraci nebo obchvat řeší po svém.

Důsledek pro organizaci je dvojí. Za prvé, **ztráta kontroly nad daty** — organizace nemá odpověď na jednoduchou otázku „která citlivá data byla v posledním čtvrtletí sdílena s externím AI nástrojem a kým", protože takový přenos neprochází žádným bodem, který by ho zaznamenal. Za druhé, **rozptýlené, nekonzistentní riziko místo jednoho spravovaného rizika** — bez centrální brány se bezpečnostní expozice neeliminuje, jen se stane neviditelnou a nekoordinovanou, roztroušenou napříč tolika ad-hoc řešeními, kolik má organizace týmů ochotných problém vyřešit po svém. To je z bezpečnostního hlediska horší stav, ne lepší — spravovaná brána s definovaným, i nedokonalým, oprávněním a auditem je kontrolovatelné riziko; nekontrolované shadow IT napříč desítkami týmů je riziko, o kterém organizace ani neví, že ho nese, dokud se nestane incidentem.

Gateway sama o sobě negarantuje, že k shadow IT nikdy nedojde — nejde o technické opatření, které by fyzicky zabránilo zaměstnanci zkopírovat text do jiné aplikace. Co ale mění, je poměr pohodlí mezi schválenou a neschválenou cestou: pokud je oficiální cesta (přes gateway) rychlá, spolehlivá a pokrývá skutečné potřeby zaměstnanců popsané v mapě případů užití výše, klesá motivace hledat obchvat. To je důvod, proč fázovaný rollout (viz [`08-provoz-observabilita-rollout.md`](./08-provoz-observabilita-rollout.md) §9) záměrně začíná co nejdřívějším ověřením, že gateway se skutečnými klienty vůbec funguje (Fáze −1), a proč pilotní fáze aktivně sbírá zpětnou vazbu o tom, co uživatelům v nabídce nástrojů chybí — cíl není jen „postavit bránu", ale postavit bránu, kterou lidé skutečně chtějí použít místo obchvatu.
