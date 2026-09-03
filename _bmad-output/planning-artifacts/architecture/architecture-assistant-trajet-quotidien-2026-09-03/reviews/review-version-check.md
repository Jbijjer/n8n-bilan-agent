# Review — Version/Currency Check of the Stack Table

**Target:** `ARCHITECTURE-SPINE.md` § Stack
**Method:** live web search per entry (PyPI/GitHub release pages, official changelogs), performed 2026-09-03.
**Verdict key:** ✅ confirmed accurate · ⚠️ accurate but stale (a newer version now exists) · ❌ wrong/needs correction

---

## Python 3.12+  → ✅ confirmed, accurate

- Latest 3.12 patch: **3.12.14**, released 2026-08-12 (security-only maintenance branch).
- Python 3.12 is out of feature development (3.14 is now the current feature series) but still receives security patches, expected through ~Oct 2028.
- The spine pins a floor (`3.12+`), not an exact patch — this is consistent with actual support status. No correction needed.

Sources: python.org/downloads/latest/python3.12, peps.python.org/pep-0693, devguide.python.org/versions, endoflife.date/python

---

## LangGraph 1.1.6  → ⚠️ accurate as a real release, but noticeably stale

- Confirmed: **1.1.6** is a real PyPI release, dated **2026-04-03**.
- Confirmed via PyPI release history: current latest stable is **1.2.11** (2026-08-11). Full chain between: 1.1.7 (yanked — "custom callback handler bug"), 1.1.8, 1.1.9, 1.1.10, 1.2.0, 1.2.4–1.2.10, 1.2.11.
- So the pinned version is ~4 months and 8+ releases behind current latest as of the architecture's own date (2026-09-03).
- Not "wrong" (it exists, wasn't yanked, no known vulnerability found), but this is the one entry most likely to have been asserted from training-data familiarity with LangGraph 1.x rather than checked against the actual release cadence. **Flag for the architect:** confirm whether 1.1.6 was deliberately pinned for a specific reason (tested compatibility, checkpointer API stability) — if not, recommend bumping to a current 1.2.x, or at minimum documenting why 1.1.6 specifically.

Sources: pypi.org/project/langgraph/#history, github.com/langchain-ai/langgraph/releases

---

## python-telegram-bot 22.8 (extra `[webhooks]`)  → ✅ confirmed accurate, currently latest

- Confirmed: **22.8** released 2026-06-12, and it is still the latest stable release as of the search date — no 22.9/23.x found.
- `[webhooks]` extra is a real, current install extra for this library (needed for the webhook-based transport AD-7 describes).

Sources: pypi.org/project/python-telegram-bot/#history, docs.python-telegram-bot.org/en/v22.8/changelog.html

---

## LiteLLM ≥1.83.0, currently 1.99.0  → ✅ confirmed accurate, both the floor and the "currently"

- **Floor claim (≥1.83.0, "incident supply-chain corrigé"):** confirmed. LiteLLM suffered a real supply-chain compromise: malicious code was injected into **1.82.7** and **1.82.8** (traced to a compromised Trivy scanner dependency in LiteLLM's CI, exploited by threat actor "TeamPCP", disclosed ~March 2026, impacting 2,500+ downstream orgs). **1.83.0** is explicitly documented by the LiteLLM team as "Official Release (Post Supply Chain Incident)" — the first release built through a rebuilt, isolated CI/CD pipeline. So "jamais en-deçà de 1.83.0" is a well-grounded, correctly-cited constraint, not an assertion.
- **"Currently 1.99.0":** confirmed as the current *stable* release (dated ~2026-09-01, essentially the day this table's data was gathered). Newer pre-release/dev builds exist (1.100.0rc1, 1.101.0.dev1) but nothing newer is stable. Accurate for the architecture's date.

Sources: docs.litellm.ai/release_notes/v1.83.0/v1-83-0, docs.litellm.ai/blog/security-update-march-2026, docs.litellm.ai/blog/security-hardening-april-2026, socradar.io/blog/litellm-supply-chain-attack, pypi.org/project/litellm/#history

---

## Ollama 0.33.x  → ✅ confirmed accurate

- Confirmed: latest is **0.33.2**, released 2026-08-27 (following 0.33.0 and 0.33.1 earlier in August 2026). The `0.33.x` pin correctly matches the currently-active minor series.

Sources: releasealert.dev/dockerhub/ollama/ollama, localaimaster.com/blog/ollama-version-history, github.com/ollama/ollama/releases

---

## faster-whisper 1.2.1 (service STT, machine GPU)  → ⚠️ accurate as "latest", but worth flagging as stale

- Confirmed: **1.2.1** is genuinely the latest tagged release on `SYSTRAN/faster-whisper` — no newer version exists. So the version number itself is correct, not an error.
- However, that release is dated **2025-10-31** — i.e., no new release in roughly 10 months as of 2026-09-03, despite the repo remaining active (open issues/PRs, non-archived). This is a real currency risk the spine doesn't surface: a pinned "latest" version that hasn't moved in 10 months is either genuinely stable, or a sign the project's release cadence has slowed — worth a one-line note in the spine (or Deferred) rather than treating "1.2.1" as self-evidently current, since a reader in a few months won't know whether 1.2.1 is still latest without re-checking.

Sources: github.com/SYSTRAN/faster-whisper/releases, pypi.org/project/faster-whisper (history)

---

## Piper TTS — OHF-Voice/piper1-gpl 1.6.0, GPL-3.0  → ⚠️ mostly accurate; one point confirmed, one now stale

- **License-change narrative (rhasspy/piper MIT repo archived Oct 2025, active fork is OHF-Voice/piper1-gpl under GPL-3.0):** confirmed exactly. `rhasspy/piper` was archived by its owner on **2025-10-06** and is read-only; development moved to `OHF-Voice/piper1-gpl`, which is GPL-3.0 licensed. This is a correctly-researched, non-obvious fact (a license change on a repo migration is exactly the kind of thing training data would get wrong or miss) — good that it's called out explicitly in the spine.
- **Version 1.6.0:** was indeed a real release (2026-07-23), but it is **no longer current**. The actual latest release as of 2026-09-03 is **v1.7.0** (2026-08-15, adds a Japanese phonemizer via OpenJTalk). The spine is one minor version behind. Not incorrect in the sense of citing a nonexistent version, but it should be bumped to 1.7.0 or explicitly marked as an intentional pin.

Sources: github.com/OHF-Voice/piper1-gpl/releases, github.com/OHF-Voice/piper1-gpl/tags, github.com/rhasspy/piper (archived notice)

---

## openai SDK (client HTTP → LiteLLM)  → ✅ accurate as written (deliberately unpinned)

- The spine intentionally does not pin a version ("dernière stable" / latest stable), which is a reasonable and low-risk choice for a thin OpenAI-compatible HTTP client pointed at LiteLLM.
- For reference at time of check: current latest is **openai-python 3.7.0** (released 2026-09-02, essentially the architecture's own date). No correction needed since no specific version is asserted.

Sources: pypi.org/project/openai/#history, github.com/openai/openai-python

---

## Overall

| Entry | Verdict |
| --- | --- |
| Python 3.12+ | ✅ confirmed |
| LangGraph 1.1.6 | ⚠️ real, but ~4 months / 8 releases behind current (1.2.11) — flag for deliberate-pin confirmation |
| python-telegram-bot 22.8 | ✅ confirmed, currently latest |
| LiteLLM ≥1.83.0 (currently 1.99.0) | ✅ confirmed, incident narrative verified, "currently" accurate |
| Ollama 0.33.x | ✅ confirmed, currently latest |
| faster-whisper 1.2.1 | ⚠️ correct as "latest" but stale (no release in ~10 months) — worth a note |
| Piper OHF-Voice/piper1-gpl 1.6.0, GPL-3.0 | ⚠️ license/archival story confirmed accurate; version is one release behind current (1.7.0) |
| openai SDK ("dernière stable") | ✅ accurate as an unpinned policy; current is 3.7.0 |

No entry was found to be factually wrong (no nonexistent version, no dead/renamed project, no mischaracterized license). Two entries (LangGraph, Piper) cite a version that is real but has since been superseded and should be bumped or explicitly justified as a deliberate pin; one entry (faster-whisper) is accurate but its staleness (10 months without a release) isn't visible from the spine text alone and is worth a one-line caveat.
