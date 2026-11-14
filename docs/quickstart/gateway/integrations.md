<!--
zdroj: maximhq/bifrost docs/quickstart/gateway/integrations.mdx
upstream-commit: 7235fb54f388679e8c4c93ce9687a79ae084700a
prelozeno: 2026-11-12
-->

# Integrace

*Bifrost použijete jako přímou náhradu stávajících SDK poskytovatelů AI bez změn kódu. Stačí změnit základní URL a získáte pokročilé funkce.*

## Co jsou integrace?

Integrace jsou protokolové adaptéry, díky kterým je Bifrost **100% kompatibilní** se stávajícími SDK poskytovatelů AI. Překládají mezi formáty API jednotlivých poskytovatelů (OpenAI, Anthropic, Google GenAI) a jednotným API Bifrostu, takže získáte:

- **Přímou náhradu**: ve stávajícím kódu změníte jen základní URL.
- **Nulovou práci s migrací**: zůstane vám vaše SDK i zpracování požadavků a odpovědí.
- **Okamžitý přístup k funkcím**: governance, cache, fallbacky a monitoring bez změn kódu.

## Rychlý příklad

### Předtím (přímo u poskytovatele)
```python
import openai

client = openai.OpenAI(
    api_key="<YOUR-OPENAI-API-KEY>"
)
```

### Potom (integrace přes Bifrost)
```python
import openai

client = openai.OpenAI(
    base_url="http://localhost:8080/openai",  # Point to Bifrost
    api_key="<YOUR-BIFROST-VIRTUAL-KEY>"  # Replace with your actual Bifrost virtual key.
)
```

> [!NOTE]
> Pokud Bifrost běží pod prefixem cesty (`server.base_path`), uveďte ho v `base_url`, například `https://example.com/bifrost/openai`. Viz [Serving under a path prefix](https://docs.getbifrost.ai/deployment-guides/how-to/nginx-reverse-proxy#serving-under-a-path-prefix).

**A je to!** Vaše aplikace teď má automatické fallbacky, governance, monitoring a všechny funkce Bifrostu.

## Dostupné integrace

Bifrost je plně kompatibilní s těmito oblíbenými AI SDK:

- **[OpenAI SDK](https://docs.getbifrost.ai/integrations/openai-sdk)**
- **[Anthropic SDK](https://docs.getbifrost.ai/integrations/anthropic-sdk)**
- **[Google GenAI SDK](https://docs.getbifrost.ai/integrations/genai-sdk)**
- **[LiteLLM](https://docs.getbifrost.ai/integrations/litellm-sdk)**
- **[LangChain](https://docs.getbifrost.ai/integrations/langchain-sdk)**
- **[Passthrough Endpoints](https://docs.getbifrost.ai/integrations/passthrough)**

## Další informace

Podrobné návody k nastavení, informace o kompatibilitě a pokročilé použití:

**➜ [Úplná dokumentace integrací](https://docs.getbifrost.ai/integrations/what-is-an-integration)**

## Další kroky

Teď, když rozumíte integracím, pokračujte souvisejícími tématy:

### Základní témata

- **[Konfigurace poskytovatelů](./provider-configuration.md)**: nastavte více poskytovatelů AI pro redundanci.
- **[Tool Calling](./tool-calling.md)**: umožněte AI modelům používat externí funkce.
- **[Streaming Responses](./streaming.md)**: generování odpovědí v reálném čase.
- **[Multimodal AI](./multimodal.md)**: zpracování obrázků, zvuku a multimediálního obsahu.

### Pokročilá témata

- **[Core Features](https://docs.getbifrost.ai/features/)**: governance, cache a observabilita.
- **[Architecture](https://docs.getbifrost.ai/architecture/)**: jak Bifrost funguje uvnitř.
- **[Deployment](https://docs.getbifrost.ai/deployment-guides)**: produkční nasazení a škálování.

---

<sub>Přeloženo z oficiální dokumentace Bifrostu (Apache 2.0), soubor `docs/quickstart/gateway/integrations.mdx`, commit `7235fb5`. Změny: překlad do češtiny, převod z MDX do Markdownu. Originál: https://docs.getbifrost.ai/quickstart/gateway/integrations</sub>
