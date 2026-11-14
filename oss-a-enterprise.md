<!--
druh: přehled (sestaveno z ceníku a dokumentace výrobce, ne překlad stránky)
zdroje: maximhq/bifrost docs/overview.mdx, docs/enterprise/overview.mdx, docs/deployment-guides/overview.mdx, docs/features/governance/budget-and-limits.mdx, docs/mcp/virtual-mcps.mdx, docs/mcp/auth/overview.mdx, docs/features/observability/prometheus.mdx
web: https://www.getmaxim.ai/bifrost/pricing (stav 2026-11-14, nehlídá se automaticky)
upstream-commit: 0fb93426bea0e211963773c0a46876452869276c
sestaveno: 2026-11-14
-->

# Bifrost OSS a Enterprise: co je v které edici

*Přehled funkcí podle oblastí, s odkazem na zdroj u každé položky.*

> [!NOTE]
> Přehled sestavil think private z [ceníku výrobce](https://www.getmaxim.ai/bifrost/pricing) a z oficiální dokumentace Bifrostu (v2.5, stav k 14. 11. 2026). Nejde o oficiální srovnání výrobce. Rozhoduje vždy aktuální stav u výrobce. Ceny neuvádíme: OSS je zdarma, Enterprise má individuální cenu a [14denní zkušební verzi](https://www.getmaxim.ai/bifrost/enterprise).

## Ve zkratce

- **Enterprise je nadmnožina OSS.** Všechno z OSS v Enterprise funguje stejně, se stejným `config.json`. Přechod nevyžaduje změny v aplikacích.
- **OSS** je plnohodnotná brána pro jednu instanci: jedno API pro všechny poskytovatele, fallbacky, virtuální klíče, rozpočty, MCP gateway, cache, observabilita a vlastní pluginy.
- **Enterprise** přidává to, co potřebuje organizace: přihlášení přes firemní identitu a role, guardrails, audit, cluster s vysokou dostupností, chytřejší směrování, napojení na trezory tajemství, exporty, Bifrost Edge pro koncová zařízení a podporu se SLA.

Podle výrobce OSS stačí startupům a středně velkým týmům a jedna instance zvládne kolem 3 000 až 5 000 požadavků za sekundu. Hranicí je hlavně provoz na více uzlech: OSS drží stav v paměti a synchronizaci mezi uzly má jen Enterprise ([Budget and Limits](https://docs.getbifrost.ai/features/governance/budget-and-limits)).

Legenda: ✅ obsaženo, ❌ není, ➕ rozšířeno oproti OSS.

## Brána a poskytovatelé

| Funkce | OSS | Enterprise | Podrobnosti |
|---|:-:|:-:|---|
| Jedno API kompatibilní s OpenAI, 20+ poskytovatelů včetně lokálních modelů | ✅ | ✅ | [Konfigurace poskytovatelů](docs/quickstart/gateway/provider-configuration.md) |
| Náhrada SDK bez změny kódu (drop-in) | ✅ | ✅ | [Integrace](docs/quickstart/gateway/integrations.md) |
| Fallbacky, opakování, vyvažování zátěže | ✅ | ✅ | [Fallbacky a vyvažování](navody/fallbacky-a-vyvazovani.md) |
| Pravidla směrování (Routing Rules) | ✅ | ✅ | [Routing Rules](https://docs.getbifrost.ai/providers/routing-rules) |
| Adaptivní směrování podle zdraví poskytovatelů a klíčů | ❌ | ✅ | [Adaptive Routing](https://docs.getbifrost.ai/enterprise/adaptive-routing/overview) |
| Circuit breaker | ❌ | ✅ | [Circuit Breaker](https://docs.getbifrost.ai/enterprise/circuit-breaker) |

## Governance a náklady

| Funkce | OSS | Enterprise | Podrobnosti |
|---|:-:|:-:|---|
| Virtuální klíče | ✅ | ➕ přihlášení i tokenem z IdP | [Virtuální klíče](navody/virtualni-klice.md) |
| Rozpočty a limity, týmy a zákazníci | ✅ | ➕ | [Rozpočty a limity](navody/rozpocty-a-limity.md) |
| Obchodní jednotky, tým u více zákazníků | ❌ | ✅ | [Teams, Business Units & Customers](https://docs.getbifrost.ai/enterprise/teams-business-units-customers) |
| Přístupové profily (sdílené politiky pro uživatele, role a týmy) | ❌ | ✅ | [Access Profiles](https://docs.getbifrost.ai/enterprise/access-profiles) |
| Projekty (rozpočet a přístup podle hlavičky požadavku) | ❌ | ✅ | [Projects](https://docs.getbifrost.ai/enterprise/projects) |
| Žádosti o navýšení rozpočtu se schvalováním | ❌ | ✅ | [Budget and Limits](https://docs.getbifrost.ai/features/governance/budget-and-limits) |
| Upozornění (Slack, Teams, PagerDuty, webhook) | ❌ | ✅ | [Alerting](https://docs.getbifrost.ai/enterprise/alerting/overview) |

## Identita a přístup

| Funkce | OSS | Enterprise | Podrobnosti |
|---|:-:|:-:|---|
| Přihlášení do dashboardu jménem a heslem | ✅ | ✅ | [Nastavení ověřování](docs/quickstart/gateway/setting-up-auth.md) |
| SSO přes OIDC a SAML (Okta, Entra, Keycloak, Zitadel, Google Workspace) | ❌ | ✅ | [User Provisioning](https://docs.getbifrost.ai/enterprise/user-provisioning) |
| Synchronizace uživatelů a skupin (SCIM) | ❌ | ✅ | [User Provisioning](https://docs.getbifrost.ai/enterprise/user-provisioning) |
| Role a oprávnění (RBAC) | ❌ | ✅ | [RBAC](https://docs.getbifrost.ai/enterprise/rbac) |
| Omezení viditelnosti dat podle týmu nebo zákazníka | ❌ | ✅ | [Data Access Control](https://docs.getbifrost.ai/enterprise/data-access-control) |
| Zobrazení dashboardu očima jiného uživatele | ❌ | ✅ | [View as User](https://docs.getbifrost.ai/enterprise/view-as-user) |
| Workload identity (CI a cloudové služby bez uloženého klíče) | ❌ | ✅ | [Workload Identity](https://docs.getbifrost.ai/enterprise/workload-identity/overview) |
| Identity-aware proxy (AWS ALB, Cloudflare Access) | ❌ | ✅ | [Auth Proxy](https://docs.getbifrost.ai/enterprise/auth-proxy/generic-proxy) |

## Bezpečnost a soulad

| Funkce | OSS | Enterprise | Podrobnosti |
|---|:-:|:-:|---|
| Guardrails: osobní údaje, tajemství, vlastní regulární výrazy, externí poskytovatelé | ❌ | ✅ | [Guardrails](https://docs.getbifrost.ai/enterprise/guardrails) |
| Redakce citlivých údajů v požadavcích a logech | ❌ | ✅ | [Redaction](https://docs.getbifrost.ai/enterprise/guardrails/redaction) |
| Trezory tajemství (Vault a cloudové trezory) | ❌ | ✅ | [Secret Management](https://docs.getbifrost.ai/enterprise/secret-management) |
| Auditní logy změn konfigurace | ❌ | ✅ | [Audit Logs](https://docs.getbifrost.ai/enterprise/audit-logs) |

## MCP

| Funkce | OSS | Enterprise | Podrobnosti |
|---|:-:|:-:|---|
| MCP gateway, Agent Mode, Code Mode | ✅ | ✅ | [MCP gateway](navody/mcp-gateway.md) |
| Ověřování MCP serverů (hlavičky, OAuth, po uživatelích) | ✅ | ✅ | [MCP Authentication](https://docs.getbifrost.ai/mcp/auth/overview) |
| Token Exchange (federované ověřování MCP pod identitou volajícího) | ❌ | ✅ | [Token Exchange](https://docs.getbifrost.ai/mcp/auth/token-exchange) |
| Virtual MCPs (sady nástrojů na `/mcp/<slug>`) | ✅ | ➕ i přes přístupové profily | [Virtual MCPs](https://docs.getbifrost.ai/mcp/virtual-mcps) |

## Observabilita

| Funkce | OSS | Enterprise | Podrobnosti |
|---|:-:|:-:|---|
| Logy požadavků, Prometheus, OpenTelemetry | ✅ | ✅ | [Observabilita](navody/observabilita.md) |
| Přístup k `/metrics` přes API klíč | ❌ | ✅ | [Prometheus](https://docs.getbifrost.ai/features/observability/prometheus) |
| Nativní exporty do Datadogu, Splunku, Kafky, BigQuery a Pub/Sub | ❌ | ✅ | [Datadog](https://docs.getbifrost.ai/features/observability/datadog) |
| Export logů do S3, GCS a datových jezer | ❌ | ✅ | [Log Exports](https://docs.getbifrost.ai/enterprise/log-exports) |

## Prompty

| Funkce | OSS | Enterprise | Podrobnosti |
|---|:-:|:-:|---|
| Repozitář promptů a playground | ✅ | ✅ | [Playground](https://docs.getbifrost.ai/features/prompt-repository/playground) |
| Nasazování verzí promptů s pravidly a vahami | ❌ | ✅ | [Prompt Deployments](https://docs.getbifrost.ai/enterprise/prompt-deployments) |

## Provoz a nasazení

| Funkce | OSS | Enterprise | Podrobnosti |
|---|:-:|:-:|---|
| Docker, Kubernetes a Helm, binárka | ✅ | ✅ | [Docker a Helm](navody/nasazeni-docker-a-helm.md) |
| SQLite | ✅ | ❌ jen PostgreSQL 16+ | [Deploy Bifrost](https://docs.getbifrost.ai/deployment-guides/overview) |
| Více instancí | jen se stejným souborem `config.json`, bez sdíleného stavu | ✅ cluster se synchronizací stavu | [Clustering](https://docs.getbifrost.ai/enterprise/clustering) |
| Nasazení ve VPC, on-premise i bez internetu s podporou výrobce | ❌ | ✅ | [In-VPC Deployments](https://docs.getbifrost.ai/enterprise/invpc-deployments) |
| Bifrost Edge: agent na počítačích, který směruje AI provoz všech aplikací přes bránu | ❌ | ✅ | [Bifrost Edge](https://docs.getbifrost.ai/edge/overview) |
| Vlastní pluginy | ✅ | ✅ | [Pluginy v kostce](navody/pluginy.md) |
| Podpora | komunita | ✅ se SLA, onboarding | [Ceník](https://www.getmaxim.ai/bifrost/pricing) |

OSS si samozřejmě můžete provozovat kdekoli sami, i on-premise a bez internetu. Řádek o nasazení ve VPC se týká Enterprise nasazení s podporou výrobce.

## Kde se zdroje liší

- **Virtual MCPs:** přehled Enterprise v dokumentaci je řadí mezi funkce Enterprise, stránka [Virtual MCPs](https://docs.getbifrost.ai/mcp/virtual-mcps) je ale popisuje jako součást OSS (dřív se jmenovaly MCP Tool Groups). Enterprise je navíc umí přidělit přes přístupové profily.
- **OpenTelemetry:** ceník uvádí „External OTel Connectors“ jen u Enterprise. Podle dokumentace je export přes OpenTelemetry do vlastního kolektoru součástí OSS, jen u Enterprise jsou nativní konektory jako Datadog.

Tabulka výše se v obou případech řídí dokumentací.

## Potřebujete Enterprise, nebo poradit?

[think private](https://thinkprivate.ai/bifrost) je partnerem pro Bifrost v Česku a na Slovensku. Pomůžeme rozhodnout, jestli vám stačí OSS, a s nasazením obou edic. 👉 **[Nezávazná konzultace](https://thinkprivate.ai/bifrost)** · info@thinkprivate.ai

---

<sub>Přehled sestavil think private z ceníku výrobce (https://www.getmaxim.ai/bifrost/pricing, stav 14. 11. 2026) a z oficiální dokumentace Bifrostu (Apache 2.0, commit `0fb9342`), mimo jiné `docs/overview.mdx` a `docs/enterprise/overview.mdx`. Nejde o oficiální dokument Maxim AI.</sub>
