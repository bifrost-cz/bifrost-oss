<!--
druh: rychlý návod (výběr a zkrácení, ne úplný překlad)
zdroje: maximhq/bifrost docs/cli-agents/overview.mdx, docs/cli-agents/claude-code.mdx, docs/cli-agents/codex-cli.mdx
upstream-commit: 0fb93426bea0e211963773c0a46876452869276c
sestaveno: 2026-11-14
-->

# CLI agenti: Claude Code a Codex CLI

*Jak pustit programovací agenty přes Bifrost, aby každý požadavek prošel governance, rozpočty a logováním.*

> [!NOTE]
> Zkrácený český návod: vybírá to podstatné pro rozjezd. Úplný popis je v dokumentaci výrobce: [CLI Agents](https://docs.getbifrost.ai/cli-agents/overview), [Claude Code](https://docs.getbifrost.ai/cli-agents/claude-code) a [Codex CLI](https://docs.getbifrost.ai/cli-agents/codex-cli).

## Proč přes bránu

Bifrost nabízí endpointy kompatibilní s OpenAI, Anthropicem i Gemini, takže agenti s ním mluví stejně jako s výrobcem modelu. Získáte:

- **kontrolu nákladů:** každý vývojář nebo tým s vlastním [virtuálním klíčem](./virtualni-klice.md) a [rozpočtem](./rozpocty-a-limity.md),
- **přehled:** všechny požadavky agentů v [logech](./observabilita.md),
- **volbu modelů:** i modely jiných poskytovatelů nebo lokální modely, pokud podporují to, co agent potřebuje,
- **MCP nástroje** z brány (viz [MCP gateway](./mcp-gateway.md)),
- **odolnost:** fallbacky a vyvažování mezi poskytovateli.

> [!WARNING]
> Kompatibilita není u všech agentů a modelů stejná. Agenti spoléhají na volání nástrojů, streamování a další funkce specifické pro protokol. Model, který funguje pro chat, nemusí být vhodný pro agentní programování. Před nasazením si projděte omezení konkrétního agenta v dokumentaci výrobce.

Virtuální klíč pro agenta musí povolovat modely, které chcete použít. Nový klíč bez poskytovatelů neprojde nikam.

## Nejrychlejší cesta: Bifrost CLI

[Bifrost CLI](https://docs.getbifrost.ai/quickstart/cli/getting-started) spustí agenta s nastavenou adresou brány, klíčem a modelem jen pro danou relaci, bez zásahu do jeho konfigurace:

```bash
bifrost context add production --base-url https://bifrost.example.com --use

# Enterprise: sign in with SSO
bifrost auth login

# OSS or Enterprise: store a virtual key instead
printf '%s\n' "$BIFROST_VIRTUAL_KEY" | bifrost auth set-virtual-key --stdin

bifrost claude --model anthropic/claude-sonnet-4-6
```

Pro Codex CLI obdobně, včetně kontroly, že klíč model povoluje:

```bash
bifrost context add production \
  --base-url https://gateway.example.com \
  --use

# Enterprise user-scoped inference
bifrost auth login

# Or configure a virtual key
printf '%s\n' "$BIFROST_VIRTUAL_KEY" |
  bifrost auth set-virtual-key --stdin

# Confirm the virtual key allows this model before launching
bifrost models list

bifrost run codex --model openai/gpt-5
```

## Claude Code ručně s virtuálním klíčem

Claude Code používá cestu `/anthropic` a klíč posílá jako `Authorization: Bearer`, což Bifrost přijme jako virtuální klíč. Nejdřív ověřte adresu, klíč a model:

```bash
export ANTHROPIC_BASE_URL=https://bifrost.example.com/anthropic
export ANTHROPIC_AUTH_TOKEN=sk-bf-your-virtual-key

curl -sS -w '\n%{http_code}\n' "$ANTHROPIC_BASE_URL/v1/messages" \
  -H "Authorization: Bearer $ANTHROPIC_AUTH_TOKEN" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{"model": "claude-sonnet-4-6", "max_tokens": 16, "messages": [{"role": "user", "content": "ping"}]}'
```

Odpověď `200` znamená, že vše sedí. `401` je odmítnutý klíč, `403` obvykle znamená, že klíč nepovoluje daného poskytovatele nebo model.

Potom nastavení uložte do uživatelského `~/.claude/settings.json` (sloučit se stávajícím obsahem, ne přepsat):

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "https://bifrost.example.com/anthropic",
    "ANTHROPIC_AUTH_TOKEN": "sk-bf-your-virtual-key"
  }
}
```

> [!CAUTION]
> Klíč nikdy nedávejte do `.claude/settings.json` v projektu. Ten se commituje a uvidí ho každý, kdo si repozitář naklonuje. Pro nastavení jednoho repozitáře použijte `.claude/settings.local.json` uvedený v `.gitignore`.

Kdo platí předplatné Claude a chce si ho nechat, může se přihlásit přes `/login` a virtuální klíč posílat v hlavičce `x-bf-vk`. Bifrost pak předá token předplatného Anthropicu a klíč použije pro governance a logování. Postup je v [dokumentaci výrobce](https://docs.getbifrost.ai/cli-agents/claude-code).

## Codex CLI ručně s virtuálním klíčem

Virtuální klíč exportujte ve stejném terminálu, ze kterého spouštíte Codex:

```bash
export OPENAI_API_KEY="your-bifrost-virtual-key"
```

A přidejte pojmenovaného poskytovatele do `~/.codex/config.toml`:

```toml
model = "openai/gpt-5"
model_provider = "bifrost"

