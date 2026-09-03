# Réconciliation — addendum.md (brief) vs prd.md

**Intrant :** `_bmad-output/planning-artifacts/briefs/brief-assistant-trajet-quotidien-2026-09-02/addendum.md`

## Couverture

| Élément de l'addendum | Couvert dans le PRD |
|---|---|
| Contrainte 1 — Indépendance LLM | FR-16 |
| Contrainte 2 — Full self-hosted | Indirect seulement (voir gap ci-dessous) |
| Contrainte 3 — Confidentialité secteur public | SM-4 (gate) |
| Contrainte 4 — Pas d'automatisation silencieuse | FR-10, FR-12 |
| Contrainte 5 — Accès réseau (Tailscale/Cloudflare) | Correctement exclu (implémentation — "how") |
| Contrainte 6 — Déclenchement manuel | FR-2, §5 Non-Goals |
| Stockage — fiche du jour | FR-7, Glossaire |
| Mémoire et continuité | FR-8, FR-9 |
| Orchestration | Correctement exclu — Open Question #1 |
| Vocal (STT/TTS) | Capacité couverte (FR-3) ; techno spécifique correctement exclue |
| LLM (test→local) | FR-16, §6.1 |
| Prompts validés | Correctement exclus (implémentation) — comportement équivalent capturé dans FR-4/5/6 |
| Gestion des erreurs | FR-13, FR-14, FR-15 |
| Sécurité (credentials chiffrés) | Mécanisme correctement exclu (implémentation) ; principe non repris explicitement dans le PRD |
| Séquence de tests | Hors PRD à raison — matière pour `bmad-create-epics-and-stories` |
| Ouvert à l'approfondissement (schéma, recherche, switch LLM, nouveau champ) | Open Questions #2-5 |
| Ouvert à l'approfondissement (liste nodes n8n, stories) | Correctement exclus — dépend du choix d'orchestration / phase epics |

## Gaps trouvés

1. **Contrainte 2 (self-hosted)** — même gap que reconcile-brief.md : mérite une section Constraints and Guardrails explicite.
2. **Sécurité — principe des credentials** — le mécanisme précis (n8n credential store) est bien de l'implémentation, mais le *principe* ("jamais en clair, jamais commit en git") est une frontière contraignante indépendante de l'outil choisi. Vaut une ligne dans la même section Constraints and Guardrails plutôt qu'un oubli total.

**Fix : même section §5 Constraints and Guardrails, 4 bullets (self-hosting, confidentialité, pas d'automatisation silencieuse, sécurité des accès).**
