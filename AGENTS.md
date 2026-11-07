# app-itonami-sip — agent notes

## CI is an oracle, not a gate

`.github/workflows/ci.yml` runs the JVM / Clojure (and, where noted, babashka) tests as an
**oracle** job: a reference measurement only. This follows the CI exception in the workspace
`AGENTS.md` (owner decision 2026-11-07, JUN-579).

- Job names start with `oracle-` and each job prints an "ORACLE job" notice.
- It is **not a required check**. No native build, gate or packaging path depends on it, and
  a native result must never silently fall back to it.
- There is no Amu selfhost native gate for this repo yet. That is **blocked**, tracked in JUN-1007.
- Some oracle jobs may fail for reasons unrelated to a change (for example a dependency repo
  that cannot be cloned). Treat a failure as a measurement to report, not as a merge condition.

## Running the `.cljk` sources as `.cljc` (kbb)

`kbb -M:kbb-test` runs `test/sip/all_test.cljk` (kbb, Node). `clojure -M:datomic:test` (the JVM
oracle CI) uses the cognitect runner, which cannot read `.cljk`, so it reports "Ran 0 tests";
that pass is empty, not a verification.

Measured 2026-11-07 (`kbb`, Node v26.0.0), one `(:require [sip.<ns>])` per namespace:

| loads under kbb | does not load, and why |
|---|---|
| assets, bible, lore, session, storyboard | `page` (java.awt.Color), `render` / `research` / `store` / `world` (java.net.http.HttpClient), `preview` (datalevin), `kotoba` / `researcher` / `ui` (cljs.core.async, browser), `schema` / `systems` / `web` (kami.scene / kami.sim from the engine SDK) |

So `sip.session-test` (5 tests, 13 assertions) is the only suite that runs under kbb today.
`research-test` (JVM `HttpServer`), `store-test` (datalevin) and `render-test` (private
story-bible checkout) are JVM-only. Making the rest portable means replacing the JVM HTTP client,
AWT and datalevin uses, or declaring each module's host with standard reader conditionals.
