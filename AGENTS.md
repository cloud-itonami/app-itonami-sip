# app-itonami-sip — agent notes

## CI is an oracle, not a gate

`.github/workflows/ci.yml` runs the JVM / Clojure (and, where noted, babashka) tests as an
**oracle** job: a reference measurement only. This follows the CI exception in the workspace
`AGENTS.md` (owner decision 2026-11-07, JUN-579).

- Job names start with `oracle-` and each job prints an "ORACLE job" notice.
- It is **not a required check**. No native build, gate or packaging path depends on it, and
  a native result must never silently fall back to it.
- An Amu selfhost native gate is **not required** for this repo (owner decision 2026-11-07). It is not blocked on one either; add a native gate only if and when it is wanted.
- Some oracle jobs may fail for reasons unrelated to a change (for example a dependency repo
  that cannot be cloned). Treat a failure as a measurement to report, not as a merge condition.

## Running the `.cljk` sources as `.cljc` (kbb)

`kbb -M:kbb-test` runs `test/sip/all_test.cljk` (kbb, Node). `clojure -M:datomic:test` (the JVM
oracle CI) uses the cognitect runner, which cannot read `.cljk`, so it reports "Ran 0 tests";
that pass is empty, not a verification.

Measured 2026-11-07 (`kbb`, Node v26.0.0), one `(:require [sip.<ns>])` per namespace:

| loads under kbb | does not load, and why |
|---|---|
| assets, bible, lore, session, storyboard, xrpc (new) | `page` (java.awt.Color), `render` / `research` / `store` / `world` (java.net.http.HttpClient), `preview` (datalevin), `kotoba` / `researcher` / `ui` (cljs.core.async, browser), `schema` / `systems` / `web` (kami.scene / kami.sim from the engine SDK) |

`sip.xrpc` (added 2026-11-07) is the portable XRPC core over a host-injected
`kotoba.lang.http/IHttp`; `sip.xrpc-test` runs it over `mock-http`. `sip.store` and
`sip.research` still use their own `java.net.http` clients; moving them onto `sip.xrpc` with
one JVM transport adapter is the next step, and needs a JVM run that can load `.cljk` first
(see the dependency `.cljc` problem, JUN-967 to JUN-969).

Under kbb today: `sip.session-test` (5 tests, 13 assertions) and `sip.xrpc-test` (3 tests, 9 assertions).
`research-test` (JVM `HttpServer`), `store-test` (datalevin) and `render-test` (private
story-bible checkout) are JVM-only. Making the rest portable means replacing the JVM HTTP client,
AWT and datalevin uses, or declaring each module's host with standard reader conditionals.

## JVM oracle for `.cljk` (measured 2026-11-07)

The Kotoba monorepo has a JVM loader, `tools/cljk_loader.cljk` (kotoba-lang/kotoba), that exposes
`.cljk` to the JVM as `.cljc` and quotes the `:kotoba/*` attribute values in `ns`. With it the JVM runs
these sources (OpenJDK 21.0.1, loader from kotoba @73e3c67):

```
clojure -Sdeps '{:aliases {:t {:extra-paths ["test"]}}}' -M:datomic:t -i cljk_loader.clj -i run.clj
```

Result: `sip.session-test` and `sip.xrpc-test` give 8 tests / 22 assertions, 0 failures on the JVM,
the same as `kbb -M:kbb-test`. `sip.schema`, `sip.systems`, `sip.web` and `sip.preview` stop on
`kami.scene` / `kami.sim`: the monorepo import `lib/kotoba.kamiEngineSdk` carries only 3 of the SDK's
sources (fsm, gpu, ipc). The rest exists only in the local `kotoba-lang/kami-engine-sdk` checkout, whose
GitHub repo is gone.
