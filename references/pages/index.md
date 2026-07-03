# Page-Level Seyfert v5 Documentation Notes

One Markdown note per supplied docs URL. Each note has a page summary, **source-verified** key APIs and code examples, an explicit **Doc vs Source Corrections** section, the Seyfert source anchors used, and agent guidance. Re-check those anchors against the authoritative Seyfert source defined in `../source-truth.md`.

Count: 63 page notes plus this index.

Labels: **Source-verified** = checked against `src/**`; **External** = primarily an external package (`@slipher/*`, `yunaforseyfert`, lavalink, `@slipher/testing`) — doc-authoritative, verify version in target project; **Recipe** = pattern/ops guidance with limited core API surface.

## Pages

1. [Introduction](learn/getting-started/index.md) — Source-verified
2. [Configuring Seyfert](learn/getting-started/setup-project.md) — Source-verified
3. [Creating Your First Command](learn/getting-started/first-command.md) — Source-verified
4. [Listening to Events](learn/getting-started/listening-events.md) — Source-verified
5. [Understanding 'declare module'](learn/getting-started/declare-module.md) — Source-verified
6. [Introduction to Commands](learn/commands/intro.md) — Source-verified
7. [Command Options](learn/commands/options.md) — Source-verified
8. [Command Middlewares](learn/commands/middlewares.md) — Source-verified
9. [Sub Commands](learn/commands/subcommands.md) — Source-verified
10. [Context Menu Commands](learn/commands/context-menus.md) — Source-verified
11. [Handling Errors](learn/commands/handling-errors.md) — Source-verified
12. [Prefix Commands](learn/commands/prefix-commands.md) — Source-verified
13. [Extending CommandContext](learn/commands/extend-commandcontext.md) — Source-verified
14. [Building components](learn/components/building-components.md) — Source-verified
15. [Handling components](learn/components/handling-components.md) — Source-verified
16. [Modals](learn/components/modals.md) — Source-verified
17. [Collectors](learn/components/collectors.md) — Source-verified
18. [Creating Polls](learn/components/polls.md) — Source-verified
19. [Components v2](learn/components/v2.md) — Source-verified
20. [Supporting different languages](learn/i18n/languages.md) — Source-verified
21. [Locales Usage](learn/i18n/usage.md) — Source-verified
22. [Structures](learn/tips/structures.md) — Source-verified
23. [Gateway Intents](learn/tips/intents.md) — Source-verified
24. [Sending Messages](recipes/sending-messages.md) — Source-verified
25. [Embeds, Formatting & Attachments](recipes/embeds-and-formatting.md) — Source-verified
26. [Custom Activity and Presence](recipes/presence.md) — Source-verified
27. [Cache](recipes/cache.md) — Source-verified
28. [Database Integration](recipes/database.md) — Recipe
29. [Custom Logger](recipes/logger.md) — Source-verified
30. [Understanding sharding](recipes/sharding.md) — Source-verified
31. [Yuna Parser](recipes/yuna.md) — External
32. [Hot Reload](recipes/hot-reload.md) — Recipe
33. [Shorters and Proxy (API access)](recipes/api-access.md) — Source-verified
34. [Music Library](recipes/music.md) — External
35. [Monetization](recipes/monetization.md) — Source-verified
36. [Using Cloudflare Workers](recipes/cloudflare-workers.md) — Source-verified
37. [Docker](recipes/using-docker.md) — Recipe
38. [Plugins Introduction](plugins/index.md) — Source-verified
39. [Official Plugins](plugins/official/index.md) — External
40. [Cooldown](plugins/official/cooldown.md) — External
41. [Scheduler](plugins/official/scheduler.md) — External
42. [Logger](plugins/official/logger.md) — External
43. [Queues](plugins/official/queues.md) — External
44. [Chart.js](plugins/official/chartjs.md) — External
45. [Creating Plugins](plugins/building/creating-plugins.md) — Source-verified
46. [Runtime Registration](plugins/building/runtime-registration.md) — Source-verified
47. [Services and Requirements](plugins/building/services-and-requirements.md) — Source-verified
48. [Runtime Hooks](plugins/building/runtime-hooks.md) — Source-verified
49. [Lifecycle and Diagnostics](plugins/building/lifecycle-and-diagnostics.md) — Source-verified
50. [Testing Introduction](testing/index.md) — External
51. [Setup](testing/writing-tests/setup.md) — External
52. [Commands](testing/writing-tests/commands.md) — External
53. [Events](testing/writing-tests/events.md) — External
54. [Components](testing/writing-tests/components.md) — External
55. [Modals](testing/writing-tests/modals.md) — External
56. [Unit Tests](testing/writing-tests/unit-tests.md) — External
57. [Mock Bot](testing/toolkit/mock-bot.md) — External
58. [Dispatching](testing/toolkit/dispatching.md) — External
59. [World & State](testing/toolkit/world.md) — External
60. [Assertions](testing/toolkit/assertions.md) — External
61. [MockGateway](testing/toolkit/gateway.md) — External
62. [Fixtures](testing/toolkit/fixtures.md) — External
63. [Defaults & Scope](testing/toolkit/defaults.md) — External
