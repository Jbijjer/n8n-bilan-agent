# Réconciliation — brief.md vs prd.md

**Intrant :** `_bmad-output/planning-artifacts/briefs/brief-assistant-trajet-quotidien-2026-09-02/brief.md`

## Couverture

| Élément du brief | Couvert dans le PRD |
|---|---|
| Executive Summary / Solution | §1 Vision |
| Success Criteria 1 (usage soutenu) | SM-1 (révisé en qualitatif) |
| Success Criteria 2 (continuité) | SM-2 |
| Success Criteria 3 (fiabilité bilan soir) | SM-3 |
| Success Criteria 4 (confidentialité dès prod) | SM-4 (gate) — mais uniquement comme métrique, pas comme contrainte explicite |
| Scope in (flux, vocal, mémoire, patterns, phase test→local) | §4, §6.1 |
| Scope out (rappels proactifs, Jira, déclenchement auto) | §5 Non-Goals |
| What Makes This Different (pas de moat technique) | §5 Non-Goals |
| Vision long terme (bascule 100% locale) | §6.2 (out of scope MVP, référencé) |

## Gap trouvé

Le brief présente "Full self-hosted" comme une **contrainte non négociable** (pas seulement un critère de succès mesuré a posteriori). Le PRD ne la formule qu'indirectement (SM-4, MVP scope) — pas de section "Constraints and Guardrails" qui la pose comme frontière contraignante pour l'architecture, alors que le produit porte clairement ce concern (secteur public, confidentialité). **Fix : ajouter une section §5 Constraints and Guardrails.**

Aucun autre gap.
