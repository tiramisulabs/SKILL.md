# Link Verification Matrix

Audit of every supplied docs URL: each was fetched as raw MDX from `tiramisulabs/seyfert-web@seyfert-v5` (`content/docs/<slug>.mdx`). Page notes carry the per-URL source anchors and explicit doc-vs-source corrections; resolve API conflicts against the authoritative Seyfert source defined in `source-truth.md`.

| # | Docs URL | Page note | Topic ref | Verification |
|---|---|---|---|---|
| 1 | [learn/getting-started/index](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/getting-started/index) | `pages/learn/getting-started/index.md` | `setup-runtime.md` | raw MDX + ./src |
| 2 | [learn/getting-started/setup-project](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/getting-started/setup-project) | `pages/learn/getting-started/setup-project.md` | `setup-runtime.md` | raw MDX + ./src |
| 3 | [learn/getting-started/first-command](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/getting-started/first-command) | `pages/learn/getting-started/first-command.md` | `setup-runtime.md` | raw MDX + ./src |
| 4 | [learn/getting-started/listening-events](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/getting-started/listening-events) | `pages/learn/getting-started/listening-events.md` | `setup-runtime.md` | raw MDX + ./src |
| 5 | [learn/getting-started/declare-module](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/getting-started/declare-module) | `pages/learn/getting-started/declare-module.md` | `setup-runtime.md` | raw MDX + ./src |
| 6 | [learn/commands/intro](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/commands/intro) | `pages/learn/commands/intro.md` | `commands.md` | raw MDX + ./src |
| 7 | [learn/commands/options](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/commands/options) | `pages/learn/commands/options.md` | `commands.md` | raw MDX + ./src |
| 8 | [learn/commands/middlewares](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/commands/middlewares) | `pages/learn/commands/middlewares.md` | `commands.md` | raw MDX + ./src |
| 9 | [learn/commands/subcommands](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/commands/subcommands) | `pages/learn/commands/subcommands.md` | `commands.md` | raw MDX + ./src |
| 10 | [learn/commands/context-menus](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/commands/context-menus) | `pages/learn/commands/context-menus.md` | `commands.md` | raw MDX + ./src |
| 11 | [learn/commands/handling-errors](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/commands/handling-errors) | `pages/learn/commands/handling-errors.md` | `commands.md` | raw MDX + ./src |
| 12 | [learn/commands/prefix-commands](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/commands/prefix-commands) | `pages/learn/commands/prefix-commands.md` | `commands.md` | raw MDX + ./src |
| 13 | [learn/commands/extend-commandcontext](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/commands/extend-commandcontext) | `pages/learn/commands/extend-commandcontext.md` | `commands.md` | raw MDX + ./src |
| 14 | [learn/components/building-components](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/components/building-components) | `pages/learn/components/building-components.md` | `components.md / builders.md` | raw MDX + ./src |
| 15 | [learn/components/handling-components](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/components/handling-components) | `pages/learn/components/handling-components.md` | `components.md / builders.md` | raw MDX + ./src |
| 16 | [learn/components/modals](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/components/modals) | `pages/learn/components/modals.md` | `components.md / builders.md` | raw MDX + ./src |
| 17 | [learn/components/collectors](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/components/collectors) | `pages/learn/components/collectors.md` | `components.md / builders.md` | raw MDX + ./src |
| 18 | [learn/components/polls](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/components/polls) | `pages/learn/components/polls.md` | `components.md / builders.md` | raw MDX + ./src |
| 19 | [learn/components/v2](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/components/v2) | `pages/learn/components/v2.md` | `components.md / builders.md` | raw MDX + ./src |
| 20 | [learn/i18n/languages](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/i18n/languages) | `pages/learn/i18n/languages.md` | `i18n-cache-recipes.md` | raw MDX + ./src |
| 21 | [learn/i18n/usage](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/i18n/usage) | `pages/learn/i18n/usage.md` | `i18n-cache-recipes.md` | raw MDX + ./src |
| 22 | [learn/tips/structures](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/tips/structures) | `pages/learn/tips/structures.md` | `i18n-cache-recipes.md` | raw MDX + ./src |
| 23 | [learn/tips/intents](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/learn/tips/intents) | `pages/learn/tips/intents.md` | `setup-runtime.md` | raw MDX + ./src |
| 24 | [recipes/sending-messages](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/recipes/sending-messages) | `pages/recipes/sending-messages.md` | `builders.md` | raw MDX + ./src |
| 25 | [recipes/embeds-and-formatting](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/recipes/embeds-and-formatting) | `pages/recipes/embeds-and-formatting.md` | `builders.md` | raw MDX + ./src |
| 26 | [recipes/presence](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/recipes/presence) | `pages/recipes/presence.md` | `setup-runtime.md` | raw MDX + ./src |
| 27 | [recipes/cache](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/recipes/cache) | `pages/recipes/cache.md` | `i18n-cache-recipes.md` | raw MDX + ./src |
| 28 | [recipes/database](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/recipes/database) | `pages/recipes/database.md` | `i18n-cache-recipes.md` | MDX + src where applicable (recipe) |
| 29 | [recipes/logger](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/recipes/logger) | `pages/recipes/logger.md` | `i18n-cache-recipes.md` | raw MDX + ./src |
| 30 | [recipes/sharding](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/recipes/sharding) | `pages/recipes/sharding.md` | `setup-runtime.md` | raw MDX + ./src |
| 31 | [recipes/yuna](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/recipes/yuna) | `pages/recipes/yuna.md` | `i18n-cache-recipes.md` | MDX (external pkg) + core APIs vs src |
| 32 | [recipes/hot-reload](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/recipes/hot-reload) | `pages/recipes/hot-reload.md` | `setup-runtime.md` | MDX + src where applicable (recipe) |
| 33 | [recipes/api-access](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/recipes/api-access) | `pages/recipes/api-access.md` | `i18n-cache-recipes.md` | raw MDX + ./src |
| 34 | [recipes/music](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/recipes/music) | `pages/recipes/music.md` | `i18n-cache-recipes.md` | MDX (external pkg) + core APIs vs src |
| 35 | [recipes/monetization](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/recipes/monetization) | `pages/recipes/monetization.md` | `i18n-cache-recipes.md` | raw MDX + ./src |
| 36 | [recipes/cloudflare-workers](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/recipes/cloudflare-workers) | `pages/recipes/cloudflare-workers.md` | `setup-runtime.md` | raw MDX + ./src |
| 37 | [recipes/using-docker](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/recipes/using-docker) | `pages/recipes/using-docker.md` | `setup-runtime.md` | MDX + src where applicable (recipe) |
| 38 | [plugins/index](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/plugins/index) | `pages/plugins/index.md` | `plugins.md` | raw MDX + ./src |
| 39 | [plugins/official/index](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/plugins/official/index) | `pages/plugins/official/index.md` | `plugins.md` | MDX (external pkg) + core APIs vs src |
| 40 | [plugins/official/cooldown](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/plugins/official/cooldown) | `pages/plugins/official/cooldown.md` | `plugins.md` | MDX (external pkg) + core APIs vs src |
| 41 | [plugins/official/scheduler](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/plugins/official/scheduler) | `pages/plugins/official/scheduler.md` | `plugins.md` | MDX (external pkg) + core APIs vs src |
| 42 | [plugins/official/logger](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/plugins/official/logger) | `pages/plugins/official/logger.md` | `plugins.md` | MDX (external pkg) + core APIs vs src |
| 43 | [plugins/official/queues](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/plugins/official/queues) | `pages/plugins/official/queues.md` | `plugins.md` | MDX (external pkg) + core APIs vs src |
| 44 | [plugins/official/chartjs](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/plugins/official/chartjs) | `pages/plugins/official/chartjs.md` | `plugins.md` | MDX (external pkg) + core APIs vs src |
| 45 | [plugins/building/creating-plugins](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/plugins/building/creating-plugins) | `pages/plugins/building/creating-plugins.md` | `plugins.md` | raw MDX + ./src |
| 46 | [plugins/building/runtime-registration](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/plugins/building/runtime-registration) | `pages/plugins/building/runtime-registration.md` | `plugins.md` | raw MDX + ./src |
| 47 | [plugins/building/services-and-requirements](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/plugins/building/services-and-requirements) | `pages/plugins/building/services-and-requirements.md` | `plugins.md` | raw MDX + ./src |
| 48 | [plugins/building/runtime-hooks](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/plugins/building/runtime-hooks) | `pages/plugins/building/runtime-hooks.md` | `plugins.md` | raw MDX + ./src |
| 49 | [plugins/building/lifecycle-and-diagnostics](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/plugins/building/lifecycle-and-diagnostics) | `pages/plugins/building/lifecycle-and-diagnostics.md` | `plugins.md` | raw MDX + ./src |
| 50 | [testing/index](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/testing/index) | `pages/testing/index.md` | `testing.md` | MDX (external pkg) + core APIs vs src |
| 51 | [testing/writing-tests/setup](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/testing/writing-tests/setup) | `pages/testing/writing-tests/setup.md` | `testing.md` | MDX (external pkg) + core APIs vs src |
| 52 | [testing/writing-tests/commands](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/testing/writing-tests/commands) | `pages/testing/writing-tests/commands.md` | `testing.md` | MDX (external pkg) + core APIs vs src |
| 53 | [testing/writing-tests/events](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/testing/writing-tests/events) | `pages/testing/writing-tests/events.md` | `testing.md` | MDX (external pkg) + core APIs vs src |
| 54 | [testing/writing-tests/components](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/testing/writing-tests/components) | `pages/testing/writing-tests/components.md` | `testing.md` | MDX (external pkg) + core APIs vs src |
| 55 | [testing/writing-tests/modals](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/testing/writing-tests/modals) | `pages/testing/writing-tests/modals.md` | `testing.md` | MDX (external pkg) + core APIs vs src |
| 56 | [testing/writing-tests/unit-tests](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/testing/writing-tests/unit-tests) | `pages/testing/writing-tests/unit-tests.md` | `testing.md` | MDX (external pkg) + core APIs vs src |
| 57 | [testing/toolkit/mock-bot](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/testing/toolkit/mock-bot) | `pages/testing/toolkit/mock-bot.md` | `testing.md` | MDX (external pkg) + core APIs vs src |
| 58 | [testing/toolkit/dispatching](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/testing/toolkit/dispatching) | `pages/testing/toolkit/dispatching.md` | `testing.md` | MDX (external pkg) + core APIs vs src |
| 59 | [testing/toolkit/world](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/testing/toolkit/world) | `pages/testing/toolkit/world.md` | `testing.md` | MDX (external pkg) + core APIs vs src |
| 60 | [testing/toolkit/assertions](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/testing/toolkit/assertions) | `pages/testing/toolkit/assertions.md` | `testing.md` | MDX (external pkg) + core APIs vs src |
| 61 | [testing/toolkit/gateway](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/testing/toolkit/gateway) | `pages/testing/toolkit/gateway.md` | `testing.md` | MDX (external pkg) + core APIs vs src |
| 62 | [testing/toolkit/fixtures](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/testing/toolkit/fixtures) | `pages/testing/toolkit/fixtures.md` | `testing.md` | MDX (external pkg) + core APIs vs src |
| 63 | [testing/toolkit/defaults](https://seyfert-web-git-seyfert-v5-tiramisulabs.vercel.app/docs/testing/toolkit/defaults) | `pages/testing/toolkit/defaults.md` | `testing.md` | MDX (external pkg) + core APIs vs src |

Total: 63 URLs mapped.
