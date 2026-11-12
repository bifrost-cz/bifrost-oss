<!--
zdroj: maximhq/bifrost docs/overview.mdx
upstream-commit: 7235fb54f388679e8c4c93ce9687a79ae084700a
prelozeno: 2026-11-12
-->

# Bifrost AI Gateway

*Nejrychlejší cesta k AI aplikacím, které nepadají. Výkonná AI brána, která sjednocuje 20+ poskytovatelů pod jedno API kompatibilní s OpenAI.*

Bifrost je výkonná AI brána, která sjednocuje přístup k 20+ poskytovatelům (OpenAI, Anthropic, AWS Bedrock, Google Vertex, Azure a dalším) přes jednotné API. Nasadíte ji za pár sekund bez jakékoli konfigurace a získáte automatické přepínání při výpadku, vyvažování zátěže, sémantickou cache a governance na úrovni enterprise. V dlouhodobých benchmarcích při 5 000 požadavcích za sekundu přidá Bifrost ke každému požadavku jen **11 µs** režie.

![Diagram architektury Bifrostu](https://raw.githubusercontent.com/maximhq/bifrost/7235fb54f388679e8c4c93ce9687a79ae084700a/docs/media/architecture.png)

## Začínáme

- **[Nastavení brány](./quickstart/gateway/setting-up.md)**: nasazení HTTP API brány s vestavěným webovým rozhraním pro vizuální konfiguraci a sledování v reálném čase.
- **[Go SDK](https://docs.getbifrost.ai/quickstart/go-sdk/setting-up)**: integrace přímo do vaší aplikace v Go pro maximální výkon a kontrolu.

---

## Funkce open source verze

- **[Drop-in Replacement](https://docs.getbifrost.ai/features/drop-in-replacement)**: stávající napojení na AI SDK nahradíte jen změnou základní URL. Kód zůstane, získáte fallbacky a governance.
- **[Automatic Fallbacks](https://docs.getbifrost.ai/features/fallbacks)**: plynulé přepínání mezi poskytovateli a modely. Když primární poskytovatel selže, Bifrost automaticky přepne na zálohu.
- **[Load Balancing](https://docs.getbifrost.ai/features/keys-management)**: chytré rozdělování API klíčů s váženým vyvažováním zátěže, filtrováním podle modelů a automatickým přepnutím při výpadku.
- **[Virtual Keys](https://docs.getbifrost.ai/features/governance/virtual-keys)**: hlavní prvek governance. Řídí oprávnění, rozpočty, limity požadavků a směrování pro každého spotřebitele.
- **[Routing](https://docs.getbifrost.ai/features/governance/routing)**: směrování požadavků na konkrétní modely, poskytovatele a klíče. Vážené strategie a automatické fallbacky.
- **[Budget & Rate Limits](https://docs.getbifrost.ai/features/governance/budget-and-limits)**: hierarchické řízení nákladů s rozpočty a limity požadavků na úrovni virtuálního klíče, týmu a zákazníka.
- **[MCP Tool Filtering](https://docs.getbifrost.ai/features/governance/mcp-tools)**: určuje, které nástroje MCP má každý virtuální klíč k dispozici, pomocí striktních seznamů povolených.
- **[Semantic Caching](https://docs.getbifrost.ai/features/semantic-caching)**: chytrá cache odpovědí podle sémantické podobnosti. Nižší náklady i latence u podobných dotazů.
- **[Built-in Observability](https://docs.getbifrost.ai/features/observability/default)**: sledování každého AI požadavku v reálném čase. Výkon, ladění problémů a analýza využití.
- **[Prometheus Metrics](https://docs.getbifrost.ai/features/observability/prometheus)**: nativní metriky pro Prometheus přes scraping nebo Push Gateway pro monitoring a alerting.
- **[OpenTelemetry](https://docs.getbifrost.ai/features/observability/otel)**: integrace OTLP pro distribuované trasování s Grafanou, New Relic, Honeycomb a dalšími.
- **[Telemetry](https://docs.getbifrost.ai/features/telemetry)**: vestavěný monitoring na bázi Promethea, který sleduje metriky na úrovni HTTP i upstream poskytovatelů.
- **[Custom Plugins](https://docs.getbifrost.ai/plugins/getting-started)**: rozšiřitelná architektura middlewaru. Pluginy v Go nebo WASM pro vlastní logiku.
- **[Mocker Plugin](https://docs.getbifrost.ai/features/plugins/mocker)**: simulace odpovědí poskytovatelů AI pro testování, vývoj a simulace.

---

## MCP Gateway

Umožněte AI modelům dynamicky objevovat a spouštět externí nástroje přes **Model Context Protocol**. Bifrost funguje jako MCP klient i server: připojuje se k externím serverům s nástroji a zpřístupňuje nástroje klientům, jako je Claude Desktop.

- **[Overview](https://docs.getbifrost.ai/mcp/overview)**: jak Bifrost integruje MCP a mění statické chatovací modely na agenty schopné jednat.
- **[Tool Execution](https://docs.getbifrost.ai/mcp/tool-execution)**: spouštění nástrojů MCP s plnou kontrolou nad schvalováním, bezpečnostní validací a průběhem konverzace.
- **[Agent Mode](https://docs.getbifrost.ai/mcp/agent-mode)**: autonomní spouštění nástrojů s nastavitelným automatickým schvalováním důvěryhodných operací.
- **[Code Mode](https://docs.getbifrost.ai/mcp/code-mode)**: AI píše Python, kterým orchestruje více nástrojů. O 50 % méně tokenů a o 40 % nižší latence.
- **[MCP Authentication](https://docs.getbifrost.ai/mcp/auth/overview)**: pět typů ověření: None, Headers, OAuth 2.0, Per-User OAuth, Per-User Headers. Odložené ověření pro jednotlivé uživatele.
- **[Bifrost as an MCP Gateway](https://docs.getbifrost.ai/mcp/overview#bifrost-as-an-mcp-gateway)**: zpřístupnění samotného Bifrostu jako MCP serveru, aby Claude Desktop, Cursor a další MCP klienti mohli používat vámi nastavené nástroje.
- **[Tool Hosting](https://docs.getbifrost.ai/mcp/tool-hosting)**: registrace vlastních nástrojů přímo ve vaší aplikaci a jejich zpřístupnění přes MCP.

---

## Funkce Enterprise

Pokročilé funkce pro týmy, které provozují produkční AI systémy ve velkém. Nasazení Enterprise zahrnuje privátní síť, vlastní bezpečnostní kontroly a governance navrženou pro spolehlivost na úrovni enterprise.

- **[Guardrails](https://docs.getbifrost.ai/enterprise/guardrails)**: bezpečnost obsahu s AWS Bedrock Guardrails, Azure Content Safety, Google Model Armor a Patronus AI pro ochranu v reálném čase.
- **[Adaptive Routing](https://docs.getbifrost.ai/enterprise/adaptive-routing/overview)**: směruje každý požadavek k nejzdravějšímu poskytovateli a klíči, zadrží klíče, které poskytovatel odmítá, a ukazuje, co udělal a proč.
- **[Clustering](https://docs.getbifrost.ai/enterprise/clustering)**: vysoká dostupnost s automatickým vyhledáváním služeb, synchronizací přes gossip a nasazováním bez výpadku.
- **[Identity Providers (Okta, Entra)](https://docs.getbifrost.ai/enterprise/advanced-governance)**: integrace OpenID Connect, governance na úrovni uživatelů, synchronizace týmů a rámce pro compliance.
- **[Role-Based Access Control](https://docs.getbifrost.ai/enterprise/rbac)**: jemně nastavitelná oprávnění s vlastními rolemi pro přístup ke všem prostředkům Bifrostu.
- **[In-VPC Deployments](https://docs.getbifrost.ai/enterprise/invpc-deployments)**: nasazení ve vaší privátní cloudové infrastruktuře s izolací VPC a rozšířenými bezpečnostními kontrolami.
- **[Audit Logs](https://docs.getbifrost.ai/enterprise/audit-logs)**: neměnná auditní stopa pro compliance se SOC 2, GDPR, HIPAA a ISO 27001.
- **[Datadog Connector](https://docs.getbifrost.ai/features/observability/datadog)**: nativní integrace s Datadogem pro trasování APM, LLM Observability a metriky.
- **[Log Exports](https://docs.getbifrost.ai/enterprise/log-exports)**: automatický export logů požadavků a telemetrie do úložišť a datových jezer.

---

## Integrace SDK

Bifrost použijete jako přímou náhradu oblíbených AI SDK bez změn kódu, stačí změnit základní URL.

- **[OpenAI SDK](https://docs.getbifrost.ai/integrations/openai-sdk/overview)**: přímá náhrada pro OpenAI SDK v Pythonu a Node.js.
- **[Anthropic SDK](https://docs.getbifrost.ai/integrations/anthropic-sdk/overview)**: přímá náhrada pro Anthropic SDK v Pythonu a TypeScriptu.
- **[Bedrock SDK](https://docs.getbifrost.ai/integrations/bedrock-sdk/overview)**: nativní integrace AWS Bedrock SDK s plnou podporou modelů.
- **[GenAI SDK](https://docs.getbifrost.ai/integrations/genai-sdk/overview)**: přímá náhrada pro Google GenAI SDK.
- **[LiteLLM](https://docs.getbifrost.ai/integrations/litellm-sdk)**: kompatibilita s proxy a SDK LiteLLM pro jednotný přístup k modelům.
- **[LangChain](https://docs.getbifrost.ai/integrations/langchain-sdk)**: integrace s frameworkem LangChain pro tvorbu AI aplikací.
- **[PydanticAI](https://docs.getbifrost.ai/integrations/pydanticai-sdk)**: integrace s PydanticAI pro typově bezpečný vývoj AI agentů.

---

## Podporovaní poskytovatelé

Bifrost podporuje 20+ poskytovatelů AI přes jedno jednotné API. Nastavíte více poskytovatelů a Bifrost se automaticky postará o směrování, přepínání při výpadku i vyvažování zátěže. Podrobné srovnání schopností najdete v [úplné tabulce podpory poskytovatelů](https://docs.getbifrost.ai/providers/supported-providers/overview).

| Poskytovatel | Popis |
|---|---|
| [OpenAI](https://docs.getbifrost.ai/providers/supported-providers/openai) | GPT-4o, o1, GPT-4 a další s plnou podporou funkcí |
| [Anthropic](https://docs.getbifrost.ai/providers/supported-providers/anthropic) | Rodina modelů Claude 4, Claude 3.5 a Claude 3 |
| [AWS Bedrock](https://docs.getbifrost.ai/providers/supported-providers/bedrock) | Přístup k více modelům s nativním ověřováním AWS |
| [Google Vertex AI](https://docs.getbifrost.ai/providers/supported-providers/vertex) | Modely Gemini a PaLM s ověřováním OAuth2 |
| [Azure OpenAI](https://docs.getbifrost.ai/providers/supported-providers/azure) | Modely OpenAI přes Azure se správou deploymentů |
| [Google Gemini](https://docs.getbifrost.ai/providers/supported-providers/gemini) | Modely Gemini s podporou obrazu, zvuku a embeddings |
| [Groq](https://docs.getbifrost.ai/providers/supported-providers/groq) | Ultrarychlá inference s hardwarovou akcelerací LPU |
| [Mistral](https://docs.getbifrost.ai/providers/supported-providers/mistral) | Modely Mistral a Mixtral s podporou nástrojů |
| [Cohere](https://docs.getbifrost.ai/providers/supported-providers/cohere) | Modely Command s chatem, embeddings a reasoningem |
| [Cerebras](https://docs.getbifrost.ai/providers/supported-providers/cerebras) | Vysokorychlostní inference s plnou podporou streamování |
| [Databricks](https://docs.getbifrost.ai/providers/supported-providers/databricks) | Model Serving a Unity AI Gateway s ověřováním PAT nebo OAuth M2M |
| [DeepSeek](https://docs.getbifrost.ai/providers/supported-providers/deepseek) | Chat, reasoning, nástroje a FIM completions kompatibilní s OpenAI |
| [Ollama](https://docs.getbifrost.ai/providers/supported-providers/ollama) | Lokální inference ve formátu kompatibilním s OpenAI |
| [Hugging Face](https://docs.getbifrost.ai/providers/supported-providers/huggingface) | Inference API s chatem, obrazem, TTS a STT |
| [OpenRouter](https://docs.getbifrost.ai/providers/supported-providers/openrouter) | Směrování k více poskytovatelům s podporou reasoningu |
| [Perplexity](https://docs.getbifrost.ai/providers/supported-providers/perplexity) | Integrace webového vyhledávání s podporou reasoningu |
| [ElevenLabs](https://docs.getbifrost.ai/providers/supported-providers/elevenlabs) | Modely pro převod textu na řeč a řeči na text |
| [Nebius](https://docs.getbifrost.ai/providers/supported-providers/nebius) | Kompatibilní s OpenAI, se streamováním a embeddings |
| [xAI](https://docs.getbifrost.ai/providers/supported-providers/xai) | Modely Grok s podporou obrazu a reasoningu |
| [Parasail](https://docs.getbifrost.ai/providers/supported-providers/parasail) | Chat a streamování s podporou volání nástrojů |
| [Replicate](https://docs.getbifrost.ai/providers/supported-providers/replicate) | Architektura založená na predikcích s asynchronními režimy |
| [SGL](https://docs.getbifrost.ai/providers/supported-providers/sgl) | Runtime SGLang se streamováním a embeddings |
| [vLLM](https://docs.getbifrost.ai/providers/supported-providers/vllm) | Vlastní provozovaná inference kompatibilní s OpenAI s chatem, embeddings a STT |

---

<sub>Přeloženo z oficiální dokumentace Bifrostu (Apache 2.0), soubor `docs/overview.mdx`, commit `7235fb5`. Změny: překlad do češtiny, převod z MDX do Markdownu (dlaždice jako seznamy a tabulka). Originál: https://docs.getbifrost.ai/overview</sub>
