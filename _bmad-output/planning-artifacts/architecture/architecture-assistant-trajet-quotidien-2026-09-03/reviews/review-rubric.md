---
title: 'Good-Spine Checklist Review — Assistant Trajet quotidien'
reviews: 'architecture/architecture-assistant-trajet-quotidien-2026-09-03/ARCHITECTURE-SPINE.md'
against: 'prds/prd-assistant-trajet-quotidien-2026-09-02/prd.md'
date: '2026-09-03'
verdict: 'Conditional pass — one critical gap, four high-severity gaps'
---

# Review — ARCHITECTURE-SPINE.md vs. good-spine checklist

## Verdict

**Conditional pass.** The spine's core hexagonal move (AD-1 through AD-5, AD-7) is sound, its Stack table is unusually well-verified (the LiteLLM pin is reasoned against a real, dated supply-chain incident, and every checked version is confirmed current), and it is genuinely greenfield-consistent (no existing `assistant-trajet/` code in the repo to ratify or contradict). But one Deferred item removes the system's single most important shared contract right where the PRD explicitly told the architecture to settle it, and the operational/environmental envelope — specifically data persistence for the two stateful stores — is silent rather than decided or deferred. Both should be closed before this spine is used to cut epics/stories.

**Findings: 1 critical, 4 high, 2 medium, 2 low.**

---

## Dimension-by-dimension judgment

| Dimension | Judgment |
| --- | --- |
| Fixes real divergence points, misses none | Mostly yes (AD-1–AD-7 hit the real seams: layering, checkpointer scope, cross-flow continuity, LLM swap point, STT/TTS topology, transport/access). Misses the fiche-du-jour schema (C1) and the checkpoint/ namespace's place in the layering (H1). |
| Every AD's Rule is enforceable and prevents its stated divergence | Mostly yes for AD-1, AD-2, AD-4 (well-formed, checkable). AD-6 and AD-7 are enforceable for what they actually say, but the Capability Map binds them to FRs their Rule text doesn't reach (H2). AD-3's Rule text is narrower than the capability it's mapped to (H4). |
| Nothing under Deferred could let two units diverge | Fails on one item: the fiche-du-jour YAML schema (C1) is exactly this kind of item — read and written by four independently-buildable flow graphs. The other six Deferred items are correctly scoped (single-flow-internal, or non-diverging by nature). |
| Named tech is verified-current | Strong. Verified via live lookup: python-telegram-bot 22.8 (current), LiteLLM ≥1.83.0 "actuel 1.99.0" (1.83.0 is precisely the clean release after the real March 2026 TeamPCP supply-chain incident — correctly dated and reasoned), faster-whisper 1.2.1 (current), Piper OHF-Voice/piper1-gpl 1.6.0 GPL-3.0 (rhasspy/piper was indeed archived Oct 2025, license claim correct), Ollama 0.33.x (0.33.2 is the actual latest as of late Aug 2026). Two low-severity softness points: LangGraph pin is ~4 months stale with no stated reason (L1), openai SDK is unpinned unlike every other row (L2). |
| Ratifies rather than contradicts brownfield code | N/A — confirmed greenfield; no `assistant-trajet/` implementation exists anywhere in the repo. |
| Covers the driving spec's capabilities (PRD FRs) | All of FR-1…FR-16 appear in `binds:` and in the Capability → Architecture Map — nominally complete. But two mapped cells are weaker than they look (H2), so "covered" is partly cosmetic for FR-2, FR-13, FR-14. |
| Every owned dimension decided/deferred/open, esp. operational/environmental envelope | Deployment topology (Docker on Unraid, separate GPU host, Tailscale, Cloudflare Tunnel) is drawn in the Structural Seed — good. But persistence/backup for the checkpointer SQLite file and the Obsidian vault, both drawn *inside* the Docker orchestrator container, is not decided, not deferred, and not an open question anywhere in the document (H3). Environment/promotion strategy (test-phase vs. production config split) is thin (M1). |

---

## Findings

### Critical

**C1 — The fiche-du-jour YAML schema is deferred, not decided, even though the PRD routed it here and it is the system's one shared contract.**
Spine, `## Deferred`, second bullet (lines 180): *"Schéma YAML exact de la fiche du jour (noms de champs, types, convention de nommage) — Open Question #5 du PRD. Pas de chiffre fixé ici."*

The PRD's own Open Questions section is explicit that this belongs at this altitude: `prd.md` §9, item 5 — *"Schéma YAML exact de la fiche du jour (FR-7) — noms de champs, types, convention de nommage des fichiers — **architecture**."* The PRD is not deferring this further; it is handing it to the architecture spine to settle. The spine instead re-defers it downstream with no resolution.

