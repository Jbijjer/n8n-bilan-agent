---
id: SPEC-assistant-trajet-quotidien
companions:
  - '../../planning-artifacts/architecture/architecture-assistant-trajet-quotidien-2026-09-03/ARCHITECTURE-SPINE.md'
  - 'glossary.md'
  - 'prompts.md'
  - 'test-sequence.md'
sources:
  - '../../planning-artifacts/prds/prd-assistant-trajet-quotidien-2026-09-02/prd.md'
  - '../../briefs/brief-assistant-trajet-quotidien-2026-09-02/brief.md'
  - '../../briefs/brief-assistant-trajet-quotidien-2026-09-02/addendum.md'
---

> **Canonical contract.** This SPEC and the files in `companions:` are the complete, preservation-validated contract for what to build, test, and validate. Source documents listed in frontmatter are for traceability — consult them only if you need narrative rationale or prose color this contract intentionally omits.

# Assistant Trajet quotidien

## Why

Sébastien, programmeur/analyste secteur public diagnostiqué TDA, a besoin de structurer sa planification quotidienne et ses bilans sans l'effort de rédaction et d'ouverture d'appli que le TDA rend coûteux (pain à résoudre). Le trajet en voiture — déjà présent dans sa routine, trois fois par jour — devient le moment naturel pour ce rituel via un assistant conversationnel vocal/texte sur Telegram, plutôt que d'exiger un moment dédié.

## Capabilities

- **CAP-1** — Interface conversationnelle et accès
  - **intent:** Sébastien peut interagir avec l'assistant exclusivement via Telegram (son `chat_id` uniquement), déclencher un des trois flux manuellement via un clavier à 3 boutons, et recevoir une réponse dans le même canal que son message (vocal→vocal, texte→texte). Réalise FR-1, FR-2, FR-3.
  - **success:** Un `chat_id` non autorisé ne reçoit aucune réponse ; aucun flux ne démarre sans action explicite ; un message vocal ne reçoit jamais une réponse texte et inversement.

- **CAP-2** — Flux matin
  - **intent:** Sébastien peut obtenir, en une séquence d'étapes fixe et sans saut, un rappel du bilan d'hier, l'état des tâches actives, et 2-3 priorités du jour proposées puis validées. Réalise FR-4.
  - **success:** Le flux cite le contenu de la veille s'il existe, se termine par un résumé de 2-3 lignes confirmé par Sébastien, et le contenu confirmé est écrit dans la fiche du jour.

- **CAP-3** — Flux midi
  - **intent:** Sébastien peut ajuster les priorités de l'après-midi en au maximum 4 échanges, à partir du plan du matin déjà connu, sans que ce soit un nouveau bilan. Réalise FR-5.
  - **success:** Aucune question déjà répondue dans la fiche du jour n'est reposée ; le flux se termine au plus tard au 4e échange par un résumé d'une ligne confirmé.

- **CAP-4** — Flux soir
  - **intent:** Sébastien peut faire, une question à la fois et dans l'ordre, le point sur le statut des tâches, les imprévus, ce qui reste à faire, les suivis pour demain, son feeling général et sa procrastination estimée — en s'appuyant sur ce que matin/midi ont déjà établi. Réalise FR-6.
  - **success:** Aucune reformulation au-delà d'une fois par point ; tout contenu hors des 6 points va dans le champ "Autres" ; le flux se termine par un résumé de 3-4 lignes confirmé et écrit dans la fiche du jour.

- **CAP-5** — Continuité et mémoire
  - **intent:** Le système peut faire persister le contexte d'une journée entre les flux matin/midi/soir sans mémoire de session partagée entre eux, et garder le contexte des échanges à l'intérieur d'un même flux. Réalise FR-7, FR-8, FR-9.
  - **success:** Le flux midi voit le plan matin même dans une session distincte ; deux flux du même jour n'accèdent jamais aux échanges bruts l'un de l'autre, seulement au contenu déjà écrit dans la fiche.

