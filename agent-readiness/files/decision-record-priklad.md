# Decision record pro AI agenta v provozu: PŘÍKLAD

**Tohle je PŘÍKLAD.** Firma, služby, jména, čísla i lhůty jsou smyšlené a slouží jen pro ilustraci. Převezměte formu záznamu, hodnoty si stanovte sami podle svého provozu.

Decision record je jednostránkový záznam o tom, že agent smí na konkrétním místě jednat na konkrétní úrovni autonomie. Je v něm, kdo to rozhodl, s jakými nástroji a právy agent pracuje, co se měří a do kdy povolení platí. Když se za půl roku někdo zeptá, proč agent v noci restartoval službu, odpovědí je tenhle papír se jménem, datem a důvodem.

Zakládejte ho pro každé místo, kde agent jedná se schválením nebo sám (U2 až U4), při každém zvýšení úrovně a pokaždé, když agent dostane nový nástroj nebo nová práva.

## Vyplněný záznam: automatický restart služby

V příkladu tým měsíc provozoval agenta na U2. Agent při alertu na paměť navrhl restart a on-call ho potvrdil nebo zamítl. Po měsíci tým rozhodl, že restart dvou konkrétních služeb smí agent dělat sám (U3).

```
PŘÍKLAD, hodnoty jsou ilustrační

ID záznamu: DR-2026-007
Datum: 4. 11. 2026

Místo nasazení:
Při alertu na zaplnění paměti u služeb orders-api a billing-api v produkci agent
posoudí, jestli restart pomůže, a pokud ano, službu restartuje.

Rozhodnutí:
U3, agent jedná sám v mezích. Jen pro restart dvou služeb uvedených níže.
Cokoli jiného agent jen navrhne do kanálu on-call (U2).

Kdo rozhodl:
Jana Horáková, vedoucí provozu

Vlastník agenta:
Lukáš Beneš, SRE

Agent a verze:
Model a jeho verze podle konfigurace agenta (zapsané v repozitáři ops-agents),
prompt verze 3 (tag restart-agent-v3). Spouští ho alert MemoryHigh přes webhook.

Nástroje a práva:
Čtení metrik a logů obou služeb: jen čtení.
Restart deploymentů orders-api a billing-api v namespace shop-prod: zápis.
Agent jedná pod servisním účtem sa-restart-agent, který nikdo jiný nepoužívá.

Uzavřený seznam akcí bez schválení:
1. Restart deploymentu orders-api.
2. Restart deploymentu billing-api.
Nejvýš jeden restart na službu za 30 minut. Druhý alert během hodiny už
agent neřeší restartem, jen eskaluje.

Čím jsou meze vynucené:
Role servisního účtu v clusteru povoluje jen restart těchto dvou deploymentů.
Guard před voláním nástrojů blokuje všechny ostatní nástroje.
Limit jednoho restartu za 30 minut hlídá webhooková brána, ne prompt.

Kdo schvaluje a kolik schválení za den:
Pro akce na seznamu neplatí, běží bez schválení. Návrhy mimo seznam
schvaluje on-call SRE, odhad 2 až 3 za týden.

Komu agent eskaluje:
On-call SRE podle rozpisu služeb, zpráva do kanálu #ops-noc s odkazem
na průběh agenta.

Vypínač:
On-call SRE nebo vlastník agenta vypne webhook pro alert MemoryHigh.
Postup je v runbooku RB-112. Vyzkoušeno nanečisto 28. 10. 2026.

Rollback:
Restart nemění konfiguraci ani data. Když služba po restartu nenaběhne
do 5 minut, agent nic dalšího nedělá a eskaluje. On-call postupuje
podle runbooku RB-112.

Co se měří a hranice:
Měsíc na U2 (1. až 31. 10. 2026): 46 návrhů na restart, on-call zamítl 3.
Zamítnuté případy jsou v testovací sadě spolu s dalšími 37 alerty
z minulých incidentů. Sada běží po každé změně promptu pětkrát.
Hranice, kterou si tým stanovil: podíl zamítnutých nebo chybných rozhodnutí
nejvýš 10 % za měsíc. Při překročení se agent vrací na U2.

Náklady na úlohu a strop:
Jedno posouzení alertu stojí v průměru kolem 3 Kč. Strop 20 spuštění
za hodinu ve webhookové bráně.

Co se z konverzací ukládá, kde a jak dlouho:
Prompty i odpovědi agenta se ukládají 30 dní v nástroji pro observabilitu
agentů. Rozhodla Jana Horáková 1. 10. 2026.

Kde je vidět, co agent udělal:
Průběh každého spuštění (volání modelu a nástrojů) v nástroji pro
observabilitu agentů. Restart v auditním logu clusteru pod účtem
sa-restart-agent. Zpráva do #ops-noc s odkazem na průběh.

Platnost:
Do 31. 1. 2027. Neplatí dřív, pokud agent dostane nový nástroj nebo nová
práva, pokud se změní model nebo prompt, pokud se má rozšířit na další
službu nebo zvýšit úroveň a po každém incidentu, který agent způsobil.

Potvrzeno:
Jana Horáková, 4. 11. 2026
```

## Na co se v příkladu dívat

Pole Platnost je nejdůležitější. Bez data revize a bez seznamu změn, které záznam zneplatní, povolení nikdo neodvolá a agent dostane nový nástroj bez nového posouzení.

Pole Čím jsou meze vynucené popisuje oprávnění v clusteru, guard a limit v bráně. Instrukce v promptu („restartuj jen tyhle dvě služby“) sem nepatří, model ji může porušit.

Pole Co se měří a hranice ukazuje cestu z U2 na U3. Tým nejdřív měsíc měřil podíl zamítnutých návrhů a teprve pak rozhodl. Hranici 10 % si v příkladu stanovil tým sám, žádná norma ji nepředepisuje.

## Prázdný formulář

```
ID záznamu:
Datum:

Místo nasazení:
(co agent dělá, jednou až dvěma větami)

Rozhodnutí:
(úroveň U1 až U4, kterou agent na tomto místě smí mít)

Kdo rozhodl:

Vlastník agenta:

Agent a verze:
(model, verze promptu nebo agenta, odkud se spouští)

Nástroje a práva:
(seznam nástrojů, u každého čtení nebo zápis, účet, pod kterým agent jedná)

Uzavřený seznam akcí bez schválení:
(jen U3 a U4)

Čím jsou meze vynucené:
(práva v cílovém systému, guard, seznam povolených nástrojů)

Kdo schvaluje a kolik schválení za den:
(jen U2)

Komu agent eskaluje:

Vypínač:
(kdo ho smí použít a jak)

Rollback:
(jak se vrací každá akce)

Co se měří a hranice:
(evaluace, sada případů, hranice, kterou si stanovil tým)

Náklady na úlohu a strop:

Co se z konverzací ukládá, kde a jak dlouho:

Kde je vidět, co agent udělal:

Platnost:
(do kdy, a že neplatí při novém nástroji, nových právech, změně modelu nebo promptu, zvýšení úrovně a po incidentu způsobeném agentem)

Potvrzeno:
```

## Kde záznamy držet

Tam, kde jde doložit, že se záznam zpětně neupravoval: v repozitáři v Gitu vedle konfigurace agenta, ve sdíleném úložišti s historií verzí nebo v dokumentovém systému. Když agent chodí přes pull requesty, může být decision record uložený ve stejném repozitáři.

Záznam zachycuje rozhodnutí týmu a jeho podmínky. O bezpečnosti agenta nic netvrdí a audit nenahrazuje.
