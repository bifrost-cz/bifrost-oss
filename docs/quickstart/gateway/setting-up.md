<!--
zdroj: maximhq/bifrost docs/quickstart/gateway/setting-up.mdx
upstream-commit: 7235fb54f388679e8c4c93ce9687a79ae084700a
prelozeno: 2026-11-12
-->

# Nastavení brány

*Bifrost jako HTTP API bránu spustíte za 30 sekund bez jakékoli konfigurace. Hodí se pro jakýkoli programovací jazyk.*

![Instalace brány Bifrost](https://raw.githubusercontent.com/maximhq/bifrost/7235fb54f388679e8c4c93ce9687a79ae084700a/docs/media/getting-started.png)

## Nastavení za 30 sekund

Spusťte Bifrost jako bleskově rychlou HTTP API bránu **bez jakékoli konfigurace**. K libovolnému poskytovateli AI (OpenAI, Anthropic, Bedrock a dalším) se připojíte přes jednotné API, které dodržuje **formát požadavků a odpovědí OpenAI**.

### 1. Zvolte způsob instalace

Obě možnosti fungují stejně dobře, vyberte si tu, která vám sedí:

#### Binárka přes NPX

[▶ Video: spuštění přes npx](https://github.com/maximhq/bifrost/raw/refs/heads/main/docs/media/run-npx.mp4)

```bash
# Install and run locally
npx -y @maximhq/bifrost

# Install a specific version
npx -y @maximhq/bifrost --transport-version v2.4.0
```

#### Docker

```bash
# Pull and run Bifrost HTTP API
docker pull maximhq/bifrost
docker run -p 8080:8080 maximhq/bifrost

# Pull a specific version
docker pull maximhq/bifrost:v2.4.0
docker pull maximhq/bifrost:v2.4.0-amd64
docker pull maximhq/bifrost:v2.4.0-arm64
```

**Trvalé uložení dat**

```bash
# For configuration persistence across restarts
docker run -p 8080:8080 -v $(pwd)/data:/app/data maximhq/bifrost
```

> [!WARNING]
> Tyto příkazy bez verze jsou určené pro lokální vyzkoušení. Pro hostované nebo produkční nasazení zafixujte verzi image, zapněte ověřování (první účet správce potřebuje [setup token](#ochrana-dashboardu)) a TLS a zvolte trvalé úložiště podle [tabulky podporovaných nasazení](https://docs.getbifrost.ai/deployment-guides/overview).

### 2. Konfigurační přepínače

| Přepínač | Výchozí | NPX | Docker | Popis |
| --------- | --------- | ----------------- | ------------------------------- | ------------------------------------ |
| port | 8080 | `-port 8080` | `-e APP_PORT=8080 -p 8080:8080` | Port HTTP serveru |
| host | localhost | `-host 0.0.0.0` | `-e APP_HOST=0.0.0.0` | Adresa, na které server naslouchá |
| log-level | info | `-log-level info` | `-e LOG_LEVEL=info` | Úroveň logování (debug, info, warn, error) |
| log-style | json | `-log-style json` | `-e LOG_STYLE=json` | Styl logů (pretty, json) |

**Adresář aplikace**

Přepínač `-app-dir` určuje, kam Bifrost ukládá všechna svá data:

```bash
# Specify custom directory
npx -y @maximhq/bifrost -app-dir ./my-bifrost-data

# If not specified, creates in your OS config directory:
# • Linux/macOS: ~/.config/bifrost
# • Windows: %APPDATA%\bifrost
```

**Co se v adresáři aplikace ukládá:**

- `config.json`: konfigurační soubor (nepovinný)
- `config.db`: databáze SQLite pro konfiguraci z webového rozhraní
- `logs.db`: databáze logů požadavků

**Poznámka:** když Bifrost používáte přes Docker, jako adresář aplikace se použije připojený svazek.

### 3. Otevřete webové rozhraní

V prohlížeči otevřete **http://localhost:8080**:

```bash
# macOS
open http://localhost:8080

# Linux
xdg-open http://localhost:8080

# Windows
start http://localhost:8080
```

🖥️ **Webové rozhraní nabízí:**

- **Vizuální nastavení poskytovatelů**: API klíče přidáte kliknutím, ne kódem
- **Konfiguraci v reálném čase**: změny platí okamžitě
- **Živé sledování**: logy požadavků, metriky a analytika
- **Správu governance**: virtuální klíče, rozpočty na spotřebu a další

### 4. Vyzkoušejte první volání API

```bash
curl -X POST http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "openai/gpt-4o-mini",
    "messages": [{"role": "user", "content": "Hello, Bifrost!"}]
  }'
```

**🎉 A je to!** Bifrost běží a je připravený směrovat AI požadavky.

### Co se právě stalo?

1. **Start bez konfigurace**: Bifrost se spustil bez konfiguračních souborů, všechno lze nastavit přes webové rozhraní nebo API.
2. **API kompatibilní s OpenAI**: všechna API Bifrostu dodržují formát požadavků a odpovědí OpenAI, takže se snadno integrují.
3. **Jednotný endpoint API**: `/v1/chat/completions` funguje s jakýmkoli poskytovatelem (OpenAI, Anthropic, Bedrock a dalšími).
4. **Určení poskytovatele**: `openai/gpt-4o-mini` říká Bifrostu, že má použít model GPT-4o Mini od OpenAI. Můžete použít i samotný název modelu, třeba `gpt-4o-mini`, a Bifrost poskytovatele určí automaticky podle [katalogu modelů](https://docs.getbifrost.ai/providers/provider-routing#default-provider-resolution).
5. **Automatické směrování**: Bifrost se automaticky postará o ověřování, limity požadavků i směrování požadavků.

### Ochrana dashboardu

Čerstvě spuštěný Bifrost nemá heslo. Kdokoli, kdo se dostane na jeho port, může otevřít dashboard i administrační API. Než Bifrost zpřístupníte mimo svůj počítač, nastavte heslo.

Provozovat Bifrost bez hesla je dovolené. Jakmile ale vytváříte první účet správce, je **setup token povinný**. Hodnotu si zvolíte sami a nastavíte ji **před spuštěním Bifrostu**. Pokud setup token nastavený není, Bifrost požadavek na vytvoření účtu správce odmítne.

```bash
# Generate a random token and keep it somewhere safe
export BIFROST_SETUP_TOKEN=$(openssl rand -hex 32)

# NPX
npx -y @maximhq/bifrost

# Docker
docker run -p 8080:8080 -v $(pwd)/data:/app/data -e BIFROST_SETUP_TOKEN maximhq/bifrost
```

Pak v dashboardu:

1. Otevřete **Workspace** → **Config** → **Security**.
2. Zapněte **Password protect the dashboard**.
3. Zadejte uživatelské jméno a heslo správce.
4. Vložte svůj token do pole **Setup token** a uložte.

> [!NOTE]
> Token se kontroluje jen při vytváření prvního účtu správce. Bifrost ho nikdy neukládá ani nezapisuje do logů a jakmile účet existuje, už na něm nezáleží. Pokud jste ho zapomněli nastavit, Bifrost zastavte a spusťte znovu s nastaveným `BIFROST_SETUP_TOKEN`. Můžete také nastavit `setup_token` v `config.json`. Celý postup najdete v části [Nastavení ověřování](./setting-up-auth.md).

---

## Režimy konfigurace

Bifrost má dva režimy konfigurace:

| Režim | Kdy ho použít | Chování |
|-------|------|----------|
| Webové rozhraní / databáze | Konfiguraci chcete spravovat z dashboardu nebo přes management API | Bifrost ukládá konfiguraci do SQLite nebo PostgreSQL. Pokud existuje i `config.json`, při startu databázi naplní nebo s ní sesouhlasí. |
| Jen soubor `config.json` | Chcete deklarativní konfiguraci, která se načte při restartu, bez konfigurační databáze | Bifrost načte konfiguraci ze souboru při startu. Úpravy konfigurace přes rozhraní a API nejsou dostupné a změny souboru vyžadují restart. |

> [!NOTE]
> Vynechání `config_store` **neznamená** režim jen se souborem. Pokud `config.json` existuje a `config_store` v něm chybí, Bifrost vytvoří výchozí konfigurační úložiště SQLite a soubor bere jako startovní konfiguraci pro databázi. `config_store.enabled: false` nastavte, jen když má být soubor jediným zdrojem konfigurace.

**Jak `config.json` s databází spolupracuje se stavem databáze:**

- Entity definované v `config.json` se při prvním načtení uloží s hashem podle obsahu.
- Pozdější změny přes rozhraní nebo API se uloží do databáze a přežijí restarty podů, škálování i nová nasazení.
- Opětovné použití stejného `config.json` tyto změny v databázi nepřepíše, protože hash obsahu je stejný. Pokud entitu upravíte přímo v `config.json`, její hash se změní a definice ze souboru přepíše kopii v databázi.
- `source_of_truth: "config.json"` nastavte, jen když mají výslovně uvedené sekce souboru při startu nahradit odpovídající stav databáze. Chybějící sekce nechají řádky v databázi beze změny, uvedené prázdné sekce mohou odpovídající řádky odstranit.

Úplná pravidla sesouhlasení najdete v části [Source of Truth & Reconciliation](https://docs.getbifrost.ai/deployment-guides/config-json/source-of-truth).

### Režim 1: konfigurace přes webové rozhraní a databázi

![Konfigurace přes webové rozhraní](https://raw.githubusercontent.com/maximhq/bifrost/7235fb54f388679e8c4c93ce9687a79ae084700a/docs/media/ui-config.png)

**Kdy je webové rozhraní dostupné:**

- Neexistuje žádný soubor `config.json` (Bifrost automaticky vytvoří databázi SQLite).
- `config.json` existuje a `config_store` je v něm vynechané nebo zapnuté.

### Volitelně: config.json s databází

> [!NOTE]
> Celé schéma konfigurace najdete [zde](https://www.getbifrost.ai/schema).

**Kdy ho použít:** chcete začít z konfigurace v souboru a přitom si ponechat konfiguraci přes rozhraní a API za běhu.

Vytvořte `config.json` v adresáři aplikace:

```json
{
  "$schema": "https://www.getbifrost.ai/schema",
  "client": {
    "drop_excess_requests": false
  },
  "providers": {
    "openai": {
      "keys": [
        {
          "name": "openai-key-1",
          "value": "env.OPENAI_API_KEY",
          "models": ["gpt-4o-mini", "gpt-4o"],
          "weight": 1.0
        }
      ]
    }
  },
  "config_store": {
    "enabled": true,
    "type": "sqlite",
    "config": {
      "path": "./config.db"
    }
  }
}
```

**S vynechaným nebo zapnutým `config_store`:**

- **Webové rozhraní je zapnuté**: plná konfigurace v reálném čase přes webové rozhraní.
- **Kontrola databáze**: Bifrost sesouhlasí `config.json` s konfiguračním úložištěm podle hashe obsahu každé entity.
  - **Prázdná databáze**: naplní databázi nastavením z `config.json` a za běhu pak čte z databáze.
  - **Existující databáze**: porovná hash každé entity ze souboru s uloženým `ConfigHash`. Ve výchozím rozděleném režimu si nezměněné entity ze souboru ponechají kopii v databázi, takže úpravy přes rozhraní a API zůstanou zachované. Pokud entitu upravíte v `config.json`, definice ze souboru pro tuto entitu přepíše kopii v databázi.
- **Trvalé úložiště**: všechny změny se okamžitě ukládají do databáze.

### Režim 2: jen soubor config.json

**Kdy ho použít:** pokročilá nastavení, postupy GitOps nebo když webové rozhraní nepotřebujete.

`config_store.enabled` nastavte na `false`, když má být `config.json` jediným zdrojem konfigurace:

```json
{
  "$schema": "https://www.getbifrost.ai/schema",
  "providers": {
    "openai": {
      "keys": [
        {
          "name": "openai-key-1",
          "value": "env.OPENAI_API_KEY",
          "models": ["gpt-4o-mini", "gpt-4o"],
          "weight": 1.0
        }
      ]
    }
  },
  "config_store": {
    "enabled": false
  }
}
```

- **Úpravy konfigurace přes rozhraní a API nejsou dostupné**: žádné konfigurační úložiště se neinicializuje.
- **Soubor jen pro čtení**: Bifrost `config.json` nikdy nemění.
- **Konfigurace jen v paměti**: nastavení se načte do paměti při startu.
- **Nutný restart**: změny v `config.json` se projeví až po restartu.

**Tři úložiště:**

- **Config Store**: ukládá konfiguraci poskytovatelů, API klíče a nastavení MCP. Nutné pro fungování webového rozhraní.
- **Logs Store**: ukládá logy požadavků zobrazované v rozhraní. Nepovinné, lze vypnout.
- **Vector Store**: používá se pro sémantickou cache. Nepovinné, lze vypnout.

## Požadavek na UTF8 v PostgreSQL

> [!NOTE]
> Minimální požadovaná verze PostgreSQL je 16.

> [!NOTE]
> Pro úložiště logů vytváří Bifrost materializované pohledy, které zrychlují analytiku. Ujistěte se, že uživatel PostgreSQL má na cílovém schématu oprávnění k těmto operacím.

> [!NOTE]
> Pokud jste za PgBouncerem v režimu transaction pooling, za RDS Proxy nebo za jakoukoli proxy, která nezaručuje afinitu relací, mohou migrace Bifrostu selhat kvůli problémům se zamykáním souběžných relací.

Pokud používáte PostgreSQL pro `config_store` nebo `logs_store`, cílová databáze musí mít kódování `UTF8`.

Při vytváření databáze použijte `template0`, aby PostgreSQL nastavil UTF8 a lokalizaci výslovně:

```sql
CREATE DATABASE bifrost
  WITH TEMPLATE template0
       ENCODING 'UTF8'
       LC_COLLATE '<your-locale>'
       LC_CTYPE '<your-locale>';
```

Použijte názvy lokalizací, které ve vašem image nebo na hostiteli PostgreSQL existují (například `en_US.UTF-8`, `C.UTF-8` nebo jinou nainstalovanou lokalizaci UTF-8).

Ověřte kódování databáze:

```sql
SELECT datname, pg_encoding_to_char(encoding) AS encoding
FROM pg_database
WHERE datname = 'bifrost';
```

Pokud databáze není v UTF8, start nebo migrace Bifrostu mohou selhat s chybou:

```text
simple protocol queries must be run with client_encoding=UTF8
```

Pokud už máte vytvořenou databázi SQL_ASCII, vytvořte novou databázi v UTF8 a v konfiguraci databáze Bifrostu na ni přesměrujte.

---

## Další kroky

Bifrost běží. Pokračujte těmito návody:

### Základní témata

- **[Konfigurace poskytovatelů](./provider-configuration.md)**: více poskytovatelů, automatické přepínání při výpadku a vyvažování zátěže.
- **[Integrace](https://docs.getbifrost.ai/integrations/what-is-an-integration)**: přímé náhrady za SDK OpenAI, Anthropic a GenAI.
- **[Multimodal Support](./multimodal.md)**: podpora textu, obrázků, zvuku a streamování za společným rozhraním.

### Pokročilá témata

- **[Tracing](https://docs.getbifrost.ai/features/observability/default)**: logování požadavků pro monitoring a ladění.
- **[MCP Tools](https://docs.getbifrost.ai/mcp/overview)**: umožněte AI modelům používat externí nástroje (souborový systém, webové vyhledávání, databáze).
- **[Governance](https://docs.getbifrost.ai/features/governance/virtual-keys)**: sledování využití, limity požadavků a řízení nákladů.
- **[Deployment](https://docs.getbifrost.ai/deployment-guides/k8s)**: produkční nasazení a škálování.

---

**Hodně štěstí při stavění s Bifrostem!** 🚀

---

<sub>Přeloženo z oficiální dokumentace Bifrostu (Apache 2.0), soubor `docs/quickstart/gateway/setting-up.mdx`, commit `7235fb5`. Změny: překlad do češtiny, převod z MDX do Markdownu. Originál: https://docs.getbifrost.ai/quickstart/gateway/setting-up</sub>
