# Bifrost česky

<p>
  <a href="https://github.com/maximhq/bifrost"><img alt="Bifrost na GitHubu" src="https://img.shields.io/github/stars/maximhq/bifrost?label=Bifrost&logo=github"></a>
  <a href="https://github.com/bifrost-cz/bifrost-oss/discussions"><img alt="Diskuze" src="https://img.shields.io/github/discussions/bifrost-cz/bifrost-oss?label=diskuze"></a>
  <img alt="Jazyk" src="https://img.shields.io/badge/jazyk-%C4%8Desky%20%7C%20slovensky-33C09E">
  <a href="#licence"><img alt="Licence" src="https://img.shields.io/badge/licence-Apache%202.0-blue"></a>
</p>

**Bifrost česky** je komunitní místo pro sdílení znalostí, návodů a osvědčených postupů k [Bifrostu](https://github.com/maximhq/bifrost), open source AI bráně od Maxim AI. Bifrost funguje jako LLM, MCP i agent gateway: jedno OpenAI-kompatibilní API pro lokální i cloudové modely s failoverem, vyvažováním zátěže, sémantickou cache, virtuálními klíči a rozpočty.

Začínáme českým překladem oficiální dokumentace. Překládáme věrně a průběžně aktualizujeme podle nových vydání. U každé stránky je uvedená verze dokumentace výrobce, ze které vychází.

## Obsah

| Oblast | Stránka | Stav |
|---|---|---|
| Úvod | [Bifrost AI Gateway](docs/overview.md) | ✅ |
| Rychlý start | [Nastavení brány](docs/quickstart/gateway/setting-up.md) | ✅ |
| Rychlý start | [Konfigurace poskytovatelů](docs/quickstart/gateway/provider-configuration.md) | ✅ |
| Rychlý start | [Nastavení ověřování](docs/quickstart/gateway/setting-up-auth.md) | ✅ |
| Rychlý start | [Integrace](docs/quickstart/gateway/integrations.md) | ✅ |
| Rychlý start | streamování, volání nástrojů, multimodalita | 🚧 připravujeme |
| Funkce | virtuální klíče, rozpočty, směrování, sémantická cache | 🚧 připravujeme |
| MCP | MCP gateway, Code Mode, Agent Mode | 🚧 připravujeme |
| Nasazení | Docker, Kubernetes a Helm | 🚧 připravujeme |

Chybí vám téma? Napište nám do [Diskuzí](https://github.com/bifrost-cz/bifrost-oss/discussions).

## Komunita a pomoc

- 💬 **[Diskuze](https://github.com/bifrost-cz/bifrost-oss/discussions):** dotazy, zkušenosti a ukázky nasazení, česky i slovensky.
- 🐞 **Chyby v Bifrostu** hlaste v oficiálním repozitáři [maximhq/bifrost](https://github.com/maximhq/bifrost/issues).
- 📚 **Oficiální dokumentace:** [docs.getbifrost.ai](https://docs.getbifrost.ai)
- 🗨️ **Discord Bifrostu:** [discord.gg/exN5KAydbU](https://discord.gg/exN5KAydbU)

## Enterprise a podpora v ČR a SK

Bifrost Enterprise přidává jednotné přihlášení (SSO) a role, audit, guardrails, provoz v clusteru, napojení na trezory tajemství a Bifrost Edge pro koncová zařízení (v programu včasného přístupu). Nasadíte ho u sebe, ve VPC i bez připojení k internetu.

[think private](https://thinkprivate.ai/bifrost) je partnerem pro Bifrost v Česku a na Slovensku. Pomůžeme s architekturou, nasazením a provozem.

👉 **[Nezávazná konzultace](https://thinkprivate.ai/bifrost)** · info@thinkprivate.ai

## Jak přispět

- Opravu překladu pošlete jako pull request, nápady a dotazy pište do [Diskuzí](https://github.com/bifrost-cz/bifrost-oss/discussions).
- Nikdy nevkládejte API klíče, hesla ani jiné citlivé údaje. V ukázkách používejte zástupné hodnoty.

## Licence

Překlady oficiální dokumentace jsou stejně jako originál pod licencí [Apache 2.0](LICENSE). Podrobnosti v [LICENSE-docs.md](LICENSE-docs.md).

---

<sub>Neoficiální komunitní projekt, který spravuje think private, partner pro Bifrost v ČR a SK. Nejde o oficiální produkt ani dokumentaci Maxim AI. Bifrost je ochranná známka H3 Labs, Inc. (Maxim AI).</sub>