This matters specifically because of AD-3 (spine lines 59–63): *"la fiche du jour est le seul canal de continuité inter-flux."* Four independently buildable units — `core/graphs/matin.py`, `midi.py`, `soir.py`, `patterns.py` — all read and write this file through `ObsidianStore`. Without a fixed field-name/type contract, nothing stops `matin.py` from writing `plan_matin` while `midi.py` (built later, by a different pass) expects `planMatin`, or a different date/nesting convention for "prévu vs fait." This is precisely the divergence AD-3 exists to prevent, and precisely the kind of Deferred item the checklist flags: it lets two units genuinely diverge. FR-7's Consequences (spine capability map row, PRD lines 137–141) list the field set in prose, but prose field names are not a schema — no types, no YAML key convention, no nesting shape for compound fields like "suivis pour demain + avec qui."

*Fix:* Add an AD (or a Consistency Conventions row) that at minimum fixes the YAML key names, types, and nesting for the fixed fields enumerated in FR-7, even if some field *values'* semantics stay open. Leave the true PRD Open Question #5 residue (e.g., exact naming *convention* rationale) deferred if needed, but the concrete keys other flows will literally read/write cannot be.

### High

**H1 — `checkpoint/` sits outside AD-1's three-layer dependency rule, with no stated wiring path into `core/graphs/*`.**
Structural Seed (spine lines 143–164) introduces a fourth top-level namespace, `checkpoint/sqlite_checkpointer.py`, alongside `core/`, `ports/`, `adapters/`. AD-1's Rule (lines 51) only constrains those three: *"`core/` n'importe que des symboles de `ports/`. `adapters/` implémentent `ports/` et sont seuls autorisés à importer des librairies externes."* LangGraph graphs must be compiled with a checkpointer instance, so `core/graphs/*.py` needs one from somewhere. As written, nothing forbids a graph module from importing `checkpoint/sqlite_checkpointer.py` directly — which is not `ports/`, so it would silently breach the hexagonal invariant AD-1 exists to enforce, and nothing catches it because the rule's own wording doesn't mention this namespace at all. Two developers building different flow graphs could resolve this differently (one injecting the checkpointer from `main.py`, one importing it in-module) without either violating the letter of AD-1.
*Fix:* State explicitly whether `checkpoint/` is adapter-equivalent (and therefore off-limits to `core/` imports) or is injected as a dependency into `core/graphs/*` at wiring time in `main.py` — one sentence closes this.

**H2 — The Capability → Architecture Map binds several FRs to ADs whose Rule text doesn't actually reach them.**
Spine lines 174 and 170 (Capability → Architecture Map):
- *"Gestion des erreurs (FR-13, FR-14, FR-15) → governed by AD-6."* AD-6's Rule (lines 81) is entirely about `ObsidianStore.write()` atomicity and Telegram error notification — it addresses FR-15 well. But FR-13 ("si une transcription vocale est vide ou incohérente, le système redemande par vocal plutôt que d'envoyer le contenu au LLM," PRD lines 198–203) and FR-14 ("si un message ne correspond à aucune étape attendue... demande une clarification," PRD lines 205–210) are about per-step input validation and retry/clarification behavior inside each flow graph — nothing in AD-6, or anywhere else in the spine, constrains how the four independently-built flow graphs implement "redemander" or "demander une clarification" consistently. Two flows could diverge on this with no Rule catching it, despite the capability map implying it's architecturally settled.
- *"Interface et accès (FR-1, FR-2, FR-3) → governed by AD-7."* AD-7's Rule (lines 87) covers `chat_id` verification and vocal/text channel symmetry — that's FR-1 and FR-3. FR-2 (manual button trigger, no schedule/geo) is satisfied only implicitly, by the absence of any scheduler/geolocation component anywhere in the Structural Seed — not by any sentence in AD-7 or elsewhere that actually rules out adding one later.
*Fix:* Either tighten the map (don't claim AD-6/AD-7 govern FR-13/FR-14/FR-2), or add the missing Rule content (e.g., an AD-1-adjacent statement that no flow graph or adapter may add a scheduled/geolocation trigger; a shared retry/clarification convention for FR-13/FR-14, even a one-line "the reformulation behavior in FR-6's Notes generalizes to STT/ambiguity handling across all flows").

**H3 — No persistence/backup decision for the two stateful stores drawn inside the Docker orchestrator container.**
Structural Seed (spine lines 114–126) draws the SQLite checkpointer and the Obsidian vault both *inside* the `Unraid — Docker (orchestrateur)` subgraph, with no volume/bind-mount notation, and neither the Stack table, the Deferred section, nor the Consistency Conventions table mentions backup, retention, or persistence-across-container-recreation for either store. This is the operational/environmental envelope the checklist calls out by name as commonly missed, and here it is genuinely silent — not decided, not deferred, not flagged as an open question. It's a real gap given what the product is *for*: the Vision (PRD line 18) and JTBD #3 (PRD line 26, "retrouver un historique fiable plutôt qu'un vide") make the vault's durability the product's core value, and SM-3 measures perceived reliability of exactly this data. A Docker container recreated without an explicit volume mount loses both the vault and the checkpointer silently.
*Fix:* At minimum, state the volume/bind-mount strategy for `vault/` and the checkpointer's SQLite file, and either commit to a backup cadence or explicitly defer it with a stated reason (the Deferred section already does this well for the checkpointer's *cleanup*, lines 185 — the same treatment is needed for *loss prevention*, which is currently absent even as a deferred item).