[model_providers.bifrost]
name = "Bifrost"
base_url = "http://localhost:8080/openai/v1"
wire_api = "responses"
env_key = "OPENAI_API_KEY"
requires_openai_auth = false
supports_websockets = false
```

- Koncové `/v1` v `base_url` je povinné, Codex k adrese přidává `/responses`.
- `supports_websockets = false` drží Codex na HTTP, což je potřeba, když Bifrost požadavek překládá pro poskytovatele bez WebSocket protokolu OpenAI.
- Pokud máte v Codexu aktivní přihlášení k OpenAI, nejdřív se odhlaste (`/logout`), jinak má přednost.

## Další agenti a editory

Stejným způsobem lze přes Bifrost pustit také Gemini CLI, Qwen Code, OpenCode, Cursor, Zed, Roo Code, GitHub Copilot, LibreChat nebo Open WebUI. Návody pro jednotlivé nástroje jsou v [dokumentaci výrobce](https://docs.getbifrost.ai/cli-agents/overview).

## Další kroky

- **[Virtuální klíče](./virtualni-klice.md)** a **[Rozpočty a limity](./rozpocty-a-limity.md):** klíč a rozpočet pro každého vývojáře nebo tým.
- **Podrobnosti u výrobce:** [Claude Code](https://docs.getbifrost.ai/cli-agents/claude-code) (dynamické přihlašování přes `apiKeyHelper`, SSO v Enterprise), [Codex CLI](https://docs.getbifrost.ai/cli-agents/codex-cli) (přihlášení přes ChatGPT), [Bifrost CLI](https://docs.getbifrost.ai/quickstart/cli/getting-started).

---

<sub>Zkrácený návod, který sestavil think private z oficiální dokumentace Bifrostu (Apache 2.0), soubory `docs/cli-agents/overview.mdx`, `docs/cli-agents/claude-code.mdx` a `docs/cli-agents/codex-cli.mdx`, commit `0fb9342`. Změny: výběr a zkrácení obsahu, překlad do češtiny, převod z MDX do Markdownu. Kód je převzatý beze změny. Originály: https://docs.getbifrost.ai/cli-agents/overview, https://docs.getbifrost.ai/cli-agents/claude-code a https://docs.getbifrost.ai/cli-agents/codex-cli</sub>
