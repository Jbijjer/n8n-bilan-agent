# PRD Quality Review — Assistant Trajet quotidien

## Overall verdict

A well-calibrated capability-spec PRD for a single-operator personal tool: the thesis is genuinely specific (meeting Sébastien in the car rather than demanding dedicated time), theater is largely absent, and Open Questions/Non-Goals are used honestly rather than as filler. The main risk is a real, unaddressed contradiction between the "self-hosted / confidentiality" constraint (§5, SM-4) and Telegram being the sole channel for all bilan content — this is the kind of trade-off the PRD otherwise surfaces well elsewhere, so its absence here stands out. A handful of done-ness and mechanical gaps (a binary FR-1 behavior left unresolved, no latency bound for a car-trip-timed interaction, a section-numbering drift) are worth a pass before this feeds architecture.

## Decision-readiness — adequate

The PRD is unusually good at naming trade-offs elsewhere: SM-1 explicitly rejects a numeric usage quota "délibéré : un seuil rigide pénaliserait vacances, télétravail, semaines sans trajet — sans que ce soit un échec du produit" (§8), SM-C1 names the tension between growing usage and never insisting twice ("un système qui pousse plus fort pour faire monter l'usage (SM-1) casse la contrainte... (FR-6)"), and the orchestration choice (§9 Open Question #1) is left genuinely open with a real spike plan in the addendum rather than smoothed over. This is the opposite of the rubric's red flag.

That makes the one place it doesn't do this more conspicuous.

### Findings

- **critical** Telegram-as-only-channel is never reconciled with the self-hosting/confidentiality constraint (§5 Constraints; §8 SM-4) — §5 states "Le contenu des bilans... ne transite jamais par un service hors de l'infrastructure de Sébastien une fois en production," and SM-4 is a binary production gate on exactly this claim. But §4.1 makes Telegram "Point d'entrée unique du système," and Telegram bot traffic (vocal audio included, per FR-3) is inherently relayed through Telegram's own cloud servers — there is no self-hosted Telegram transport. Given the bilans may contain "des détails de travail secteur public" (§0), this is a substantive, unacknowledged tension, not a cosmetic one, and it sits exactly where the PRD's confidentiality gate is supposed to be airtight. *Fix:* either scope SM-4/§5 explicitly to exclude the Telegram transport layer with a stated rationale (encryption in transit, single authorized chat_id, accepted residual risk), or flag it as an open `[NOTE FOR PM]` risk-acceptance decision for Sébastien to confirm explicitly before production bascule.

## Substance over theater — strong

No persona theater (single persona, explicitly "forme allégée," §2.2), no innovation theater — the PRD pre-empts the risk itself: "Pas d'ambition d'innovation algorithmique ou de moat technique — la valeur est dans l'ajustement à l'usage réel, pas dans la nouveauté" (§6). Constraints (§5) are product-specific, not boilerplate ("Sécurité des accès. Identifiants et secrets ne sont jamais exposés en clair ni versionnés... Mécanisme précis laissé à l'architecture" — concrete and appropriately scoped). The Vision (§1) is idiosyncratic to this product (trajet en voiture, three fixed daily windows, Obsidian vault) and would not drop into another PRD unchanged. No findings.

## Strategic coherence — strong

The thesis is explicit and load-bearing: meet Sébastien in an already-available moment rather than demand a new one (§1). Feature shape follows from it directly — the midi flow is capped at "au maximum 4 échanges" to fit a ~20-minute trip (FR-5), the soir flow is one-question-at-a-time to suit voice while driving (FR-6). Success metrics validate the thesis rather than measuring activity: SM-1 is deliberately qualitative to avoid punishing normal life variance, SM-2 targets "0 occurrence" of a repeated question — a crisp, thesis-relevant number, not a vanity metric. A counter-metric (SM-C1) is named. No findings.

## Done-ness clarity — adequate

Every FR (1–16) carries "Consequences (testable)" bullets, and most are genuinely verifiable — e.g. FR-6's reformulation rule is disambiguated with an explicit note ("s'applique par point du bilan (jusqu'à 6 reformulations possibles)... confirmé par Sébastien," §4.4), which is exactly the kind of precision this dimension rewards. No instances of "gracefully," "reasonable performance," or "user-friendly" were found. But two gaps would leave an engineer guessing:

### Findings

- **medium** FR-1's consequence names two different, non-equivalent behaviors without choosing one (§4.1): "Tout message d'un `chat_id` différent est ignoré silencieusement ou rejeté, jamais traité par un flux." Silent-ignore and explicit-reject have different security/observability implications (e.g. whether an unauthorized sender can tell the bot exists) and can't both be "the" testable consequence. *Fix:* pick one and state it as the sole behavior.
- **medium** No latency or response-time bound exists anywhere in the PRD despite the trajet window being a hard, safety-relevant design constraint (FR-5 caps the midi flow at "~20 minutes" / 4 échanges specifically because it happens while driving). Nothing in §5 Constraints or any FR states an acceptable bound for the STT→LLM→TTS round trip, which matters more once the system moves to a local LLM (per the addendum, Qwen2.5 14B / Llama 3.1 8B on a 16GB GPU). *Fix:* add a constraint with a concrete bound, e.g. "each agent turn responds within N seconds of Sébastien's message."

## Scope honesty — adequate

Non-Goals (§6) does real work — five explicit, specific exclusions, not filler ("Pas de correction ou de jugement sur le contenu des bilans... le système capture, il ne critique pas"). The Assumptions Index (§10) is honest about its own state: "Toutes les hypothèses inférées ont été confirmées avec Sébastien... aucune ouverte à ce stade" — consistent with zero inline `[ASSUMPTION]` tags in the body. De-scoping in §7.2 gives reasons, not silent omission ("différés, non priorisés," "prévue mais postérieure au MVP"). Open-items density (5 Open Questions, all architecture-level, zero open assumptions) is proportionate to a personal, low-stakes tool feeding straight into architecture.

### Findings

- **low** No inline `[NOTE FOR PM]` callouts appear anywhere in the document, even though §0 sets up a tagging convention (for `[ASSUMPTION]`) that implies similar rigor elsewhere. Real deferred tensions exist at the point they arise — e.g. FR-11's "sélection pertinente" in the history search, and FR-12's field-addition mechanism ("Out of Scope : Le mécanisme exact d'ajout... est un point ouvert," §4.6) — but are only recoverable via the numbered list in §9, not flagged inline. Functionally captured, just less discoverable when a section is read standalone. *Fix:* add brief `[NOTE FOR PM]` tags at §4.6 pointing to Open Questions #2 and #4.

## Downstream usability — adequate

This PRD is chain-top (explicit direct input to `bmad-architecture`, §0), so this dimension carries real weight. FR IDs (1–16) are contiguous and unique; UJ IDs (1–4) are each realized by name in a feature section ("Réalise UJ-1" §4.2, UJ-2 §4.3, UJ-3 §4.4, UJ-4 §4.6) — no floating UJs. SM cross-references resolve both ways (SM-1 "Valide FR-4, FR-6"; the Constraints section reciprocally states "Valide SM-4"). All UJs carry a named protagonist (Sébastien) with context inline.

### Findings

- **medium** Section numbering drift: §7 is titled "7. MVP Scope" but its subsections are numbered "6.1 In Scope" and "6.2 Out of Scope for MVP" (lines directly under §7, not §6). Any downstream tool or reader extracting by section number will resolve these to the wrong parent section. *Fix:* renumber to 7.1 / 7.2.
- **low** The single most-used domain noun in the document, "bilan," is never defined in the Glossaire (§3) — "Fiche du jour," "Flux," "Trajet," and four other terms are, but "bilan" (used in button labels "Bilan matin/midi/soir," in the Vision, in SM-4's "contenu de bilan," etc.) is left informal. It's not clear whether "bilan" is synonymous with "Flux," with the resulting "Fiche du jour," or is the general activity both formalize. *Fix:* add a one-line Glossaire entry for "bilan" clarifying its relationship to "Flux" and "Fiche du jour."

## Shape fit — strong

Single-operator personal tool, correctly shaped as a lightweight capability spec rather than a heavyweight consumer-product PRD: UJs exist but are deliberately "une phrase par parcours" (§2.2) with detail pushed to the feature FRs where it belongs, and the PRD explicitly justifies skipping a Non-Users section given the single-user context is already established in the brief (§2.1). No over-formalization (no persona proliferation, no forced multi-stakeholder framing) and no under-formalization (UJs and a Glossary still exist where they earn their keep, given this feeds architecture). No findings.

## Mechanical notes

- **Section numbering:** §7 "MVP Scope" contains subsections labeled 6.1/6.2 instead of 7.1/7.2 (see Downstream usability finding above).
- **Glossary gap:** "bilan," the PRD's most central domain noun, has no Glossaire entry (see Downstream usability finding above); the seven terms that are defined are used consistently throughout (no case/plural drift found for Fiche du jour, Flux, Trajet, Continuité intra-journée, Champ "Autres", Pattern, chat_id).
- **ID continuity:** FR-1 through FR-16 contiguous, unique, no gaps. UJ-1 through UJ-4 contiguous, each cross-referenced from the feature that realizes it. SM-1/2/3, SM-C1, SM-4 all cross-reference FRs that exist and resolve both directions.
- **Assumptions Index roundtrip:** Trivially satisfied — zero inline `[ASSUMPTION]` tags in the body, and §10 explicitly states all inferred assumptions were confirmed externally (via `.memlog.md`, not reviewed here) rather than left open.
- **UJ protagonist naming:** All four UJs name Sébastien explicitly with situational context inline (time of day, duration, trigger action) — no floating UJs.
