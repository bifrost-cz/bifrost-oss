<!--
zdroj: maximhq/bifrost docs/quickstart/gateway/tool-calling.mdx
upstream-commit: 0fb93426bea0e211963773c0a46876452869276c
prelozeno: 2026-11-14
-->

# Volání nástrojů

*Modely AI mohou používat externí funkce a služby: definujete schémata nástrojů, nebo připojíte servery Model Context Protocol (MCP). AI tak může pracovat s databázemi, API, souborovými systémy a dalšími zdroji.*

## Volání funkcí s vlastními nástroji

Modely AI mohou používat externí funkce, když jim schémata nástrojů definujete ve formátu OpenAI. Modely pak tyto funkce volají automaticky podle požadavků uživatele.

```bash
curl --location 'http://localhost:8080/v1/chat/completions' \
--header 'Content-Type: application/json' \
--data '{
    "model": "openai/gpt-4o-mini",
    "messages": [
        {"role": "user", "content": "What is 15 + 27? Use the calculator tool."}
    ],
    "tools": [
        {
            "type": "function",
            "function": {
                "name": "calculator",
                "description": "A calculator tool for basic arithmetic operations",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "operation": {
                            "type": "string",
                            "description": "The operation to perform",
                            "enum": ["add", "subtract", "multiply", "divide"]
                        },
                        "a": {
                            "type": "number",
                            "description": "The first number"
                        },
                        "b": {
                            "type": "number",
                            "description": "The second number"
                        }
                    },
                    "required": ["operation", "a", "b"]
                }
            }
        }
    ],
    "tool_choice": "auto"
}'
```

**Odpověď obsahuje volání nástrojů:**

```json
{
    "choices": [{
        "message": {
            "role": "assistant", 
            "tool_calls": [{
                "id": "call_abc123",
                "type": "function",
                "function": {
                    "name": "calculator",
                    "arguments": "{\"operation\":\"add\",\"a\":15,\"b\":27}"
                }
            }]
        }
    }]
}
```

## Připojení serverů MCP

Připojte servery Model Context Protocol (MCP) a modely AI získají přístup k externím nástrojům a službám, aniž byste museli každou funkci definovat ručně.

### Přes webové rozhraní

![Rozhraní pro konfiguraci MCP](https://raw.githubusercontent.com/maximhq/bifrost/0fb93426bea0e211963773c0a46876452869276c/docs/media/ui-mcp-new-server.png)

1. Otevřete **http://localhost:8080**.
2. V postranním panelu přejděte na **"MCP Clients"**.
3. Klikněte na **"Add MCP Client"**.
4. Zadejte údaje o serveru a uložte.

### Přes API

```bash
curl --location 'http://localhost:8080/api/mcp/client' \
--header 'Content-Type: application/json' \
--data '{
    "name": "filesystem",
    "connection_type": "stdio",
    "stdio_config": {
        "command": ["npx", "@modelcontextprotocol/server-filesystem", "/tmp"],
        "args": []
    }
}'
```

**Výpis nastavených klientů MCP:**

```bash
curl --location 'http://localhost:8080/api/mcp/clients'
```

### Přes config.json

```json
{
    "mcp": {
        "client_configs": [
            {
                "name": "filesystem",
                "connection_type": "stdio",
                "stdio_config": {
                    "command": ["npx", "@modelcontextprotocol/server-filesystem", "/tmp"],
                    "args": []
                }
            },
            {
                "name": "youtube-search",
                "connection_type": "http",
                "connection_string": "http://your-youtube-mcp-url"
            }
        ]
    }
}
```

Více o připojeních MCP a pokročilém spouštění nástrojů od začátku do konce najdete v části [MCP Features](https://docs.getbifrost.ai/mcp/overview).

## Možnosti volby nástroje

Určete, jak má AI nástroje používat:

```bash
# Force use of specific tool
"tool_choice": {
    "type": "function",
    "function": {"name": "calculator"}
}

# Let AI decide automatically (default)
"tool_choice": "auto"

# Disable tool usage
"tool_choice": "none"
```

## Další kroky

Když už rozumíte volání nástrojů, podívejte se na související témata:

### Základní témata

- **[Multimodální AI](./multimodal.md)**: zpracování obrázků, zvuku a multimediálního obsahu.
- **[Streamování odpovědí](./streaming.md)**: generování odpovědí s voláním nástrojů v reálném čase.
- **[Konfigurace poskytovatelů](./provider-configuration.md)**: více poskytovatelů pro redundanci.
- **[Integrace](./integrations.md)**: přímá kompatibilita se stávajícími SDK.

### Pokročilá témata

- **[MCP Features](https://docs.getbifrost.ai/mcp/overview)**: pokročilá správa a konfigurace serverů MCP.
- **[Core Features](https://docs.getbifrost.ai/features/drop-in-replacement)**: pokročilé schopnosti Bifrostu.
- **[Architecture](https://docs.getbifrost.ai/architecture/core/request-flow)**: jak Bifrost funguje uvnitř.

---

<sub>Přeloženo z oficiální dokumentace Bifrostu (Apache 2.0), soubor `docs/quickstart/gateway/tool-calling.mdx`, commit `0fb9342`. Změny: překlad do češtiny, převod z MDX do Markdownu (záložky jako podnadpisy). Originál: https://docs.getbifrost.ai/quickstart/gateway/tool-calling</sub>
