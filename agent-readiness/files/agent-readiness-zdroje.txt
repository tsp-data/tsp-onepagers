# Zdroje: AI Agent Readiness Advisor

Datum ověření celé rešerše: 3. listopadu 2026.

Tento dokument shromažďuje ověřená zjištění z primárních zdrojů, na která se odkazuje průvodce AI Agent Readiness Advisor: dokumentace a oznámení Grafana Labs, oficiální repozitáře na GitHubu, specifikace OpenTelemetry, původní odborné práce o úrovních autonomie agentů, bezpečnostní rámce OWASP a NIST, doporučení Anthropicu a nařízení EU o umělé inteligenci. Není to návod k instrumentaci ani produktové srovnání. Je to přehled s odkazy, u kterých si každý může ověřit, odkud tvrzení pochází a kdy bylo naposledy zkontrolováno.

Pravidla zápisu:

- Citace jsou v původním jazyce (angličtina) v uvozovkách, parafráze jsou česky bez uvozovek.
- Kde se tvrzení nepodařilo ověřit z primárního zdroje, je to v textu výslovně označené `EVIDENCE-GAP`.
- Kde se dva primární zdroje rozcházejí, jsou uvedené oba a rozpor je označený `ROZPOR`. Souhrn rozporů je na konci dokumentu.
- Stav produktu (GA, public preview) platí k datu ověření. Grafana Labs vydává nové funkce v řádu týdnů, proto má každý záznam vlastní datum.
- Čísla z průzkumů výrobců jsou uvedená jen s konkrétním zdrojem a s poznámkou, že jde o průzkum výrobce.

---

# A. Grafana Labs: produkty a jejich stav

## Z-01 – Agent Observability je GA (dříve AI Observability, interně Sigil, SDK agento11y)

