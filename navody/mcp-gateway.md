<!--
druh: rychlý návod (výběr a zkrácení, ne úplný překlad)
zdroje: maximhq/bifrost docs/mcp/overview.mdx, docs/mcp/connecting-to-servers.mdx, docs/cli-agents/overview.mdx
upstream-commit: 0fb93426bea0e211963773c0a46876452869276c
sestaveno: 2026-11-14
-->

# MCP gateway

*Jak připojit MCP servery k Bifrostu a zpřístupnit jejich nástroje modelům i klientům jako Claude Desktop nebo Cursor.*

> [!NOTE]
> Zkrácený český návod: vybírá to podstatné pro rozjezd. Úplný popis je v dokumentaci výrobce: [MCP Overview](https://docs.getbifrost.ai/mcp/overview) a [Connecting to MCP Servers](https://docs.getbifrost.ai/mcp/connecting-to-servers).

## Dvě role Bifrostu

- **MCP klient:** Bifrost se připojí k MCP serverům (souborový systém, vyhledávání, databáze, interní API) a jejich nástroje nabídne modelům v požadavcích na `/v1/chat/completions`.
- **MCP server (gateway):** Bifrost vystaví všechny připojené nástroje na jednom endpointu `/mcp`. Klienti jako Claude Desktop, Claude Code nebo Cursor se připojí k bráně místo k jednotlivým serverům a uvidí jen nástroje, které jim povoluje jejich virtuální klíč.

## Bezpečnost na prvním místě

Bifrost ve výchozím stavu volání nástrojů **nespouští sám**. Model jen navrhne, co zavolat, a spuštění je samostatný krok vaší aplikace:

```
1. POST /v1/chat/completions
   → LLM returns tool call suggestions (NOT executed)

2. Your app reviews the tool calls
   → Apply security rules, get user approval if needed

3. POST /v1/mcp/tool/execute
   → Execute approved tool calls explicitly

4. POST /v1/chat/completions
   → Continue conversation with tool results
```

Automatické spouštění vybraných nástrojů umí [Agent Mode](https://docs.getbifrost.ai/mcp/agent-mode), musí se ale výslovně zapnout. Celý postup volání nástrojů popisuje [Volání nástrojů](../docs/quickstart/gateway/tool-calling.md).

## Připojení MCP serveru

| Typ | Jak funguje | Hodí se pro |
|---|---|---|
| **STDIO** | Bifrost spustí proces a komunikuje přes stdin a stdout | lokální nástroje a skripty |
| **HTTP** | Požadavky na HTTP endpoint | vzdálená API, mikroslužby |
| **SSE** | Trvalé spojení přes Server-Sent Events | streamovaná data |
| **OpenAPI** (beta) | Bifrost z OpenAPI dokumentu vytvoří nástroj pro každou operaci | REST API bez vlastního MCP serveru |

### Ve webovém rozhraní

1. V levém menu otevřete **MCP Gateway** a klikněte na **Add MCP Server**.
2. Vyplňte **Name** (bez mezer a pomlček), zvolte **Connection Type** a zadejte příkaz (STDIO) nebo URL (HTTP, SSE).
3. Klikněte na **Create**. Po připojení uvidíte načtené nástroje a můžete je jednotlivě vypínat.

### V config.json

```json
{
  "mcp": {
    "client_configs": [
      {
        "name": "filesystem",
        "connection_type": "stdio",
        "is_ping_available": true,
        "stdio_config": {
          "command": "npx",
          "args": ["-y", "@anthropic/mcp-filesystem"],
          "envs": ["HOME", "PATH"]
        },
        "tools_to_execute": ["*"]
      },
      {
        "name": "web_search",
        "connection_type": "http",
        "connection_string": "env.WEB_SEARCH_MCP_URL",
        "is_ping_available": false,
        "tools_to_execute": ["search", "fetch_url"]
      },
      {
        "name": "database",
        "connection_type": "sse",
        "connection_string": "https://db-mcp.example.com/sse",
        "is_ping_available": true,
        "tools_to_execute": []
      }
    ]
  }
}
```

- `tools_to_execute` určuje, které nástroje serveru jsou k dispozici: `["*"]` všechny, výčet jen vybrané, prázdné pole žádný.
- Citlivé hodnoty (URL s klíčem, hlavičky) dávejte do proměnných prostředí přes `env.NAZEV`.

> [!WARNING]
> Pokud Bifrost běží v Dockeru, STDIO servery potřebují v kontejneru příslušné příkazy (`npx`, `python`). Když tam nejsou, STDIO spojení nebude fungovat. Postavte si vlastní image s potřebnými závislostmi, nebo použijte MCP servery přes HTTP či SSE.

Přes API je přidání STDIO serveru a HTTP serveru s privátní adresou odmítnuto, dokud není zapnuté [ověřování dashboardu](../docs/quickstart/gateway/setting-up-auth.md). V `config.json` toto omezení neplatí.

## Bifrost jako MCP server

Endpoint `/mcp` přijímá JSON-RPC 2.0 (POST) a SSE (GET):

```bash
# List available tools
curl -X POST http://localhost:8080/mcp \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/list"
  }'

# Call a tool
curl -X POST http://localhost:8080/mcp \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 2,
    "method": "tools/call",
    "params": {
      "name": "filesystem_read_file",
      "arguments": {
        "path": "/tmp/test.txt"
      }
    }
  }'
```

Claude Desktop s virtuálními klíči pro produkci a vývoj:

```json
{
  "mcpServers": {
    "bifrost-production": {
      "url": "http://localhost:8080/mcp",
      "headers": {
        "Authorization": "Bearer vk_your_production_key"
      }
    },
    "bifrost-development": {
      "url": "http://localhost:8080/mcp",
      "headers": {
        "Authorization": "Bearer vk_your_development_key"
      }
    }
  }
}
```

Claude Code:

```bash
claude mcp add-json bifrost '{"type":"http","url":"http://localhost:8080/mcp","headers":{"Authorization":"Bearer sk-bf-your-virtual-key"}}'
```

Bez virtuálního klíče a s vypnutým vynucením ověřování vidí klient všechny nástroje. S virtuálním klíčem vrací `tools/list` jen nástroje, které klíč povoluje, a `tools/call` se kontroluje proti stejnému seznamu. Nastavení popisuje [MCP Tool Filtering](https://docs.getbifrost.ai/features/governance/mcp-tools).

> [!TIP]
> Plánujete tři a víc MCP serverů? Podívejte se na [Code Mode](https://docs.getbifrost.ai/mcp/code-mode). Model místo stovek definic nástrojů píše krátký kód, který nástroje volá, a podle výrobce tím výrazně klesne spotřeba vstupních tokenů.

## Další kroky

- **[Virtuální klíče](./virtualni-klice.md):** každý klient nebo prostředí s vlastním klíčem a vlastní sadou nástrojů.
- **[CLI agenti](./cli-agenti.md):** Claude Code a Codex CLI přes Bifrost.
- **Podrobnosti u výrobce:** [Agent Mode](https://docs.getbifrost.ai/mcp/agent-mode), [Code Mode](https://docs.getbifrost.ai/mcp/code-mode), [ověřování MCP serverů](https://docs.getbifrost.ai/mcp/auth/overview) (OAuth, hlavičky po uživatelích), [Tool Filtering](https://docs.getbifrost.ai/mcp/filtering), [OpenAPI servery](https://docs.getbifrost.ai/mcp/openapi-servers).

---

<sub>Zkrácený návod, který sestavil think private z oficiální dokumentace Bifrostu (Apache 2.0), soubory `docs/mcp/overview.mdx`, `docs/mcp/connecting-to-servers.mdx` a `docs/cli-agents/overview.mdx`, commit `0fb9342`. Změny: výběr a zkrácení obsahu, překlad do češtiny, převod z MDX do Markdownu. Kód je převzatý beze změny. Originály: https://docs.getbifrost.ai/mcp/overview a https://docs.getbifrost.ai/mcp/connecting-to-servers</sub>