- **CAP-6** — Analyse de patterns à la demande
  - **intent:** Sébastien peut demander explicitement une analyse de ses bilans passés ; le système répond factuellement et, si un thème vraiment récurrent apparaît, propose (sans jamais l'appliquer seul) un nouveau champ structuré. Réalise FR-10, FR-11, FR-12.
  - **success:** Aucun déclenchement automatique n'existe ; en l'absence de pattern net, le système le dit explicitement plutôt que d'extrapoler ; aucune fiche passée n'est modifiée par cette fonction.

- **CAP-7** — Gestion des erreurs
  - **intent:** Le système peut réagir à une entrée inexploitable ou un échec technique sans jamais perdre un bilan silencieusement : redemander en cas de transcription vide, demander clarification en cas d'ambiguïté, notifier Sébastien sur Telegram en cas d'échec technique. Réalise FR-13, FR-14, FR-15.
  - **success:** Un audio inexploitable ne produit jamais d'appel LLM sur un contenu vide ; aucune information non fournie par Sébastien n'apparaît dans la fiche ; un échec technique ne produit jamais une fiche incomplète sans notification associée.

- **CAP-8** — Indépendance du modèle LLM
  - **intent:** Le système peut faire passer tous les flux par un point d'appel LLM unique et isolé du reste de la logique produit. Réalise FR-16.
  - **success:** Remplacer le modèle ou le fournisseur LLM ne nécessite de modifier qu'un seul point du système, jamais la logique propre à un flux.

## Constraints

- **Self-hosting** — aucune dépendance à un service cloud tiers pour les données sensibles en production (orchestration, stockage, LLM, vocal). Exception assumée et temporaire : phase de test sur LLM API externe.
- **Confidentialité** — le contenu des bilans (potentiellement des détails de travail secteur public) n'est stocké et traité que sur l'infrastructure de Sébastien en production. Le transport réseau via Telegram est un risque résiduel explicitement accepté (chiffrement en transit, `chat_id` unique) — pas une violation de cette contrainte.
- **Pas d'automatisation silencieuse** — aucune modification du schéma de données ni réécriture d'une fiche passée sans validation humaine explicite.
- **Sécurité des accès** — identifiants et secrets jamais en clair ni versionnés, quel que soit l'outil retenu.
- **Temps de réponse = facteur de sécurité** — le trajet se fait en conduisant ; le temps de réponse ne doit jamais donner l'impression d'attendre activement la machine. Qualitatif par choix délibéré, pas de borne chiffrée dans ce contrat — à mesurer empiriquement en implémentation.
- **Ne jamais optimiser l'usage au prix de la rigueur** — un système qui pousse plus fort pour faire monter la fréquence d'usage (voir Success signal) casse la règle "jamais insister deux fois" de CAP-4. Contre-indicateur explicite, pas une cible à améliorer.
- **Paradigme, stack, topologie matérielle et schéma de données** sont fixés par le spine d'architecture (companion adopté) — non répétés ici ; ce SPEC décrit le QUOI, pas le COMMENT.

## Non-goals

- Rappels proactifs spontanés (le système ne parle jamais sans que Sébastien ait initié un flux).
- Intégration avec un outil de scrum d'équipe (Jira ou autre).
- Déclenchement automatique par géolocalisation ou horaire.
- Ambition d'innovation algorithmique ou de moat technique — la valeur est dans l'ajustement à l'usage réel.
- Correction ou jugement sur le contenu des bilans (tâches non faites, procrastination, imprévus) — le système capture, il ne critique pas.

## Success signal

Sébastien utilise les flux matin et soir régulièrement, sans que ce soit ressenti comme une corvée, et n'a jamais à répéter une information déjà donnée dans un flux précédent le même jour. En se relisant une semaine plus tard, il retrouve un historique de bilans honnête et fiable. Avant toute bascule en production, aucun contenu de bilan n'est stocké ni traité hors de son infrastructure (vérifié par revue technique, pas mesuré en continu).

## Open Questions

- Mécanique exacte de sélection des entrées pertinentes pour la recherche historique de CAP-6 (grep par plage de dates vs requêtes structurées) — PRD Open Question #2, spine AD-3 (Deferred).
- Mécanisme précis de la proposition/ajout d'un nouveau champ structuré pour CAP-6 (manuel ou semi-automatisé) — PRD Open Question #4, spine AD-9 (Deferred).