**Co zdroj pokrývá:** Dva záznamy v oficiálním přehledu novinek Grafana Labs („What's new“): přejmenování produktu a oznámení obecné dostupnosti.

**Co přesně říká:**
- Záznam s datem vydání 22. 7. 2026 (štítek „Generally Available“): „Grafana’s AI Observability (alias: Sigil) is being renamed to Grafana Agent Observability.“ Přejmenování vyžaduje úpravy SDK balíčků, vlastních dashboardů, alertů a RBAC oprávnění. Staré názvy fungují v přechodném období, jehož konec byl naplánovaný na konec října 2026. Odkazovaný migrační návod má v URL `migrate-sigil-sdk-to-agento11y`.
- Záznam s datem vydání 30. 7. 2026 (štítek „Generally Available“, „No self-managed release“): „Agent Observability is now generally available in Grafana Cloud.“ Produkt přidává „a new, fifth telemetry signal: agent sessions“ a popisuje oblasti Observe, Evaluate („score response quality continuously and alert when pass rates drop“), Guard („enforce policies in real time“), Experiment a Optimize costs („see exactly what each agent spends, per agent, model, and session“).
- Označení `agento11y` se v primárních zdrojích objevuje jako technický identifikátor (ID pluginu `grafana-agento11y-app`, prefix nástrojů v MCP serveru, název SDK v migračním návodu), ne jako obchodní název. Obchodní název je „Agent Observability“.

**Použití v nástroji:** Hlavní odkaz pro bránu „dohledatelnost kroků agenta“ a „náklady na úlohu“: kde se v Grafaně vidí konverzace agenta, volání nástrojů, kvalita odpovědí a cena. Jen v Grafana Cloud, ne v self-managed Grafaně.

**Odkaz:** https://grafana.com/whats-new/2026-07-30-say-goodbye-to-black-box-agents-with-agent-observability/ a https://grafana.com/whats-new/2026-07-22-ai-observability-is-now-agent-observability--updated-branding-and-migration-guide/

**Datum ověření:** 3. 11. 2026

---

## Z-02 – Agent Observability: co zachytává a jak souvisí s OpenTelemetry

**Co zdroj pokrývá:** Úvodní stránka dokumentace „Introduction to Agent Observability“ a rozcestník produktu.

**Co přesně říká:**
- Základní jednotkou je generation: každé volání LLM poskytovatele s modelem, vstupními zprávami (systémový prompt, uživatelské zprávy, výsledky nástrojů), výstupy (odpovědi, volání nástrojů), spotřebou tokenů a časováním.
- Generace se skládají do konverzací podle `conversation_id`. Konverzace ukazuje časovou osu s trasami, spotřebou tokenů, rozpadem nákladů a skóre kvality.
- Agent catalog: produkt sám eviduje agenty podle `agent_name` a efektivní verze. Výchozí verze se počítá ze systémového promptu, takže změna promptu znamená novou verzi.
- Multi-agent: přes `parent_generation_ids` vzniká graf závislostí a „if an upstream generation fails evaluation, downstream dependents are flagged.“
- Workflow steps zachycují ne-LLM kroky (směrování, plánování, retrieval, orchestrace nástrojů).
- „Agent Observability is built on OpenTelemetry. The SDKs emit standard `gen_ai.*` semantic convention spans and metrics alongside generation data.“
- SDK pro Go, Python, TypeScript, Java a .NET; integrace pro LangChain, LangGraph, OpenAI Agents, LlamaIndex, Google ADK a Vercel AI SDK. Další cesty: příjem čistého OpenTelemetry, trasy z Claude Platform a přeposílání relací coding agentů bez SDK.
- „OpenTelemetry ingestion is in public preview.“

**ROZPOR:** Tisková zpráva z 20. 10. 2026 (Z-17) uvádí „Agent Observability with OpenTelemetry support (GA)“. Dokumentace (úvod i „Get started“) ke stejnému datu ověření píše, že příjem OpenTelemetry bez SDK je ve veřejném preview. Pro nástroj platí: Agent Observability přes vlastní SDK je GA, u příjmu čistého OpenTelemetry se primární zdroje rozcházejí.

**Použití v nástroji:** Mapa „co měřit a kde to v Grafaně vidět“: dohledatelnost (konverzace, generace, volání nástrojů), verze agenta (změna promptu = nová verze, tedy důvod znovu projít bránu), náklady per agent a model. Nepopisovat instrumentaci, jen odkázat.

**Odkaz:** https://grafana.com/docs/grafana-cloud/observe-and-act/agent-observability/introduction/ a https://grafana.com/docs/grafana-cloud/observe-and-act/agent-observability/

**Datum ověření:** 3. 11. 2026

---

## Z-03 – Agent Observability: guards (synchronní brána před voláním modelu)

**Co zdroj pokrývá:** Dokumentace „Protect agents“ a „Set up guards“.

**Co přesně říká:**
- „Guards are synchronous checks that run on the request path before an LLM call. Use them to block harmful prompts, redact sensitive data, or filter dangerous tool calls.“
- Guards používají stejné typy evaluátorů jako online evaluace (LLM judge, regex, JSON schema, heuristic) a vracejí SDK rozhodnutí `allow` nebo `deny`, případně očištěnou kopii vstupu.
- Akce při selhání evaluátoru: `deny` (zablokuje požadavek) nebo `warn` (zaloguje a pustí).
- Tool filter guard blokuje volání nástrojů podle jména přes glob vzory (příklady v dokumentaci: `shell_exec`, `file_delete`, `Bash(*rm*)`); vzory se závorkou se porovnávají i s argumenty volání.
- Přehled guardu ukazuje za posledních 24 hodin počty shod, zamítnutí, varování a redakcí a seznam zamítnutých nebo varovaných konverzací.
- Dokumentace guards nezmiňuje lidské schválení. Guards jsou automatická brána, ne human in the loop.

**Použití v nástroji:** Příklad „vypínače a mezí“ na úrovni jednotlivého volání nástroje. Pro úroveň „agent jedná sám v mezích“ je to doklad, že meze lze vynucovat a měřit (míra zamítnutí). Neplést s lidským schválením.

**Odkaz:** https://grafana.com/docs/grafana-cloud/observe-and-act/agent-observability/protect-agents/ a https://grafana.com/docs/grafana-cloud/observe-and-act/agent-observability/protect-agents/guards/

**Datum ověření:** 3. 11. 2026

---

## Z-04 – Agent Observability: evaluace kvality (online, offline, lidské hodnocení)

**Co zdroj pokrývá:** Dokumentace „Evaluate and improve“ a úvod produktu.

**Co přesně říká:**
- Online evaluace průběžně hodnotí produkční provoz. Čtyři typy evaluátorů: LLM judge, JSON schema, regex, heuristic.
- „You can also create Grafana alert rules directly from evaluation rules to be notified when pass rates drop below a threshold.“
- Offline experimenty porovnávají modely, prompty a verze agenta nad verzovanou testovací sadou; produkční konverzace lze použít jako testovací případy.
- Human review: lidé odpovídají na otázky nad vzorkem konverzací a jejich odpovědi se porovnávají se skóre LLM judge.

**Použití v nástroji:** Brána „error rate a kvalita“: konkrétní měřitelná veličina (pass rate evaluací, alert na pokles) a kalibrace automatického hodnocení lidským. Pro decision record: „jaký pass rate a na jaké sadě byl podmínkou zvýšení autonomie“.

**Odkaz:** https://grafana.com/docs/grafana-cloud/observe-and-act/agent-observability/evaluate-and-improve/

**Datum ověření:** 3. 11. 2026

---

## Z-05 – Grafana Assistant: stav a co umí

**Co zdroj pokrývá:** Oznámení GA, dokumentace „Get started“ a „Explore Assistant capabilities“, související záznamy „What's new“.

**Co přesně říká:**
- „We are excited to announce the General Availability of Grafana Assistant in Grafana Cloud.“ Datum vydání 8. 10. 2025, bez self-managed vydání.
- „Grafana Assistant is an AI-powered observability agent.“ Pět hlavních workflow: Assistant Investigations, správa dashboardů, dotazování (PromQL, LogQL, TraceQL, SQL, k6), navigace v Grafaně, znalosti a best practices.
- Další schopnosti: Workspace, Pages, Skills, Automations, Assistant Watchers, Investigations, Infrastructure memory, Knowledge Graph mode, CLI, Managed Sandboxes.
- Stav dílčích funkcí podle samostatných záznamů „What's new“: Skills v Assistantu GA (17. 3. 2026), Assistant ve Slacku GA (17. 3. 2026), Assistant CLI a Assistant Tunnel public preview (13. 3. 2026). Stránka CLI v dokumentaci k datu ověření stále uvádí public preview.
- Assistant používá modely Claude od Anthropicu (přímo, přes Google Vertex AI nebo Amazon Bedrock); zákazník si poskytovatele nevybírá. U stacků v EU zůstává uložení konverzací, inference i zpracování Agent Observability v EU/EHP.

**Použití v nástroji:** Ukázka, že Grafana sama provozuje AI asistenta i agenta v IT provozu. Nástroj na to odkazuje jako na místo, kde se dané věci dělají, a nepopisuje jejich ovládání.

**Odkaz:** https://grafana.com/whats-new/2025-10-08-grafana-assistant-is-now-generally-available/ , https://grafana.com/docs/grafana-cloud/platform/grafana-assistant/get-started/ a https://grafana.com/docs/grafana-cloud/platform/grafana-assistant/platform/

**Datum ověření:** 3. 11. 2026

---

## Z-06 – Grafana Assistant: identita, auditní stopa a uchování konverzací

**Co zdroj pokrývá:** Dokumentace „Your data and security“ a „Your data and privacy“.

**Co přesně říká:**
- „Assistant actions run under your user identity. When the Assistant creates or updates resources, Grafana Cloud records the changes in user-level audit logs the same way as manual actions.“
- „Grafana Cloud doesn’t provide a separate Assistant-only audit log. Audit logs don’t include full chat transcripts.“
- RBAC umožňuje jemná oprávnění Assistantu (čtení, zápis, spouštění investigací, správa MCP serverů); Assistant smí dotazovat jen datové zdroje, ke kterým má přístup uživatel.
- Konverzace (prompty, odpovědi, metadata) se ukládají v tenantu v regionu stacku. Poskytovatelé modelů zpracovávají data se zero retention.
- Chat retention platí pro všechny chaty Assistantu „including chats created by investigations, automations, watchers, and other agents“. Od 1. 9. 2026 Grafana plánovala výchozí 90denní lhůtu pro organizace, které si žádnou nezvolily. Po aktivaci nejde retenci vypnout, jen změnit lhůtu.
- Dokumentace RBAC uvádí mimo jiné role Assistant Admin, Assistant User, Assistant Investigation User, Assistant MCP User, Assistant Cloud MCP User, Assistant Coding Admin a Assistant Durable Agents User/Admin (agenti „Grots“).

**EVIDENCE-GAP:** Stav produktu „Durable Agents“ (Grots) se z primárního zdroje nepodařilo zjistit, objevuje se jen v dokumentaci RBAC. Sekundární zprávy zmiňují „Grafana Cloud Agents“ v public preview; oficiální oznámení s tímto názvem se nepodařilo dohledat.

**Použití v nástroji:** Brána „auditní stopa“: změna provedená agentem je v auditním logu vidět jako změna uživatele a samotný přepis konverzace v auditním logu není. Doporučení pro decision record: ověřit, jestli retence konverzací stačí na dohledání rozhodnutí agenta, a nastavit ji vědomě.

**Odkaz:** https://grafana.com/docs/grafana-cloud/platform/grafana-assistant/privacy-and-security/security/ , https://grafana.com/docs/grafana-cloud/platform/grafana-assistant/privacy-and-security/privacy/ a https://grafana.com/docs/grafana-cloud/platform/grafana-assistant/privacy-and-security/rbac/

**Datum ověření:** 3. 11. 2026

---

## Z-07 – Grafana Assistant: schvalování akcí (MCP nástroje, Skills, Pages, CLI)

**Co zdroj pokrývá:** Dokumentace „MCP servers“, „Create Skills“, „Configure Grafana Pages“ a „Use Grafana Assistant CLI“.

**Co přesně říká:**
- Externí MCP servery: každý nástroj lze zapnout nebo vypnout a je označený jako **Read** nebo **Write**. Politika schválení na nástroj: **Default**, **Auto-approve** („Allow Assistant to run the enabled tool without asking first“), **Always ask** („Require explicit approval every time“).
- „Use auto-approval carefully. Auto-approving enabled tools lets Assistant run them without prompting you, including tools that may write, modify, or delete data in the connected service.“
- Change review pull requestů: „Nobody approves tool calls during a review, so a tool that would ask for approval doesn’t run.“
- Skills mohou mít seznam automaticky schválených MCP nástrojů; „If an admin sets a tool to **Always ask**, allowed-tools auto-approval can’t override it.“
- Grafana Pages (public preview): před akcí uživatel vidí cíl a hodnoty; volba „Always allow for this page“ povolí všechny podporované akce stránky bez dalšího schválení, „including requests that change external systems“. „Revoking approval can’t undo an action already sent.“
- Assistant CLI (public preview): přístup k lokálním souborům je ve výchozím stavu jen pro čtení, spouštění shell příkazů je volitelné a vyžaduje schválení.

**EVIDENCE-GAP:** Co přesně znamená politika **Default** u vestavěných nástrojů Grafany (dashboardy, alerty), dokumentace nespecifikuje.

**Použití v nástroji:** Přímý příklad přechodu mezi úrovněmi „agent jedná se schválením člověka“ a „agent jedná sám v mezích“ na jednom produktu: přepnutí z Always ask na Auto-approve je přesně to rozhodnutí, ke kterému má nástroj vést decision record. Varování dokumentace o auto-approve lze citovat jako argument, proč zvyšovat autonomii po nástrojích, ne plošně.

**Odkaz:** https://grafana.com/docs/grafana-cloud/platform/grafana-assistant/configure/mcp-servers/ , https://grafana.com/docs/grafana-cloud/platform/grafana-assistant/platform/skills/ , https://grafana.com/docs/grafana-cloud/platform/grafana-assistant/configure/pages/ a https://grafana.com/docs/grafana-cloud/platform/grafana-assistant/platform/cli/

**Datum ověření:** 3. 11. 2026

---

## Z-08 – Assistant Investigations: GA, automatické spouštění, limity

**Co zdroj pokrývá:** Záznam „What's new“ z 29. 7. 2026, dokumentace „Grafana Assistant Investigations“ a „Configure investigation settings“.

**Co přesně říká:**
- „Assistant Investigations is now generally available.“ Investigace prochází metriky, logy, trasy a profily, tvoří hypotézy a vydá strukturovaný report „with findings, supporting evidence, and recommended next steps“.
- Spouštění: ručně, přes alert enrichment nebo IRM webhooky („an investigation starts automatically“), ze Slacku, z Grafana Cloud MCP serveru nebo z gcx.
- Člověk může hypotézu označit **Disprove** nebo **Reopen** a investigaci přesměrovat; po reportu hodnotí verdikt.
- Ochrana proti zahlcení: limity nových investigací podle závažnosti; „By default, **All** limits alert enrichment to 50 new investigations per 20 minutes.“ IRM webhooky mají samostatný limit se stejným výchozím nastavením.
- Sift Investigation je podle dokumentace samostatná funkce pro automatické třídění v Kubernetes bez promptu.

**ROZPOR:** Referenční soubor v oficiálním repozitáři `grafana/skills` (skill `ml-ai`, `references/llm-and-graph.md`) k datu ověření stále uvádí „Assistant Investigations (public preview)“. Novější primární zdroj (What's new 29. 7. 2026, tisková zpráva 27. 7. 2026) uvádí GA.

**Použití v nástroji:** Vzor pro scénář „třídění alertů“: agent sám spouští analýzu (jen čtení dat), výsledek je report pro člověka. Limit počtu investigací je příklad brány na náklady.

**Odkaz:** https://grafana.com/whats-new/2026-07-29-assistant-investigations-is-now-generally-available-in-grafana-cloud/ , https://grafana.com/docs/grafana-cloud/platform/grafana-assistant/platform/investigation/ a https://grafana.com/docs/grafana-cloud/platform/grafana-assistant/configure/investigation-alerts/

**Datum ověření:** 3. 11. 2026

---

## Z-09 – Assistant Watchers: GA, běh pod servisním účtem, samočinná rekalibrace

**Co zdroj pokrývá:** Záznam „What's new“ z 19. 10. 2026 a dokumentace „Use Assistant Watchers“.

**Co přesně říká:**
- „Assistant Watchers are now generally available.“ Watchers jsou „always-on agents for the problems that are hard to write an alert rule for“. Kritická zjištění mohou automaticky spustit Assistant Investigation.
- „Runs use the stack’s Assistant service account, not your identity.“
- Automatická rekalibrace: Assistant pravidelně (týdně nebo měsíčně) přezkoumá a upraví kontroly, prahy a baseline. Revize „run in the background without pausing scanning or asking questions“, běží pod identitou tvůrce watcheru. Změny jsou ve verzích a lze je vrátit.
- Od 1. 12. 2026 se Watchers účtují podle spotřeby tokenů Assistantu; stránka watcheru ukazuje odhad tokenů.

**Použití v nástroji:** Ukázka agenta, který bez ptaní mění vlastní konfiguraci (prahy) a běží pod servisním účtem, ne pod člověkem. Pro bránu „kdo schvaluje“ a „auditní stopa“ je to modelová otázka: pod jakou identitou agent jedná a kde jsou vidět jeho změny (zde verze watcheru).

**Odkaz:** https://grafana.com/whats-new/2026-10-19-assistant-watchers-are-now-generally-available-in-grafana-cloud/ a https://grafana.com/docs/grafana-cloud/platform/grafana-assistant/platform/watchers/

**Datum ověření:** 3. 11. 2026

---

## Z-10 – Assistant Automations a Grafana Workflows

**Co zdroj pokrývá:** Dokumentace „Use Automations“, záznam „What's new“ ke Grafana Workflows (datum vydání 20. 10. 2026) a tisková zpráva z 27. 7. 2026.

**Co přesně říká:**
- Automations jsou uložené prompty Assistantu spouštěné ručně, podle plánu nebo událostí (např. GitHub pull request). Historie běhů uvádí spouštěč a odkaz na konverzaci běhu; neúspěšný běh je zaznamenaný se stavem `failed`. Podle tiskové zprávy z 27. 7. 2026 jsou Automations GA.
- Omezené plány („bounded schedules“): Assistant před spuštěním ukáže jedno potvrzení s kadencí, koncem, očekávaným počtem běhů a varováním o spotřebě tokenů.
- „Grafana Workflows is now in public preview for all Grafana Cloud stacks.“ Kroky workflow mohou vyhlásit a aktualizovat incident, poslat page přes IRM, volat libovolné HTTP API a spouštět Python v sandboxu. Krok „Ask Assistant“ předá úlohu Assistantu. Workflows mají RBAC, uložení tokenů jako secrets a „inspect every run step by step“.

**Použití v nástroji:** Scénář „automatický restart služby“: v ekosystému Grafany by šlo o kombinaci deterministického workflow a kroku s AI. Nástroj má pomoci rozlišit, který krok rozhoduje AI a který je předem daný, a podle toho nastavit bránu. Workflows jsou v preview, v posouzení to uvést.

**Odkaz:** https://grafana.com/docs/grafana-cloud/platform/grafana-assistant/platform/automations/ a https://grafana.com/whats-new/2026-10-13-automate-incident-response-and-operations-with-grafana-workflows/

**Datum ověření:** 3. 11. 2026

---

## Z-11 – Grafana Managed Sandboxes a Grafana Pages (public preview)

**Co zdroj pokrývá:** Dokumentace „Use Grafana Managed Sandboxes“ a „Configure Grafana Pages“.

**Co přesně říká:**
- Obě funkce mají v dokumentaci banner: „currently in public preview. Grafana Labs offers limited support, and breaking changes might occur prior to the feature being made generally available.“
- Managed Sandboxes: agent v sandboxu upraví kód a připraví pull request. Politika publikace repozitáře: **Ask before publishing** („Assistant pushes nothing to GitHub until you approve“) nebo **Auto-publish PRs** (otevírá draft PR bez ptaní, jen u ověřených privátních repozitářů).
- Předání úlohy externímu poskytovateli (Cursor, Claude): „Grafana can’t review changes before they reach GitHub.“
- Change review kontroluje pull request proti živé telemetrii na pět druhů rizik: deploy hazards, production impact, observability breakage, sensitive telemetry, instrumentation.

**Použití v nástroji:** Scénář „návrh opravy“: hranice mezi „agent navrhne“ a „agent změní“ leží u merge pull requestu, který zůstává na člověku. Užitečné jako příklad úrovně „agent jedná se schválením člověka“.

**Odkaz:** https://grafana.com/docs/grafana-cloud/platform/grafana-assistant/platform/managed-sandboxes/ a https://grafana.com/docs/grafana-cloud/platform/grafana-assistant/configure/pages/

**Datum ověření:** 3. 11. 2026

---

## Z-12 – Grafana Cloud MCP server (hostovaný): GA a úrovně přístupu

**Co zdroj pokrývá:** Záznam „What's new“ z 28. 7. 2026 a dokumentace „Use Grafana Cloud MCP tools“ a „Choose Grafana Cloud MCP tool areas“.

**Co přesně říká:**
- Záznam s datem vydání 28. 7. 2026 má štítek „Generally Available“: „Give your AI agents instant, zero-infrastructure access to your entire observability ecosystem.“
- Úrovně přístupu při autorizaci externího agenta: **Read** (prohlížení zdrojů a dotazy jen na čtení), **Query**, **Write** („Also create, change, and delete resources“). Pro zápis je potřeba role Assistant Admin.
- Nástroje pro zápis zahrnují mimo jiné `alerting_rules_write`, `alerting_silences_write`, `alerting_routing_write`. `ask_assistant` vyžaduje zápisový přístup, „because Assistant can perform actions that change resources“.
- „Grafana Cloud MCP tool areas is currently in public preview.“ (výběr skupin nástrojů, které externí agent uvidí)

**ROZPOR (název):** Podtitulek tiskové zprávy z 27. 7. 2026 uvádí „the Grafana MCP server“, text téže zprávy „the Grafana Cloud MCP server“. Jde o hostovaný server Grafana Cloud, odlišný od open-source `grafana/mcp-grafana` (Z-13).

**Použití v nástroji:** Kontrolní otázka pro každé nasazení externího agenta (Claude, Copilot, Cursor) nad Grafanou: s jakou úrovní přístupu je připojený. Read bez Write je jednoduchá a měřitelná mez pro úroveň „asistent radí“.

**Odkaz:** https://grafana.com/whats-new/2026-07-28-cloud-mcp-server--connect-your-agents-to-grafana-cloud-with-a-single-url/ , https://grafana.com/docs/grafana-cloud/platform/grafana-assistant/configure/cloud-mcp/ a https://grafana.com/docs/grafana-cloud/platform/grafana-assistant/configure/cloud-mcp-tool-areas/

**Datum ověření:** 3. 11. 2026

---

## Z-13 – Open-source MCP server `grafana/mcp-grafana`: režim jen pro čtení

**Co zdroj pokrývá:** Repozitář `grafana/mcp-grafana` na GitHubu, README. Poslední release v2.2.0 z 2. 11. 2026, licence Apache-2.0.

**Co přesně říká:**
- „The `--disable-write` flag provides a way to run the MCP server in read-only mode, preventing any write operations to your Grafana instance.“
- Kategorie nástrojů lze vypínat přepínači `--disable-<category>`; některé kategorie jsou ve výchozím stavu vypnuté a zapínají se přes `--enabled-tools`.
- Nástroje Agent Observability (`agento11y_*`) jsou „disabled by default and work only in Grafana Cloud“. Umí číst konverzace, generace, skóre evaluací, katalog agentů a jejich verze, evaluátory, eval rules a guards („guards (hook rules) that run inline and can warn or deny“) a offline experimenty.
- Nástroje Assistantu jsou ve výchozím stavu vypnuté a jsou to „write tools (the assistant may mutate stack state), so they are skipped when `--disable-write` is set.“
- Zápisové nástroje zahrnují mimo jiné správu incidentů v Grafana Incident a `update_alert_group` (acknowledge, resolve).

**Použití v nástroji:** Konkrétní technická mez pro úroveň „asistent radí“ při práci s Grafanou z vlastního agenta. Ukazuje, že i čtení dat agentem je oddělitelné od zápisu.

**Odkaz:** https://github.com/grafana/mcp-grafana

**Datum ověření:** 3. 11. 2026

---

## Z-14 – Sift

**Co zdroj pokrývá:** Dokumentace „Sift investigations“.

**Co přesně říká:**
- Sift je „a powerful, free diagnostic assistant included in Grafana Cloud“, který spustí sadu kontrol nad telemetrií a sestaví seznam „interesting results“.
- „By default, users with the **Viewer** basic role can start investigations because running an investigation only queries existing data.“ Úprava kontrol vyžaduje roli Sift Editor.
- Dokumentace Assistant Investigations popisuje Sift Investigation jako samostatnou funkci „for automatic Kubernetes triage without a prompt“.

**EVIDENCE-GAP:** Formální oznámení GA Siftu se nepodařilo dohledat. Dokumentace k datu ověření nemá banner preview a popisuje Sift jako součást Grafana Cloud zdarma.

**Použití v nástroji:** Příklad automatizace, která jen čte data, a proto ji Grafana pouští i roli Viewer. Pro nástroj argument, že rozhodující pro bránu je zápis a dopad akce, ne samotné použití AI nebo ML.

**Odkaz:** https://grafana.com/docs/grafana-cloud/machine-learning/sift/

**Datum ověření:** 3. 11. 2026

---

## Z-15 – Knowledge Graph a infrastructure memory

**Co zdroj pokrývá:** Dokumentace „Knowledge graph“.

**Co přesně říká:**
- „The knowledge graph is included in every Grafana Cloud stack.“ Observability řešení do něj přidávají entity, které monitorují, a insights o jejich zdraví.
- Entity a insights podle dokumentace „typically add 5% to 10% more active series. There is no separate charge.“
- Grafana Assistant používá entity a vztahy z grafu; „Infrastructure memory is a capability of the knowledge graph.“
- Součástí jsou entity catalog, entity graph (závislosti) a RCA workbench (časová osa insights).

**Použití v nástroji:** Kontext pro agenta: pokud agent nemá model závislostí, jeho návrhy zásahu nevidí dopad na okolní služby. V posouzení jako otázka „má agent k dispozici topologii?“, s odkazem sem.

**Odkaz:** https://grafana.com/docs/grafana-cloud/knowledge-graph/

**Datum ověření:** 3. 11. 2026

---

## Z-16 – Grafana IRM

**Co zdroj pokrývá:** Dokumentace „Grafana IRM“ a „Configure IRM webhooks for automated investigations“.

**Co přesně říká:**
- „Grafana IRM is the on-call management and incident response product in Grafana Cloud.“ Pokrývá rozpisy služeb, eskalace, vyhlášení a řízení incidentu až po post-incident review. Je to placený doplněk účtovaný podle měsíčně aktivních uživatelů IRM.
- IRM webhooky mohou automaticky spustit Assistant Investigation při vytvoření skupiny alertů nebo vyhlášení incidentu, „providing context and insights before human responders arrive“.

**Použití v nástroji:** Místo, kde se v Grafaně odehrává předávka mezi agentem a člověkem (kdo je on-call, kdo schvaluje). Pro bránu „kdo schvaluje“ odkázat na eskalační řetězce v IRM.

**Odkaz:** https://grafana.com/docs/grafana-cloud/alerting-and-irm/irm/ a https://grafana.com/docs/grafana-cloud/platform/grafana-assistant/configure/irm-webhooks/

**Datum ověření:** 3. 11. 2026

---

## Z-17 – Tiskové zprávy Grafana Labs 27. 7. 2026 a 20. 10. 2026: souhrn stavů a čísla z průzkumů

**Co zdroj pokrývá:** Dvě tiskové zprávy Grafana Labs (AI Week, ObservabilityCON 2026).

**Co přesně říká:**
- 27. 7. 2026: GA šesti funkcí: Grafana Assistant Investigations, Grafana Assistant Workspace, Grafana Assistant Automations, Grafana Cloud MCP server, gcx a Grafana Agent Observability. Citace Mat Ryer (Grafana Labs): „It’s the same assistant end-to-end, and it doesn’t just advise; it acts.“ Číslo z průzkumu výrobce: podle „Grafana Labs’ 2026 Observability Survey“ 92 % praktiků vidí přínos v tom, že AI zachytí anomálie, ale jen 57 % uvádí, že nějak řeší observabilitu vlastních AI systémů.
- 20. 10. 2026 (titulek „Grafana Labs Charts the Path to Agentic Operations with Watchers, Pages, and Agent Observability at ObservabilityCON 2026“): Assistant Watchers (GA), Agent Observability with OpenTelemetry support (GA), Grafana Pages (public preview), Grafana Managed Sandboxes (public preview). Assistant Investigations „generally available since late July“. Managed Sandboxes připravují opravu do pull requestu „for human review and approval“. Grafana Labs uvádí, že staví agenty, kteří jednají, a nechává na každé organizaci rozhodnutí, kolik lidského dohledu si ponechá „based on its own level of trust and risk profile“.
- Číslo z průzkumu výrobce ve zprávě z 20. 10. 2026 (zdroj uvedený ve zprávě: report Grafana Labs a ETR „2026 State of Observability“, 300 IT a engineering lídrů): „85% report comfort with read-only investigation, compared to 52% for full autonomous remediation.“

**Poznámka:** Obě čísla jsou z průzkumů výrobce, každé z jiného průzkumu. Pokud je nástroj použije, vždy s tímto zdrojem a označením „průzkum Grafana Labs“. Plné texty průzkumů nebyly v této rešerši ověřené (`EVIDENCE-GAP` pro metodiku průzkumů).

**Použití v nástroji:** Rámec, ve kterém se pohybuje akce 25. 11. 2026: Grafana sama říká, že míru dohledu volí organizace. To je přesně místo pro náš nástroj. Rozdíl 85 % vs. 52 % ilustruje, že důvěra se liší podle úrovně autonomie, ne podle technologie.

**Odkaz:** https://grafana.com/press/2026/07/27/grafana-labs-ships-six-tools-that-power-agentic-operations-from-planning-to-production/ a https://grafana.com/press/2026/10/20/observabilitycon-2026-ai-updates/

**Datum ověření:** 3. 11. 2026

---

# B. Oficiální agent skills od Grafany

## Z-18 – Repozitář `grafana/skills`: co obsahuje a co nástroj nesmí duplikovat

**Co zdroj pokrývá:** Repozitář `grafana/skills` na GitHubu (README, strom souborů). Licence Apache-2.0, poslední commit ve větvi `main` 27. 10. 2026 (8fb72a5).

**Co přesně říká:**
- „Public skills for working with Grafana, Prometheus, Loki, Tempo, Pyroscope, k6, and the broader LGTM observability stack. Compatible with Claude Code, Cursor, Codex, and any tool supporting the Agent Skills open standard.“
- Instalace přes `npx skills add grafana/skills` nebo jako plugin marketplace v Claude Code (`claude plugin marketplace add grafana/skills`).
- Repozitář obsahuje 53 souborů SKILL.md v sedmi skupinách (52 je uvedených v tabulkách README, navíc `grafana-core/skill-authoring`):
  - **grafana-core (8):** grafana-oss, dashboarding, promql, alerting-irm, alloy, beyla, opentelemetry, skill-authoring.
  - **grafana-cloud (20):** admin, send-data, fleet-management, cloud-integrations, infrastructure, app-observability (APM, Frontend Observability, AI Observability), frontend-session-rca, database-observability, adaptive-metrics, cost-management, dpm-finder, loki-label-analyzer, prometheus-label-strategy, prometheus-cardinality-troubleshooter, oncall-irm, ml-ai (Assistant, Dynamic Alerting, Sift, Knowledge Graph, LLM Plugin), assistant-mcp, private-connectivity, testing, synthetic-monitoring-checks.
  - **grafana-lgtm (7):** loki, tempo, prometheus, mimir, pyroscope, profilecli-insights, performance-rca.
  - **grafana-plugins (5):** plugin-bundle-size, react-19-plugin-migration, grafana-scenes, check-npm, audit-and-reduce-dependencies.
  - **grafana-app-sdk (4):** app-sdk-concepts, cue-kind-definition, reconciler-logic, admission-control.
  - **grafana-k6 (8):** k6, k6-docs, k6-perf-test-website, k6-manage, k6-cloud-investigate-test, k6-trend-analysis, k6-test-maintenance, k6-browser-test.
  - **grafana-datasources (1):** datasources-provisioning.
- Fulltextové hledání v obsahu všech skills: žádný skill neřeší úrovně autonomie agentů, human in the loop, readiness ani rozhodování, kam agenta pustit. AI témata pokrývají jen `app-observability` (instrumentace LLM aplikací, metriky tokenů a nákladů, evals a guards přes OpenLIT), `ml-ai` (zapnutí AI a ML funkcí Grafany) a `assistant-mcp` (připojení coding agentů ke Grafana Cloud přes MCP).

**ROZPOR:** Skill `app-observability` popisuje AI Observability přes OpenLIT (`openlit.init()`, dashboard „GenAI Observability“). Aktuální dokumentace Grafany (Z-02) popisuje Agent Observability přes vlastní SDK, OpenTelemetry, Claude Platform a přeposílání relací coding agentů a OpenLIT nezmiňuje. Skill `ml-ai` uvádí Investigations jako public preview (viz Z-08). Repozitář tedy v AI části zaostává za dokumentací.

**Použití v nástroji:** Hranice scope. Nástroj nesmí dělat: instrumentaci (OpenTelemetry, Alloy, OpenLIT, SDK), dashboardy, PromQL, LogQL, TraceQL, alerty, konfiguraci IRM a on-call, nastavení AI funkcí Grafany, připojení agenta přes MCP. Prostor pro nás: rozhodnutí, kam AI v provozu pustit, kolik autonomie dát a jaké brány musí být splněné. Na skills Grafany odkazovat jako na „jak to pak změřit“.

**Odkaz:** https://github.com/grafana/skills

**Datum ověření:** 3. 11. 2026

---

# C. OpenTelemetry GenAI sémantické konvence

## Z-19 – GenAI semantic conventions: přesun do samostatného repozitáře a stav Development

**Co zdroj pokrývá:** Repozitář `open-telemetry/semantic-conventions-genai`, stránka GenAI na opentelemetry.io a původní umístění v `open-telemetry/semantic-conventions`.

**Co přesně říká:**
- Stránka na opentelemetry.io (pod verzí Semantic conventions 1.44.0) i soubor v hlavním repozitáři: „GenAI semantic conventions have moved to the OpenTelemetry GenAI semantic conventions repository. This page has moved and is no longer maintained in this repository.“ Hlavní repozitář vydal 2. 11. 2026 verzi v1.45.0, GenAI konvence v ní už nejsou.
- Nový repozitář `semantic-conventions-genai` (založený 5. 5. 2026) zatím nemá vydanou verzi ani schema URL (README: „Schema URL: TODO“).
- „Semantic conventions for generative AI systems. **Status**: Development“. Konvence pokrývají signály: Client Inference, Client Embeddings, Events, Exceptions, Metrics, Inference Token Metrics, Model spans, Agent spans. Dále konvence pro Anthropic, Azure AI Inference, AWS Bedrock, OpenAI, Agent2Agent Protocol (A2A) a Model Context Protocol (MCP).
- Sdílený atribut napříč operacemi: `gen_ai.conversation.id`.

**Použití v nástroji:** Odkaz pro bránu „dohledatelnost“ jako na otevřený standard nezávislý na výrobci. V posouzení vždy uvést, že standard je ve stavu Development, tedy názvy atributů se mohou měnit.

**Odkaz:** https://github.com/open-telemetry/semantic-conventions-genai a https://opentelemetry.io/docs/specs/semconv/gen-ai/

**Datum ověření:** 3. 11. 2026

---

## Z-20 – GenAI spany a metriky pro agenty a nástroje

**Co zdroj pokrývá:** Dokumenty `gen-ai-agent-spans.md`, `gen-ai-spans.md`, `gen-ai-metrics.md`, `gen-ai-token-metrics.md` a `mcp.md` v repozitáři `semantic-conventions-genai`.

**Co přesně říká:**
- Agentní spany (všechny se stavem Development), hodnota `gen_ai.operation.name`: `create_agent`, `invoke_agent` (klientský i interní span), `invoke_workflow`, `plan`. Dále spany pro agent skills (load skill, read skill resource) a command execution.
- Span pro nástroje: `gen_ai.operation.name` „SHOULD be `execute_tool`“. „GenAI instrumentations that can instrument tool execution calls SHOULD do so.“ Volání nástrojů přes MCP lze trasovat i MCP instrumentací.
- Metriky (Development): `gen_ai.client.operation.duration`, `gen_ai.server.request.duration`, `gen_ai.server.time_per_output_token`, `gen_ai.server.time_to_first_token`, `gen_ai.invoke_workflow.duration`, `gen_ai.invoke_agent.duration`, `gen_ai.invoke_agent.inference_calls`, `gen_ai.invoke_agent.tool_calls`, `gen_ai.execute_tool.duration`.
- Tokenové metriky: `gen_ai.client.inference.usage.input_tokens`, `.output_tokens`, `.cache_read.input_tokens`, `.cache_write.input_tokens`, `.reasoning.output_tokens` a `gen_ai.client.inference.operation.input_tokens` / `.output_tokens`.
- MCP metriky: `mcp.client.operation.duration`, `mcp.server.operation.duration`, `mcp.client.session.duration`, `mcp.server.session.duration`.

**Použití v nástroji:** Slovník pro bránu „co musí být měřené“: počet volání nástrojů na úlohu, doba běhu agenta, tokeny na úlohu (náklady). Nástroj jen jmenuje, co má být vidět, a odkazuje sem; jak to nastavit, řeší Grafana a její skills.

**Odkaz:** https://github.com/open-telemetry/semantic-conventions-genai/tree/main/docs/gen-ai

**Datum ověření:** 3. 11. 2026

---

## Z-21 – GenAI konvence: obsah promptů a odpovědí se ve výchozím stavu nezaznamenává

**Co zdroj pokrývá:** Sekce „Capturing instructions, inputs, and outputs“ v `gen-ai-spans.md`.

**Co přesně říká:**
- „Model instructions, user messages, and model outputs are considered sensitive and are often large in size.“
- „OpenTelemetry instrumentations SHOULD NOT capture them by default, but SHOULD provide an option for users to opt in.“
- Výchozí vzor: nezaznamenávat instrukce, vstupy ani výstupy. Zápis do atributů `gen_ai.system_instructions`, `gen_ai.input.messages`, `gen_ai.output.messages` je vhodný tam, kde je objem zvládnutelný a buď se neuplatní předpisy o ochraně soukromí, nebo jim úložiště telemetrie vyhovuje; alternativou je nahrávání obsahu do externího úložiště.

**Použití v nástroji:** Důležitý rozpor mezi „dohledatelností“ a ochranou dat. Brána „dohledatelnost kroků agenta“ musí obsahovat vědomé rozhodnutí, jestli a kde se obsah konverzací ukládá. Bez něj se v trasách objeví jen metadata. Právní posouzení (GDPR) mimo scope, jen upozornit.

**Odkaz:** https://github.com/open-telemetry/semantic-conventions-genai/blob/main/docs/gen-ai/gen-ai-spans.md

**Datum ověření:** 3. 11. 2026

---

# D. Rámce úrovní autonomie a lidského dohledu

## Z-22 – Feng, McDonald, Zhang: Levels of Autonomy for AI Agents (Knight First Amendment Institute)

**Co zdroj pokrývá:** Původní práce K. J. Kevin Feng, David W. McDonald, Amy X. Zhang (University of Washington), arXiv:2506.12469, v1 14. 6. 2025, v2 28. 7. 2025. PDF je „accompaniment to the web publication by the Knight 1st Amendment Institute“, licence CC BY 4.0.

**Co přesně říká:**
- „We define an AI agent’s autonomy as the extent to which it is designed to act without user involvement.“ Autonomie je „a deliberate design decision“, oddělená od schopností: „a capable agent ... can still act with a low level of autonomy if it is required to consult its user before taking each action.“
- Pět úrovní podle role uživatele, s povinnými kontrolami (Table 1):
  - **L1 Operator:** uživatel plánuje, agent jedná na vyžádání. Kontroly: „User-managed planning“, „Invocation or approval before actions“.
  - **L2 Collaborator:** společné plánování a provádění, uživatel může kdykoli převzít práci. Kontroly: předání řízení oběma směry, sdílený přehled o postupu.
  - **L3 Consultant:** agent plánuje a provádí většinu, uživatele konzultuje kvůli expertíze a preferencím. Kontrola: rozhraní pro bohatou zpětnou vazbu (nejen schválení).
  - **L4 Approver:** agent zapojí uživatele jen při selhání, vysokém riziku nebo potřebě schválení. Kontroly: „Approval elicitation for consequential actions“, „Customizable conditions for seeking approval“. Otevřená otázka práce: „How to prevent user disengagement and meaningless approvals?“
  - **L5 Observer:** agent plánuje a provádí vše, uživatel může „monitor and audit agent via activity logs“. Kontrola: „Emergency off switch“.
- Navrhuje „autonomy certificates“: certifikát platí pro konkrétní technickou specifikaci a provozní prostředí a jejich změna (např. nový nástroj schopný měnit databázi) ho může zneplatnit.
- Příklady systémů v tabulce jsou podle stavu k červnu 2025.

**Použití v nástroji:** Hlavní teoretická opora pro naše čtyři pracovní úrovně (asistent radí / se schválením / sám v mezích / sám). Mapování nástroj odvodí sám a označí jako vlastní. Myšlenka certifikátu platného jen pro danou sadu nástrojů je přímá opora pro decision record a pro pravidlo „nový nástroj = znovu projít bránu“. Otázka „meaningless approvals“ podporuje bránu proti únavě ze schvalování.

**Odkaz:** https://arxiv.org/abs/2506.12469 a https://knightcolumbia.org/content/levels-of-autonomy-for-ai-agents-1

**Datum ověření:** 3. 11. 2026

---

## Z-23 – AWS Agentic AI Security Scoping Matrix

**Co zdroj pokrývá:** Stránka AWS „Agentic AI Security Scoping Matrix“.

**Co přesně říká:**
- Čtyři rozsahy (scopes) podle míry agency a lidského dohledu:
  - **Scope 1, No Agency:** „Human-initiated systems with no autonomous change capabilities.“ Agent nemůže měnit externí systémy „without explicit human action“.
  - **Scope 2, Prescribed Agency:** „Human-initiated systems that can recommend actions, including the ability to make changes to the environment.“ Změny vyžadují „mandatory human approval before execution“.
  - **Scope 3, Supervised Agency:** „Human-initiated systems with autonomous execution capabilities.“ Po aktivaci jedná „without requiring further HITL approvals“.
  - **Scope 4, Full Agency:** „Self-initiating systems that operate continuously with minimal human oversight.“
- Šest bezpečnostních dimenzí: Identity Context; Data, Memory & State Protection; Audit & Logging; Agent & LLM Controls; Agency Perimeters & Policies; Orchestration.
- Stránka nemá datum publikace ani aktualizace (`EVIDENCE-GAP` pro datum vzniku).

**Použití v nástroji:** Druhá opora pro úrovně. Rozlišení Scope 3 („sám v mezích“, spouští člověk) a Scope 4 („sám“, spouští se sám) odpovídá našim dvěma horním úrovním lépe než Feng et al. Šest dimenzí lze použít jako kontrolní seznam bran.

**Odkaz:** https://aws.amazon.com/ai/security/agentic-ai-scoping-matrix/

**Datum ověření:** 3. 11. 2026

---

## Z-24 – Etické pokyny pro zajištění důvěryhodnosti UI (HLEG): human-in-the-loop, on-the-loop, in-command

**Co zdroj pokrývá:** Etické pokyny Expertní skupiny na vysoké úrovni pro umělou inteligenci (HLEG) z 8. 4. 2019, oficiální česká verze dokumentu na portálu Evropské komise.

**Co přesně říká (oficiální český text, bod 65):**
- „Dohled může být prováděn pomocí řídicích mechanismů, jako je aktivní zapojení člověka (human-in-the-loop, HITL), sledování ze strany člověka (human-on-the-loop, HOTL) nebo člověk ve velení (human-in-command, HIC).“
- „Aktivním zapojením člověka (HITL) se rozumí možnost lidského zásahu do každého rozhodovacího cyklu v systému, což v mnoha případech není možné ani žádoucí.“
- „Sledováním ze strany člověka (HOTL) se rozumí možnost lidského zásahu během cyklu návrhu a sledování činnosti systému.“
- „Člověkem ve velení (HIC) se rozumí možnost celkově dohlížet na činnost systému UI ... a schopnost v každé konkrétní situaci rozhodnout, kdy a jak systém použít.“

**Použití v nástroji:** Definice pojmů human in the loop a human on the loop ve slovníku, s oficiálním českým zněním. Nejde o právně závazný text (to je až AI Act, Z-33).

**Odkaz:** https://digital-strategy.ec.europa.eu/en/library/ethics-guidelines-trustworthy-ai (česká verze PDF: https://ec.europa.eu/newsroom/dae/document.cfm?doc_id=60421)

**Datum ověření:** 3. 11. 2026

---

# E. Bezpečnostní rámce a doporučení k agentům

## Z-25 – OWASP Top 10 for LLM Applications 2025: LLM06 Excessive Agency

**Co zdroj pokrývá:** Záznam LLM06:2025 projektu OWASP GenAI Security Project.

**Co přesně říká:**
- Excessive Agency je zranitelnost, která umožní škodlivé akce „in response to unexpected, ambiguous or manipulated outputs from an LLM“.
- Tři kořenové příčiny: „excessive functionality“, „excessive permissions“, „excessive autonomy“.
- Opatření: Minimize extensions; Minimize extension functionality; Avoid open-ended extensions; Minimize extension permissions; Execute extensions in user's context; Require user approval; Complete mediation; Sanitise LLM inputs and outputs.

**Použití v nástroji:** Tři osy doptávání (co agent smí volat, s jakými právy, jak moc sám). Opatření „Require user approval“ odpovídá úrovni „se schválením“, „Complete mediation“ (autorizace v cílovém systému, ne v modelu) bráně pro úroveň „sám v mezích“.

**Odkaz:** https://genai.owasp.org/llmrisk/llm062025-excessive-agency/

**Datum ověření:** 3. 11. 2026

---

## Z-26 – OWASP Top 10 for Agentic Applications for 2026 a princip Least-Agency

**Co zdroj pokrývá:** Dokument OWASP GenAI Security Project zveřejněný 9. 12. 2025 (PDF, 57 stran).

**Co přesně říká:**
- Deset rizik: ASI01 Agent Goal Hijack; ASI02 Tool Misuse and Exploitation; ASI03 Identity and Privilege Abuse; ASI04 Agentic Supply Chain Vulnerabilities; ASI05 Unexpected Code Execution (RCE); ASI06 Memory & Context Poisoning; ASI07 Insecure Inter-Agent Communication; ASI08 Cascading Failures; ASI09 Human-Agent Trust Exploitation; ASI10 Rogue Agents.
- „We expand on the concepts of Least-Privilege and Excessive Agency by citing Least-Agency. This captures our advice to organizations to avoid unnecessary autonomy; deploying agentic behavior where it is not needed expands the attack surface without adding value.“
- „Similarly, strong observability becomes non-negotiable: without clear visibility into what agents are doing, why they are doing it, and which tools they are invoking, unnecessary autonomy can quietly expand the attack surface.“
- Mezi opatřeními: lidské schválení pro eskalaci oprávnění (ASI03), kill switch, neměnné a podepsané auditní logy.

**Použití v nástroji:** Princip Least-Agency je přesná formulace výchozího postoje nástroje: tam, kde stačí asistent, nedávat agenta. Citace o observabilitě propojuje bezpečnost s tématem akce (Grafana AI Observability Breakfast).

**Odkaz:** https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/

**Datum ověření:** 3. 11. 2026

---

## Z-27 – OWASP Agentic AI: Threats and Mitigations

**Co zdroj pokrývá:** Průvodce OWASP Agentic Security Initiative.

**Co přesně říká:**
- T8 Repudiation & Untraceability: nastává, když akce agentů „cannot be traced back or accounted for due to insufficient logging or transparency in decision-making processes“. Opatření: komplexní logování, kryptografické ověření, obohacená metadata, monitorování v reálném čase.
- T10 Overwhelming Human in the Loop: hrozba cílí na systémy s lidským dohledem a zneužívá „human cognitive limitations“. Doporučení: adaptivní mechanismy důvěry s dynamickými prahy zásahu podle rizika, jistoty a kontextu a hierarchická spolupráce, „where low-risk decisions are automated, and human intervention is prioritized for high-risk anomalies“.

**ROZPOR (verze):** Stránka zdroje uvádí datum zveřejnění 17. 2. 2025 (verze 1.0, hrozby T1 až T15). Soubor, který stránka k datu ověření nabízí ke stažení, je „Version 1.1, December 2025“ a obsahuje hrozby T1 až T17 (přibyly T16 Insecure Inter-Agent Protocol Abuse a T17 Supply Chain Compromise). Citace výše jsou z verze 1.1.

**Použití v nástroji:** T8 je bezpečnostní zdůvodnění brány „dohledatelnost a auditní stopa“. T10 je zdůvodnění, proč úroveň „se schválením“ není automaticky bezpečná: schvalování ve velkém objemu se stává formalitou.

**Odkaz:** https://genai.owasp.org/resource/agentic-ai-threats-and-mitigations/

**Datum ověření:** 3. 11. 2026

---

## Z-28 – NIST AI Risk Management Framework 1.0 (NIST AI 100-1): relevantní podkategorie

**Co zdroj pokrývá:** NIST AI 100-1, Artificial Intelligence Risk Management Framework 1.0 (leden 2023), jen podkategorie relevantní pro dohled a provoz agentů.

**Co přesně říká:**
- MAP 3.5: „Processes for human oversight are defined, assessed, and documented in accordance with organizational policies from the GOVERN function.“
- MEASURE 2.4: „The functionality and behavior of the AI system and its components ... are monitored when in production.“
- MEASURE 2.6: systém je pravidelně hodnocen z hlediska bezpečnosti, reziduální riziko nepřekračuje toleranci a „it can fail safely“.
- MANAGE 2.4: „Mechanisms are in place and applied, and responsibilities are assigned and understood, to supersede, disengage, or deactivate AI systems that demonstrate performance or outcomes inconsistent with intended use.“
- MANAGE 4.1: plány monitorování po nasazení zahrnují mimo jiné „appeal and override, decommissioning, incident response, recovery, and change management“.
- GOVERN 1.7: procesy pro bezpečné vyřazení AI systému z provozu.

**Použití v nástroji:** Brány „vypínač“ (MANAGE 2.4, včetně toho, že je jasné, kdo ho smí použít), „rollback“ (MANAGE 4.1 recovery a change management) a „monitorování v produkci“ (MEASURE 2.4). Rámec je dobrovolný.

**Odkaz:** https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.100-1.pdf

**Datum ověření:** 3. 11. 2026

---

## Z-29 – NIST CAISI: AI Agent Standards Initiative (stav)

**Co zdroj pokrývá:** Oznámení NIST z 17. 2. 2026.

**Co přesně říká:**
- Iniciativu oznámilo Center for AI Standards and Innovation (CAISI) při NIST. Tři pilíře: podpora standardů pro agenty vyvíjených průmyslem, otevřené protokoly pro agenty vyvíjené komunitou, výzkum bezpečnosti a identity agentů.
- „In the months ahead, NIST will announce research, guidelines, and further deliverables.“ Sběr podnětů: RFI k bezpečnosti agentů (do 9. 3.) a koncepční dokument k identitě a autorizaci agentů (do 2. 4.).

**EVIDENCE-GAP:** Zda NIST od února do listopadu 2026 vydal finální dokument k agentům, se v této rešerši nepodařilo ověřit. Nástroj na NIST k agentům zatím neodkazuje jako na hotový standard.

**Použití v nástroji:** Jen jako poznámka „standardy k agentům se teprve tvoří“. Nevyvozovat z toho požadavky.

**Odkaz:** https://www.nist.gov/news-events/news/2026/02/announcing-ai-agent-standards-initiative-interoperable-and-secure

**Datum ověření:** 3. 11. 2026

---

## Z-30 – Anthropic: Building effective agents

**Co zdroj pokrývá:** Článek Anthropic Engineering, publikovaný 19. 12. 2024, autoři Erik S. a Barry Zhang. Stránka nově obsahuje poznámku, že se nástrojová krajina od prosince 2024 výrazně změnila.

**Co přesně říká:**
- Rozlišení: „Workflows are systems where LLMs and tools are orchestrated through predefined code paths. Agents, on the other hand, are systems where LLMs dynamically direct their own processes and tool usage.“
- „We recommend finding the simplest solution possible, and only increasing complexity when needed. This might mean not building agentic systems at all.“
- „The autonomous nature of agents means higher costs, and the potential for compounding errors. We recommend extensive testing in sandboxed environments, along with the appropriate guardrails.“
- Agenti mohou „pause for human feedback at checkpoints or when encountering blockers“; běžné jsou i podmínky zastavení, např. maximální počet iterací.
- Tři principy: jednoduchost návrhu, transparentnost („explicitly showing the agent’s planning steps“), pečlivě navržené rozhraní agenta k nástrojům.

**Použití v nástroji:** Rozlišení workflow vs. agent je první otázka u každého místa v provozu (předem daný postup s AI krokem, nebo AI řídí postup). Doporučení „nejjednodušší řešení“ podporuje Least-Agency (Z-26).

**Odkaz:** https://www.anthropic.com/engineering/building-effective-agents

**Datum ověření:** 3. 11. 2026

---

## Z-31 – Anthropic: Demystifying evals for AI agents

**Co zdroj pokrývá:** Článek Anthropic Engineering publikovaný 9. 1. 2026.

**Co přesně říká:**
- „An evaluation (“eval”) is a test for an AI system: give an AI an input, then apply grading logic to its output to measure success.“
- Capability evals začínají na nízké úspěšnosti, regression evals „should have a nearly 100% pass rate“. Úspěšné capability evals mohou „graduate“ do regresní sady, která běží průběžně.
- Metriky: pass@k (aspoň jeden úspěch z k pokusů) a pass^k (všech k pokusů úspěšných). Příklad z článku: při úspěšnosti 75 % na pokus a třech pokusech je pravděpodobnost, že projdou všechny tři, přibližně 42 %. pass^k „especially matters for customer-facing agents where users expect reliable behavior every time.“
- Typy hodnotitelů: kódové, modelové a lidské (lidé slouží i ke kalibraci modelových).
- „In reality, 20-50 simple tasks drawn from real failures is a great start.“
- Čtení přepisů běhů je nutné, jinak nelze poznat, jestli selhal agent, nebo hodnotitel.

**Použití v nástroji:** Brána „error rate a kvalita“: pro opakované provozní zásahy (restart služby) je správná metrika spolehlivost napříč pokusy, ne jediný úspěch. Start s 20 až 50 úlohami z reálných incidentů je realistický první krok do plánu „maximálně pět kroků hned“.

**Odkaz:** https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents

**Datum ověření:** 3. 11. 2026

---

## Z-32 – Anthropic: Measuring AI agent autonomy in practice

**Co zdroj pokrývá:** Výzkum Anthropic (tým Societal Impacts) publikovaný 18. 2. 2026, analýza interakcí v Claude Code a ve veřejném API.

**Co přesně říká:**
- „Newer users (<50 sessions) employ full auto-approve roughly 20% of the time; by 750 sessions, this increases to over 40% of sessions.“ Autoři upozorňují, že výchozí nastavení Claude Code vyžaduje ruční schválení každé akce, takže část posunu je změna konfigurace.
- Zkušení uživatelé schvalují automaticky častěji, ale také častěji přerušují běh.
- „On the most complex tasks, Claude Code stops to ask for clarification more than twice as often as humans interrupt it.“ Zastavení iniciované agentem je samostatná forma dohledu.
- Ve veřejném API: 80 % volání nástrojů pochází od agentů s aspoň jednou pojistkou (omezená oprávnění nebo lidské schválení), 73 % má nějak člověka ve smyčce a jen 0,8 % akcí se jeví jako nevratných. Riziko a autonomii odhadoval model Claude, ne lidé.
- „Our central conclusion is that effective oversight of agents will require new forms of post-deployment monitoring infrastructure and new human-AI interaction paradigms.“

**Použití v nástroji:** Empirická opora pro dvě věci: autonomie se v praxi zvyšuje postupně s důvěrou (tedy brány jako cesta, ne jednorázové rozhodnutí) a monitorování po nasazení je podmínka dohledu. Čísla jsou z produktů Anthropicu, nepřenášet je jako obecnou statistiku.

**Odkaz:** https://www.anthropic.com/research/measuring-agent-autonomy

**Datum ověření:** 3. 11. 2026

---

# F. AI Act (jen odkaz, bez výkladu)

## Z-33 – AI Act, článek 14: Lidský dohled

**Co zdroj pokrývá:** Text článku 14 nařízení (EU) 2024/1689 na portálu AI Act Service Desk (Evropská komise).

**Co přesně říká:** Stránka nese název „Article 14: Human oversight“. Obsah článku se v nástroji nevykládá.

**Použití v nástroji:** Jen odkaz. Pro posouzení povinností a jejich účinnosti odkázat na AI Act advisor (profil P3 AI Operator), kde je stav po novele (EU) 2026/1744 popsaný v jeho zdrojích Z-07 a Z-08 (`ai-act-zdroje-eu.md`).

**Odkaz:** https://ai-act-service-desk.ec.europa.eu/en/ai-act/article-14

**Datum ověření:** 3. 11. 2026

---

## Z-34 – AI Act, článek 12: Uchovávání záznamů

**Co zdroj pokrývá:** Text článku 12 nařízení (EU) 2024/1689 na portálu AI Act Service Desk.

**Co přesně říká:** Stránka nese název „Article 12: Record-keeping“. Obsah článku se v nástroji nevykládá.

**Použití v nástroji:** Jen odkaz, stejně jako Z-33. Úplné znění nařízení je na EUR-Lex (https://eur-lex.europa.eu/eli/reg/2024/1689/oj); EUR-Lex v této rešerši automatizovaně nevrátil obsah (odpověď HTTP 202 bez těla), odkaz je převzatý z oficiálního identifikátoru ELI.

**Odkaz:** https://ai-act-service-desk.ec.europa.eu/en/ai-act/article-12

**Datum ověření:** 3. 11. 2026

---

# Souhrn rozporů mezi zdroji

| Téma | Zdroj 1 | Zdroj 2 | Záznam |
|---|---|---|---|
| Agent Observability a OpenTelemetry | Tisková zpráva 20. 10. 2026: „with OpenTelemetry support (GA)“ | Dokumentace: „OpenTelemetry ingestion is in public preview“ | Z-02, Z-17 |
| Stav Assistant Investigations | What's new 29. 7. 2026 a tisková zpráva 27. 7. 2026: GA | `grafana/skills`, skill `ml-ai`: public preview | Z-08, Z-18 |
| Cesta k AI/Agent Observability | Dokumentace Grafany: vlastní SDK, OpenTelemetry, Claude Platform, coding agenti | `grafana/skills`, skill `app-observability`: OpenLIT | Z-02, Z-18 |
| Název MCP serveru | Podtitulek tiskové zprávy 27. 7. 2026: „Grafana MCP server“ | Text téže zprávy a What's new: „Grafana Cloud MCP server“ | Z-12 |
| Verze OWASP Threats and Mitigations | Stránka zdroje: 17. 2. 2025, T1 až T15 | Stažený soubor: Version 1.1, December 2025, T1 až T17 | Z-27 |
| Čísla z průzkumů Grafana Labs | „Grafana Labs’ 2026 Observability Survey“ (92 %, 57 %) | Report Grafana Labs a ETR „2026 State of Observability“ (85 %, 52 %) | Z-17 |

# Co se nepodařilo ověřit z primárního zdroje

- Stav produktu „Durable Agents“ (Grots) v Grafana Assistant a existence produktu pod názvem „Grafana Cloud Agents“ (Z-06).
- Formální datum GA pro Sift (Z-14).
- Chování politiky **Default** u vestavěných nástrojů Grafana Assistant (Z-07).
- Metodika průzkumů Grafana Labs, ze kterých pocházejí čísla v tiskových zprávách (Z-17).
- Datum vzniku a aktualizace AWS Agentic AI Security Scoping Matrix (Z-23).
- Zda NIST CAISI do 3. 11. 2026 vydal finální dokument k agentům (Z-29).
- Obsah EUR-Lex (automatizovaný přístup neprošel); články AI Actu ověřené přes AI Act Service Desk (Z-34).
