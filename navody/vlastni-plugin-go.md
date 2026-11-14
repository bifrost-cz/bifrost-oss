<!--
druh: rychlý návod (výběr a zkrácení, ne úplný překlad)
zdroje: maximhq/bifrost docs/plugins/writing-go-plugin.mdx, docs/plugins/building-dynamic-binary.mdx
upstream-commit: 0fb93426bea0e211963773c0a46876452869276c
sestaveno: 2026-11-14
-->

# Vlastní plugin v Go

*Od prázdné složky k pluginu, který Bifrost načte a zavolá při každém požadavku.*

> [!NOTE]
> Zkrácený český návod: vybírá to podstatné pro rozjezd. Úplný popis všech hooků a pokročilých vzorů je v [dokumentaci výrobce](https://docs.getbifrost.ai/plugins/writing-go-plugin). Přehled pluginů a pořadí hooků najdete v návodu [Pluginy v kostce](./pluginy.md).

## Než začnete

Go plugin je sdílená knihovna (`.so`), kterou Bifrost načte za běhu. Go je u toho velmi přísné. Plugin a Bifrost se musí shodovat ve všem:

| Co | Musí sedět | Jak zjistit |
|---|---|---|
| Verze Go | přesně (teď 1.27.3) | `go version` |
| Verze `github.com/maximhq/bifrost/core` | přesně | `transports/go.mod` vydané verze Bifrostu |
| Ostatní závislosti, které plugin sdílí s Bifrostem | přesně | `go version -m` u pluginu i binárky |
| Operační systém | Linux nebo macOS (Windows ne) | |
| Architektura | `amd64` nebo `arm64`, stejná | `uname -m` |
| libc | musl (Alpine), nebo glibc (Debian, Ubuntu, RHEL), stejná | `ldd` |

A Bifrost musí být **dynamicky linkovaný**. Oficiální image to není, viz [Dynamická image](./dynamicka-image.md).

> [!TIP]
> **Doplněk think private:** verzi Go a `core`, proti které máte plugin sestavit, zjistíte nejspolehlivěji z tagu vydání, které provozujete. Vydání brány mají tagy `transports/vX.Y.Z`:

<!-- kód: think private -->
```bash
# Verze Bifrostu, kterou provozujete (příklad)
BIFROST_VERSION=2.5.1
git clone --depth 1 --branch transports/v$BIFROST_VERSION https://github.com/maximhq/bifrost bifrost-src
grep -E '^go |maximhq/bifrost/core ' bifrost-src/transports/go.mod
```

Pro `transports/v2.5.1` vypíše `go 1.27.3` a `github.com/maximhq/bifrost/core v1.15.2`. Stejný klon použijete i pro sestavení dynamického Bifrostu.

## 1. Projekt

```
hello-world/
├── main.go          # Plugin implementation
├── go.mod           # Go module definition
├── go.sum           # Dependency checksums
├── Makefile         # Build automation
└── .gitignore       # Git ignore patterns
```

```bash
mkdir my-plugin
cd my-plugin
go mod init github.com/yourusername/my-plugin
```

Závislost na Bifrostu přidejte **v přesné verzi** z předchozího kroku, ne `@latest` z ukázky výrobce (doplněk think private):

<!-- kód: think private -->
```bash
go get github.com/maximhq/bifrost/core@v1.15.2
go mod tidy
```

`go.mod` pak vypadá takto (s vaší verzí místo ukázkové):

```go
module github.com/yourusername/my-plugin

go 1.27.3

require github.com/maximhq/bifrost/core v1.2.38
```

## 2. Kód pluginu

Ukázka výrobce `hello-world` se všemi hooky. Pro vlastní plugin si nechte jen ty, které potřebujete (povinné jsou `GetName` a `Cleanup`):

```go
package main

import (
	"fmt"

	"github.com/maximhq/bifrost/core/schemas"
)

// Init is called when the plugin is loaded
// config contains the plugin configuration from config.json
func Init(config any) error {
	fmt.Println("Init called")
	// Initialize your plugin here (database connections, API clients, etc.)
	return nil
}

// GetName returns the plugin's unique identifier
func GetName() string {
	return "Hello World Plugin"
}

// HTTPTransportPreHook intercepts requests BEFORE they enter Bifrost core
// Modify req in-place. Return (*HTTPResponse, nil) to short-circuit.
// Only called when using HTTP transport (bifrost-http)
func HTTPTransportPreHook(ctx *schemas.BifrostContext, req *schemas.HTTPRequest) (*schemas.HTTPResponse, error) {
	// Use ctx.Log for structured, scoped logging (recommended)
	ctx.Log(schemas.LogLevelInfo, "HTTPTransportPreHook called")

	// Read headers using case-insensitive helper (recommended)
	contentType := req.CaseInsensitiveHeaderLookup("Content-Type")
	fmt.Printf("Content-Type: %s\n", contentType)

	// Modify request in-place
	req.Headers["x-custom-header"] = "custom-value"

	// Store values in context for use in other hooks
	ctx.SetValue(schemas.BifrostContextKey("my-plugin-key"), "pre-hook-value")

	// Return nil to continue, or return &schemas.HTTPResponse{} to short-circuit
	return nil, nil
}

// HTTPTransportPostHook intercepts responses AFTER they exit Bifrost core
// Modify resp in-place. Called in reverse order of pre-hooks.
// Streaming post-processing may run after headers are committed; use
// HTTPTransportResponseHeadersHook for headers that must reach streaming clients.
func HTTPTransportPostHook(ctx *schemas.BifrostContext, req *schemas.HTTPRequest, resp *schemas.HTTPResponse) error {
	ctx.Log(schemas.LogLevelInfo, "HTTPTransportPostHook called")

	// Read context values set in pre-hook
	if val := ctx.Value(schemas.BifrostContextKey("my-plugin-key")); val != nil {
		fmt.Printf("Context value: %v\n", val)
	}

	// Return nil to continue, or return error to short-circuit
	return nil
}

// HTTPTransportResponseHeadersHook runs once at the last safe point before headers
// are committed. Use it for headers that must also be present on streaming responses.
func HTTPTransportResponseHeadersHook(ctx *schemas.BifrostContext, req *schemas.HTTPRequest, resp *schemas.HTTPResponseMetadata) error {
	resp.SetHeader("x-processed-by", "my-plugin")
	return nil
}

// HTTPTransportStreamChunkHook intercepts streaming chunks BEFORE they're sent to the client
// Modify chunk or return nil to skip. Called in reverse order of pre-hooks.
// Only called for STREAMING responses when using HTTP transport (bifrost-http)
func HTTPTransportStreamChunkHook(ctx *schemas.BifrostContext, req *schemas.HTTPRequest, chunk *schemas.BifrostStreamChunk) (*schemas.BifrostStreamChunk, error) {
	ctx.Log(schemas.LogLevelInfo, "HTTPTransportStreamChunkHook called")

	// chunk is a typed struct containing one of:
	// - BifrostTextCompletionResponse (text completion streaming)
	// - BifrostChatResponse (chat completion streaming)
	// - BifrostResponsesStreamResponse (responses API streaming)
	// - BifrostSpeechStreamResponse (speech synthesis streaming)
	// - BifrostTranscriptionStreamResponse (transcription streaming)
	// - BifrostImageGenerationStreamResponse (image generation streaming)
	// - BifrostError (error during streaming)

	// Return chunk unchanged to pass through
	return chunk, nil

	// Or return nil to skip/filter this chunk:
	// return nil, nil

	// Or return modified chunk:
	// modifiedChunk := &schemas.BifrostStreamChunk{BifrostChatResponse: ...}
	// return modifiedChunk, nil
}


// PreRequestHook is called once per top-level request (NOT per fallback attempt).
// This is the routing phase — use it for provider/model/fallback decisions.
// Mutations to req.Provider/req.Model/req.Fallbacks commit and propagate to every attempt.
// Errors are non-blocking (logged + skipped).
func PreRequestHook(ctx *schemas.BifrostContext, req *schemas.BifrostRequest) error {
	ctx.Log(schemas.LogLevelInfo, "PreRequestHook called")
	// Plugins that don't participate in routing should just return nil
	return nil
}

// PreLLMHook is called before the request is sent to the provider
// This is where you can modify requests or short-circuit the flow
func PreLLMHook(ctx *schemas.BifrostContext, req *schemas.BifrostRequest) (*schemas.BifrostRequest, *schemas.LLMPluginShortCircuit, error) {
	ctx.Log(schemas.LogLevelInfo, "PreLLMHook called")
	// Modify the request or return a short-circuit to skip provider call
	return req, nil, nil
}

// PostLLMHook is called after receiving a response from the provider
// This is where you can modify responses or handle errors
func PostLLMHook(ctx *schemas.BifrostContext, resp *schemas.BifrostResponse, bifrostErr *schemas.BifrostError) (*schemas.BifrostResponse, *schemas.BifrostError, error) {
	ctx.Log(schemas.LogLevelInfo, "PostLLMHook called")
	// Modify the response or error before returning to caller
	return resp, bifrostErr, nil
}

// Cleanup is called when Bifrost shuts down
func Cleanup() error {
	fmt.Println("Cleanup called")
	// Clean up resources (close connections, flush buffers, etc.)
	return nil
}
```

Nejdůležitější hooky:

- `Init(config any)`: jednou při načtení, dostane `config` z konfigurace pluginu. Tady načtěte nastavení, otevřete spojení, zkontrolujte proměnné prostředí.
- `PreLLMHook`: před voláním poskytovatele. Může požadavek upravit, nebo vrátit `LLMPluginShortCircuit` s vlastní odpovědí a poskytovatele vůbec nevolat:

```go
func PreLLMHook(ctx *schemas.BifrostContext, req *schemas.BifrostRequest) (*schemas.BifrostRequest, *schemas.LLMPluginShortCircuit, error) {
	// Return cached response without calling provider
	if cachedResponse := checkCache(req) {
		return req, &schemas.LLMPluginShortCircuit{
			Response: cachedResponse,
		}, nil
	}
	return req, nil, nil
}
```

- `PostLLMHook`: po odpovědi poskytovatele, úprava odpovědi nebo chyby.
- `HTTPTransportPreHook`: hlavičky, tělo a parametry HTTP požadavku. Hlavičky čtěte přes `req.CaseInsensitiveHeaderLookup`.
- `ctx.Log(...)`: strukturované logování s názvem pluginu, místo `fmt.Println`.

## 3. Sestavení

Plugin se sestavuje s `-buildmode=plugin`. Makefile z ukázky výrobce:

```makefile
.PHONY: all build clean install help

PLUGIN_NAME = my-plugin
OUTPUT_DIR = build

# Platform detection
UNAME_S := $(shell uname -s)
ifeq ($(UNAME_S),Linux)
	PLUGIN_EXT = .so
	PLATFORM = linux
endif
ifeq ($(UNAME_S),Darwin)
	PLUGIN_EXT = .so
	PLATFORM = darwin
endif

# Architecture detection
UNAME_M := $(shell uname -m)
ifeq ($(UNAME_M),x86_64)
	ARCH = amd64
endif
ifeq ($(UNAME_M),arm64)
	ARCH = arm64
endif

OUTPUT = $(OUTPUT_DIR)/$(PLUGIN_NAME)$(PLUGIN_EXT)

build: ## Build the plugin for current platform
	@echo "Building plugin for $(PLATFORM)/$(ARCH)..."
	@mkdir -p $(OUTPUT_DIR)
	go build -buildmode=plugin -o $(OUTPUT) main.go
	@echo "Plugin built successfully: $(OUTPUT)"

clean: ## Remove build artifacts
	@rm -rf $(OUTPUT_DIR)

install: build ## Build and install to Bifrost plugins directory
	@mkdir -p ~/.bifrost/plugins
	@cp $(OUTPUT) ~/.bifrost/plugins/
	@echo "Plugin installed to ~/.bifrost/plugins/"
```

```bash
make build
```

> [!WARNING]
> Křížová kompilace pluginů nefunguje. Plugin pro Linux sestavte na Linuxu, nebo v Dockeru se stejným základním image jako Bifrost (viz [Dynamická image](./dynamicka-image.md#plugin-sestavte-ve-stejném-prostředí)).

## 4. Načtení v Bifrostu

```json
{
  "plugins": [
    {
      "enabled": true,
      "name": "my-plugin",
      "path": "/path/to/my-plugin.so",
      "config": {
        "api_key": "your-api-key",
        "custom_setting": "value"
      }
    }
  ]
}
```

- `enabled`: plugin se načte, jen když je `true`,
- `name`: název v logu,
- `path`: cesta k `.so`, absolutní nebo relativní,
- `config`: libovolný objekt, který dostane `Init()`.

Po startu brány uvidíte v logu `Init called` a při prvním požadavku volání jednotlivých hooků.

## Když se plugin nenačte

| Chyba | Příčina | Řešení |
|---|---|---|
| `plugin: not a plugin file` | sestaveno bez `-buildmode=plugin` | sestavit znovu s tímto přepínačem |
| `plugin was built with a different version of package ...` | jiná verze Go, `core` nebo jiné sdílené závislosti, případně jiná architektura | porovnat `go version -m` (níže), srovnat verze, sestavit znovu |
| `error while loading shared libraries: libc.musl-...` | plugin nebo binárka pro jinou libc | sestavit ve stejném základním image |
| `symbol ... not found` | chybí export nebo má špatnou signaturu | exportované funkce začínají velkým písmenem |
| `realpath failed: no such file` | špatná `path` v konfiguraci | použít absolutní cestu |

Porovnání verzí pluginu a binárky:

```bash
# Check what versions your plugin was built with:
$ go version -m my-plugin.so
my-plugin.so: go1.27.3
  dep  github.com/maximhq/bifrost/core  v1.14.0
  dep  github.com/valyala/fasthttp      v1.51.0

# Check what versions Bifrost was built with:
$ go version -m bifrost-http
bifrost-http: go1.27.3
  dep  github.com/maximhq/bifrost/core  v1.14.1   # <-- MISMATCH!
  dep  github.com/valyala/fasthttp      v1.55.0   # <-- MISMATCH!
```

Oprava:

```bash
# Update to match Bifrost's core version
go get github.com/maximhq/bifrost/core@v1.14.1
go mod tidy

# Rebuild the plugin
go build -buildmode=plugin -o my-plugin.so main.go
```

> [!IMPORTANT]
> Plugin se musí sestavit znovu při **každém** upgradu Bifrostu. Pro staging a produkci s různými verzemi potřebujete různé buildy pluginu.

## Další kroky

- **[Dynamická image](./dynamicka-image.md):** Bifrost, který plugin načte, v Dockeru a Kubernetes.
- **Podrobnosti u výrobce:** [Writing Go Plugins](https://docs.getbifrost.ai/plugins/writing-go-plugin) (MCP hooky, stavové pluginy, cache, fallbacky, výpočet ceny v pluginu), [ukázkové pluginy](https://github.com/maximhq/bifrost/tree/main/examples/plugins) (`llm-only`, `http-transport-only`, `mcp-only`, `virtual-key-from-config` a další), [Plugin Migration Guide](https://docs.getbifrost.ai/plugins/migration-guide).

---

<sub>Zkrácený návod, který sestavil think private z oficiální dokumentace Bifrostu (Apache 2.0), soubory `docs/plugins/writing-go-plugin.mdx` a `docs/plugins/building-dynamic-binary.mdx`, commit `0fb9342`. Změny: výběr a zkrácení obsahu, překlad do češtiny, převod z MDX do Markdownu, doplněk think private ke zjištění verzí (označený). Kód výrobce je převzatý beze změny. Originály: https://docs.getbifrost.ai/plugins/writing-go-plugin a https://docs.getbifrost.ai/plugins/building-dynamic-binary</sub>