**H4 — AD-3's Rule text scopes `ObsidianStore` access to `read_today()` only, but is mapped to FR-10–FR-12 which need multi-day historical reads.**
AD-3 Rule (spine line 63): *"un flux ne connaît le contenu produit par un flux précédent que via `ObsidianStore.read_today()`."* Capability Map (line 173) binds AD-3 to *"Analyse de patterns (FR-10, FR-11, FR-12)"*, but pattern analysis (FR-11, PRD lines 171–178) fundamentally requires reading a *selection of past fiches*, not just today's — `read_today()` as the sole named method cannot serve it. The spine's Deferred section (line 181) does defer the *search mechanics* ("Mécanique exacte de la recherche ciblée," Open Question #2), but that's a different question from whether `ObsidianStore` even exposes a multi-day/range read method consistent with AD-3's stated invariant, or whether patterns.py is meant to access the vault a different way. As written, AD-3's Rule and its own Capability Map row are in tension.
*Fix:* Either broaden AD-3's Rule to name the (unspecified-shape) historical-read method it also governs, or narrow the Capability Map row to note that FR-10–12's read path is a distinct, still-open port method — currently it reads as resolved when it isn't.

### Medium

**M1 — Environment/promotion strategy for test-phase vs. production is thin.**
The PRD (§5, lines 236 and §7.1, line 258) treats the LLM-API test phase vs. self-hosted production as a real environment split with a gate (SM-4, PRD line 281–282, "vérifié par revue technique avant bascule"). AD-4 (spine lines 68–69) correctly makes the model-routing switch config-only ("jamais un changement de code côté orchestrateur"), which is good design, but nothing in the spine ties this to how the *other* environment-dependent config (e.g. which `AUTHORIZED_CHAT_ID`, which `OBSIDIAN_VAULT_PATH`, whether a review checklist exists) differs between the test phase and production, or says SM-4's manual review is out of scope for this spine. Low cost to add a sentence; currently silent.

**M2 — Cloudflare Tunnel and Tailscale are load-bearing in the Structural Seed but absent from the Stack table.**
Spine lines 139–140 show both as the transport for the Telegram webhook and the STT call respectively — both are single points of failure for their flows, yet neither appears in the Stack table (lines 99–109) the way every other dependency does. As hosted services they don't need a version pin, but their absence from the one table meant to enumerate "named tech" is a minor completeness gap.

### Low

**L1 — LangGraph pin (1.1.6) is not the verified-current release, with no stated rationale.**
Spine line 102 pins `LangGraph 1.1.6`. Live lookup confirms the current release is 1.2.1 (released May 21, 2026), so as of this spine's own creation date (2026-09-03) the pin is roughly four months behind latest. Not necessarily wrong — but unlike the LiteLLM row, which explains *why* it floors at 1.83.0 (a dated, real supply-chain incident), this pin gives no reason, so a reader can't tell deliberate-and-stable from stale.

**L2 — `openai` SDK entry is unpinned ("dernière stable"), inconsistent with every other Stack row.**
Spine line 109. Every other dependency in the Stack table is pinned to a specific version or version floor; this one — notably the client library for the system's single LLM-swap seam governed by AD-4 — is left as "latest stable," which drifts silently and is the one row in the table not actually reproducible.

---

## What's working well (not findings, noted for calibration)

- AD-1/AD-2/AD-4 are exemplary: crisp, checkable rules that map cleanly onto a real, stated divergence risk.
- The Stack table's version claims are unusually well-grounded — the LiteLLM floor is tied to a real, correctly-dated incident, and the Piper license-change note (MIT → GPL-3.0 fork) is accurate.
- `binds:` and the Capability → Architecture Map achieve nominal 1:1 coverage of all 16 PRD FRs — the shape of the coverage exercise is right, even where specific cells are weaker than they look (see H2, H4).
- Confirmed greenfield: no existing `assistant-trajet/` implementation in the repository for this spine to ratify or contradict.
