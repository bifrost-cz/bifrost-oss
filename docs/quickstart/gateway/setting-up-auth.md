<!--
zdroj: maximhq/bifrost docs/quickstart/gateway/setting-up-auth.mdx
upstream-commit: 7235fb54f388679e8c4c93ce9687a79ae084700a
prelozeno: 2026-11-12
-->

# Nastavení ověřování

*Jak zapnout základní ověřování pro dashboard Bifrostu a zabezpečit administrační rozhraní a endpointy API.*

## Přehled

Bifrost má vestavěné ověřování, které chrání dashboard a endpointy administračního API. Když ho zapnete, uživatelé se musí přihlásit, než otevřou dashboard nebo zavolají administrační API. Tato funkce pomáhá zabezpečit vaši instanci Bifrostu, hlavně při nasazení v produkci.

Mazání logů nástrojů MCP vyžaduje ověřený přístup ke správě. Požadavky, které ověřování obcházejí, protože je vypnuté, nenastavené nebo je trasa na seznamu povolených, dostanou `403`. Než budete logy mazat, zapněte ověřování a odeberte endpoint pro mazání ze seznamu tras bez ověřování.

> [!NOTE]
> **Bifrost Enterprise:** pokud používáte jednotné přihlášení, tento přepínač hesla nepotřebujete. Nastavené přihlášení přes OAuth 2.0 / OIDC nebo příchozí zřizování přes SCIM 2.0 se už počítá jako nastavené ověřování dashboardu a chrání dashboard i administrační API stejně. Viz [User Provisioning](https://docs.getbifrost.ai/enterprise/user-provisioning).

## Zapnutí ověřování

### Krok 1: Otevřete nastavení zabezpečení

1. Otevřete dashboard Bifrostu.
2. Přejděte na **Workspace** → **Config** → záložka **Security**.
3. Sjeďte k části **Password protect the dashboard**.

![Nastavení ověřování](https://raw.githubusercontent.com/maximhq/bifrost/7235fb54f388679e8c4c93ce9687a79ae084700a/docs/media/setting-up-dashboard-auth.png)

### Krok 2: Zapněte ověřování

1. Přepínačem **Password protect the dashboard** ověřování zapněte.
2. Do pole pro uživatelské jméno správce zadejte **Username**.
3. Do pole pro heslo správce zadejte **Password**.

> [!NOTE]
> Pole pro uživatelské jméno a heslo jsou aktivní, jen když je přepínač ověřování zapnutý. Použijte silné heslo.

> [!NOTE]
> **Vytvoření prvního účtu správce:** dokud účet správce neexistuje, je dashboard a administrační API Bifrostu dostupné bez hesla pro kohokoli, kdo se k němu dostane po síti. Aby se tato mezera uzavřela, musíte při prvním zapnutí ověřování na instanci zadat i **Setup token**. Hodnotu předem nastaví provozovatel přes `setup_token` v `config.json` (přijímá přímou hodnotu nebo odkaz `env.VAR_NAME` / `vault.path`, stejně jako `admin_username`/`admin_password`) nebo přes proměnnou prostředí `BIFROST_SETUP_TOKEN`. Stejnou hodnotu vložte do pole **Setup token**, které se objeví pod polem pro heslo. Setup token se nikdy neukládá a nikdy se nevypisuje do logů. Pokud setup token nastavený není, vytvoření prvního účtu správce se odmítne, dokud ho nenastavíte. Protože každý uzel ve vícenodovém nasazení čte stejnou nastavenou hodnotu, nemusíte mířit na konkrétní uzel.

> [!NOTE]
> Setup token je povinný od verze 2.0.0-prerelease3.

> [!NOTE]
> **Opětovné zapnutí ověřování po jeho vypnutí:** dokud je ověřování vypnuté, je administrační API dostupné bez hesla, přestože je účet správce stále uložený. Aby ho nikdo, kdo se dostane na port, nemohl znovu zapnout se svými vlastními údaji, musí požadavek `PUT /api/config`, který ověřování znovu zapíná nebo mění `admin_username` / `admin_password`, zatímco je vypnuté, prokázat kontrolu nad instancí. Buď přes `auth_config.current_password` (uložené heslo správce), nebo přes nastavený `auth_config.setup_token`. Požadavky z ověřené relace (například přes Basic Auth) nepotřebují ani jedno.

### Krok 3: Ověřování inferenčních volání (volitelné)

Když je ověřování zapnuté, ve výchozím stavu vyžadují ověření všechna volání API, včetně inferenčních. Ověřování inferenčních volání můžete volitelně vypnout a ponechat ho zapnuté pro dashboard a administrační API:

1. Zapněte přepínač **Disable authentication on inference calls**.
2. Po zapnutí:
   - volání dashboardu a administračního API budou ověření dál vyžadovat,
   - inferenční volání API (chat completions, embeddings atd.) ověření vyžadovat nebudou,
   - volání pro spouštění nástrojů MCP budou ověření dál vyžadovat.

> [!NOTE]
> Tato volba se hodí, když chcete chránit dashboard a administrační funkce a inferenční endpointy nechat veřejně dostupné.

### Krok 4: Trasy bez ověřování (volitelné)

Můžete nastavit konkrétní trasy, které middleware pro ověřování úplně obejdou. Požadavky na tyto trasy nebudou vyžadovat ověření, ani když je ověřování zapnuté.

1. Sjeďte k části **Whitelisted Routes**.
2. Do textového pole zadejte seznam tras oddělených čárkou.

![Nastavení tras bez ověřování](https://raw.githubusercontent.com/maximhq/bifrost/7235fb54f388679e8c4c93ce9687a79ae084700a/docs/media/ui-security-whitelisted-routes.png)

**Zástupné znaky:** trasy končící `*` se porovnávají podle prefixu. Například `/api/webhook*` odpovídá `/api/webhook`, `/api/webhook/v1`, `/api/webhook/github` a podobně.

**Příklad hodnot:**

```
/api/custom-webhook, /api/public-endpoint, /api/webhook*
```

> [!NOTE]
> Systémové trasy jako `/health`, `/api/session/login`, `/api/session/is-auth-enabled`, `/api/oauth/callback` a `/api/info` jsou bez ověřování vždy, bez ohledu na toto nastavení. Trasy bez ověřování platí jen pro dashboard a endpointy administračního API. Inferenční endpointy mají vlastní přepínač (viz krok 3).

### Krok 5: Uložte změny

1. Nastavení ověřování použijete kliknutím na **Save Changes**.
2. Změny platí okamžitě, restart není potřeba.

## Přihlášení

Když je ověřování zapnuté:

1. Otevřete URL dashboardu Bifrostu.
2. Budete automaticky přesměrováni na přihlašovací stránku.
3. Zadejte nastavené uživatelské jméno a heslo.
4. Klikněte na **Sign in**.

Po úspěšném přihlášení budete přesměrováni na dashboard. Relace zůstane aktivní 30 dní, pak se musíte přihlásit znovu.

## Způsoby ověřování

Bifrost podporuje různé způsoby ověřování podle typu požadavku:

### Přístup k dashboardu

- **Ověřování Bearer tokenem**: dashboard používá ověřování Bearer tokenem.
- Tokeny se spravují automaticky v rámci přihlášené relace.
- Tokeny se ukládají do localStorage prohlížeče a posílají se s každým požadavkem na API.

### Volání API

Když je ověřování zapnuté, volání API lze provádět pomocí:

- **Basic Authentication**: uživatelské jméno a heslo zakódované v base64 přes HTTP Basic auth,
- **Bearer Token**: token relace vydaný po přihlášení (Bearer token z relace).

Když je ověřování inferenčních volání zapnuté (tedy přepínač **Disable authentication on inference calls** je VYPNUTÝ), inferenční volání lze provádět pomocí:

- **Basic Authentication**: uživatelské jméno a heslo v Basic auth,
- **Bearer Token**: řetězec username:password v base64 jako bearer token.

> [!NOTE]
> **Enterprise: přijímají se i tokeny z IdP.** V Bifrost Enterprise s [nastaveným SSO](https://docs.getbifrost.ai/enterprise/user-provisioning) můžete všude, kde lze inferenční volání ověřit virtuálním klíčem, poslat i token vydaný vaším poskytovatelem identity: `Authorization: Bearer <idp-token>`. Bifrost ho ověří proti nastavenému IdP a požadavek přiřadí danému uživateli, včetně jeho access profilu, rozpočtů a limitů požadavků. Se zapnutým [`enforce_auth_on_inference`](https://docs.getbifrost.ai/deployment-guides/config-json/client) se nový uživatel zřídí při svém prvním požadavku. Token posílejte v hlavičce `Authorization`, `x-api-key` a `x-bf-vk` nesou jen virtuální klíče. Pokud požadavek nese obojí, rozhodne o použití [`dual_credential_conflict_behavior`](https://docs.getbifrost.ai/deployment-guides/config-json/client).

### Trasy bez ověřování

Když je trasa na seznamu povolených tras v nastavení Security, požadavky na ni ověřování úplně obcházejí, není potřeba Basic Auth ani Bearer token. Platí to jen pro dashboard a endpointy administračního API. Inferenční endpointy se řídí zvlášť přepínačem **Disable authentication on inference calls**.

### Příklad: Basic Auth pro inferenční volání

```bash
# Using curl with Basic Auth
curl -X POST http://localhost:8080/v1/chat/completions \
  -u "your-username:your-password" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "openai/gpt-4o",
    "messages": [{"role": "user", "content": "Hello!"}]
  }'
```

## Důležité poznámky

- **Bez restartu**: změny ověřování platí okamžitě, restart serveru není potřeba.
- **Délka relace**: přihlášení platí 30 dní.
- **Bezpečnost hesel**: hesla se ukládají do databáze bezpečně jako hash.
- **Inferenční volání**: pokud ověřování inferenčních volání vypnete, chráněné zůstanou jen dashboard a endpointy administračního API.
- **Trasy bez ověřování**: trasy nastavené v Security obcházejí ověřování jen u dashboardu a administračního API. Pro porovnání podle prefixu dejte na konec trasy `*` (například `/api/webhook*`).

## Vypnutí ověřování

Ověřování vypnete takto:

1. Otevřete **Workspace** → **Config** → **Security**.
2. Vypněte přepínač **Password protect the dashboard**.
3. Klikněte na **Save Changes**.

Po vypnutí je dashboard okamžitě dostupný bez ověření.

---

<sub>Přeloženo z oficiální dokumentace Bifrostu (Apache 2.0), soubor `docs/quickstart/gateway/setting-up-auth.mdx`, commit `7235fb5`. Změny: překlad do češtiny, převod z MDX do Markdownu. Originál: https://docs.getbifrost.ai/quickstart/gateway/setting-up-auth</sub>
