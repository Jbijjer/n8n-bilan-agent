---
title: "Reconciliation — PRD vs Architecture Spine (Assistant Trajet quotidien)"
purpose: "Vérifier que chaque exigence, contrainte et attente qualitative du PRD a un point d'atterrissage explicite dans ARCHITECTURE-SPINE.md"
sources:
  - '_bmad-output/planning-artifacts/prds/prd-assistant-trajet-quotidien-2026-09-02/prd.md'
  - '_bmad-output/planning-artifacts/architecture/architecture-assistant-trajet-quotidien-2026-09-03/ARCHITECTURE-SPINE.md'
created: '2026-09-03'
---

# Reconciliation — PRD → Architecture Spine

## Méthode

Passage systématique : (1) chaque FR-1..16, (2) §5 Constraints and Guardrails, (3) §6 Non-Goals, (4) §8 Success Metrics (portion architecturale), (5) §9 Open Questions, (6) attentes qualitatives/de ton non capturées par un ID (§1 Vision, glossaire, notes de flux). Pour chaque item : où il atterrit dans la spine (AD / convention / stack / seed / Deferred), ou constat d'absence.

## 1. FR-1..16 — couverture

| FR | Exigence (résumé) | Atterrissage dans la spine | Verdict |
| --- | --- | --- | --- |
| FR-1 | Restriction chat_id, ignoré silencieusement si non autorisé | AD-7 (vérification avant `core/`) | Couvert, avec une nuance — voir Gap E |
| FR-2 | Déclenchement manuel par bouton, jamais automatique | AD-7 ; seed : `main.py` = "serveur webhook" seul point d'entrée, aucun composant scheduler/polling dans l'arborescence | Couvert (implicite via absence de tout composant de déclenchement automatique) |
| FR-3 | Règle vocal/texte stricte | AD-7 ("branchement dans cet adaptateur") ; AD-5 (STT/TTS indépendants) | Couvert |
| FR-4 | Flux matin (séquence fixe, citation veille, résumé confirmé) | Capability Map → `core/graphs/matin.py`, gouverné par AD-1/2/3 | Couvert au niveau structurel ; le détail comportemental (séquence, citation) reste logique de graphe interne, raisonnablement hors du grain d'une spine |
| FR-5 | Flux midi (≤4 échanges, jamais re-demander le connu) | Capability Map → `midi.py`, AD-1/2/3 | Couvert au niveau structurel, idem FR-4 |
| FR-6 | Flux soir (6 points, reformulation unique par point, résumé confirmé) | Capability Map → `soir.py`, AD-1/2/3 | Couvert au niveau structurel — mais la règle précise "reformulation une seule fois par point" (confirmée explicitement par Sébastien, note PRD) n'a aucune trace dans la spine, pas même en Deferred — voir Gap C |
| FR-7 | Fiche du jour structurée, 1/jour, champs fixes + "Autres" | Convention "fiche du jour `{YYYY-MM-DD}.md`" ; schéma exact → Deferred (= Open Question #5) | Couvert (structure) + déféré explicitement (schéma) |
| FR-8 | Lecture systématique de la fiche du jour, pas de mémoire héritée | AD-3 | Couvert — voir nuance Gap F (portée historique pour FR-11) |
| FR-9 | Mémoire intra-trajet | AD-2 (checkpointer scopé par thread_id) | Couvert |
| FR-10 | Déclenchement explicite uniquement (patterns) | Capability Map → `patterns.py`, AD-1/AD-3 | Couvert |
| FR-11 | Recherche ciblée dans l'historique, jamais tout injecter | Deferred ("Mécanique exacte de la recherche ciblée... Open Question #2") | Déféré explicitement — mais voir tension avec AD-3, Gap F |
| FR-12 | Proposition de champ, jamais application/réécriture automatique | Capability Map → AD-1/AD-3 ; mécanisme d'ajout → Deferred (Open Question #4) | Le *mécanisme* est déféré, mais l'*invariant* (jamais de réécriture rétroactive, jamais d'application automatique) n'est élevé à aucune règle architecturale — voir Gap B |
| FR-13 | Transcription inexploitable → redemander, jamais appel LLM à vide | Capability Map → `obsidian_fs.py`/`notifier.py`, AD-6 (mapping un peu large — AD-6 parle d'écriture atomique, pas de validation STT) | Couvert nominalement par le Capability Map ; le mécanisme de détection lui-même reste dans `core/graphs` (raisonnable) |
| FR-14 | Message hors-sujet/ambigu → clarification, jamais halluciner | idem FR-13, mêmes gouvernances | Couvert nominalement, détection = logique de graphe (raisonnable) |
| FR-15 | Échec technique → notification Telegram explicite, jamais silencieux | AD-6 (write atomique + notification avant fin de flux) | Bien couvert |
| FR-16 | Point d'appel LLM isolé | AD-4 | Bien couvert |

## 2. §5 Constraints and Guardrails

| Guardrail | Atterrissage | Verdict |
| --- | --- | --- |
| Self-hosting (prod sans cloud tiers pour données sensibles ; exception LLM API test) | AD-4, Stack (Ollama/LiteLLM sur machine GPU), Structural Seed (sous-graphe Cloud isolé, étiqueté "phase de test uniquement") | Bien couvert |
| Confidentialité (contenu stocké/traité uniquement sur l'infra Sébastien ; exception transport Telegram) | AD-7, Structural Seed (Telegram API étiquetée "risque résiduel accepté") | Couvert pour Telegram — mais le seed introduit **deux autres** points de transit réseau tiers (Cloudflare Tunnel, Tailscale) non mentionnés au PRD et non rattachés explicitement à ce raisonnement — voir Gap D |
| Pas d'automatisation silencieuse (FR-12) | Deferred (mécanisme d'ajout de champ) | Le mécanisme est déféré ; l'invariant lui-même (jamais de réécriture rétroactive sans validation humaine) n'a pas de règle structurelle dédiée — voir Gap B |
| Sécurité des accès (secrets jamais en clair/versionnés) | Consistency Conventions ("exclusivement par variables d'environnement, `.env` gitignored") + Deferred ("détails de credentials... laissé à l'implémentation, gouverné par la contrainte") | Bien couvert |
| Temps de réponse (qualitatif, à valider empiriquement) | Deferred ("Borne de temps de réponse... instrumentation dès le premier déploiement") + Consistency Conventions (log par étape = base d'instrumentation) | Bien couvert |

## 3. §6 Non-Goals

- Pas de rappels proactifs / pas de déclenchement auto → reflété en creux par AD-7 + absence de tout composant scheduler dans le Structural Seed. OK.
- Pas d'intégration Jira → absence de mention, cohérent avec un non-goal (rien à faire atterrir). OK.
- Pas d'ambition d'innovation algorithmique → non structurel, sans objet pour une spine. OK.
- **Pas de correction/jugement sur le contenu des bilans** ("le système capture, il ne critique pas") → **aucune trace dans la spine**, ni AD, ni convention, ni Deferred. Voir Gap C.

## 4. §8 Success Metrics (portion architecturale)

- SM-2 (continuité parfaite, 0 question reposée) → AD-2 + AD-3 la rendent structurellement atteignable. OK.
- SM-4 (gate : aucun contenu hors infra Sébastien en prod, exceptions test LLM API + transport Telegram) → AD-4 + Structural Seed. OK, sous réserve de Gap D (Cloudflare Tunnel/Tailscale non explicitement rattachés à cette gate).
- SM-1, SM-3, SM-C1 : qualitatifs / niveau usage, hors du grain d'une architecture spine — normal qu'ils n'aient pas d'atterrissage structurel dédié.

## 5. §9 Open Questions

| # | Question | Statut dans la spine |
| --- | --- | --- |
| 1 | Outil d'orchestration (n8n vs OpenClaw) — à trancher via spike en architecture | **Absent.** Aucune mention de n8n ni d'OpenClaw dans tout le document — voir Gap A |
| 2 | Mécanique de recherche ciblée (FR-11) | Deferred, explicite |
| 3 | Détails switch LLM (credentials multiples) | Résolu par AD-4 + Consistency Conventions (liste des env vars), donc pas besoin de rester "ouvert" |
| 4 | Mécanisme d'ajout de champ (FR-12) | Deferred, explicite |
| 5 | Schéma YAML exact de la fiche du jour | Deferred, explicite |

## 6. Attentes qualitatives / ton — non rattachées à un FR

- Note PRD FR-6 : "la règle 'reformuler une seule fois' s'applique par point du bilan (jusqu'à 6 reformulations possibles) — confirmé par Sébastien." → absente de la spine (voir Gap C).
- Non-Goal §6 : ton non-critique, le système "capture, ne juge pas" → absente de la spine (voir Gap C).
- SM-C1 (le taux de relance ne doit jamais augmenter, contrepoids à SM-1) → cohérent avec le point précédent, également absent — probablement acceptable de laisser au niveau prompt/story, mais aucune trace même en Deferred qui l'acterait comme "hors grain, à traiter en story".

---

## Gaps signalés

### Gap A — Décision d'orchestration (n8n vs OpenClaw) totalement absente
Le PRD (§0, §9 Open Question #1, §7.2) est explicite : ce choix est **hors scope du PRD**, à trancher "via spike en phase `bmad-architecture`". La spine ne mentionne ni n8n ni OpenClaw nulle part — ni dans le Stack, ni dans le Structural Seed, ni en Deferred. Elle présente directement une stack Python/LangGraph/webhook comme si la question n'avait jamais existé, sans trace du spike, de son résultat, ni de la justification du choix "code Python maison" plutôt que l'un des deux outils d'orchestration évalués. C'est la question ouverte la plus explicitement déléguée à l'architecture dans tout le PRD, et elle est la seule à n'avoir absolument aucun atterrissage — ni tranchée, ni explicitement redéférée. (Remarque : le nom du dépôt, `n8n-bilan-agent`, suggère qu'un choix n8n a peut-être été fait ailleurs sans être documenté ici, ce qui rend l'absence d'autant plus notable.)

### Gap B — L'invariant "pas d'automatisation silencieuse" n'est pas élevé en règle architecturale
Le PRD en fait un guardrail de premier rang (§5, item 3) et une conséquence testable de FR-12 ("Aucune fiche existante n'est modifiée rétroactivement par cette fonction"). La spine défère correctement le *mécanisme* concret d'ajout de champ (Open Question #4), mais ne pose aucune règle structurelle empêchant, par exemple, que `core/graphs/patterns.py` appelle directement `ObsidianStore.write()` sur une fiche passée. AD-6 codifie l'atomicité de l'écriture mais pas *qui* a le droit d'écrire *quoi* ni *quand*. Contraste avec AD-3, qui, elle, élève "la fiche du jour comme seul canal" en règle explicite — le pendant "jamais de réécriture rétroactive sans validation humaine" mériterait le même traitement (ou au moins une ligne Deferred qui le nomme explicitement, comme c'est fait pour le mécanisme).

### Gap C — Contraintes de ton/comportement conversationnel totalement absentes
Deux attentes qualitatives explicites du PRD n'ont aucune trace dans la spine, pas même en Deferred :
- Non-Goal §6 : "le système capture, il ne critique pas" (jamais de correction/jugement sur le contenu des bilans).
- Note FR-6 : règle de reformulation unique par point (jusqu'à 6 par flux soir), explicitement confirmée par Sébastien.

Il est défendable qu'une spine architecturale (frontières de modules, invariants structurels) ne soit pas l'endroit pour coder ces règles — elles relèvent du prompt/de la logique de nœud LangGraph. Mais la spine a une section Deferred faite pour exactement ce genre de cas ("ceci est intentionnellement hors du grain de ce document, à traiter en story/prompt"), et elle ne l'utilise pas ici. Résultat : rien ne distingue, à la lecture de la spine seule, un oubli d'une décision de scope délibérée.

### Gap D — Nouvelles dépendances réseau tierces (Cloudflare Tunnel, Tailscale) non rattachées au guardrail self-hosting/confidentialité
Le Structural Seed introduit Cloudflare Tunnel (ingress du webhook Telegram) et Tailscale (accès au service STT sur la machine GPU) — deux services tiers non mentionnés dans le PRD. Le PRD (§5, Confidentialité) accepte explicitement *un seul* risque résiduel de transport tiers : Telegram, avec une justification dédiée (chiffrement en transit, restriction chat_id, absence d'alternative self-hosted). Cloudflare Tunnel et Tailscale ne stockent/n'analysent pas le contenu des bilans (couche transport uniquement), donc l'argument tiendrait probablement — mais la spine ne le formule pas : aucune ligne ne les rattache explicitement à ce raisonnement ni ne les traite comme une exception documentée au même titre que Telegram. Un lecteur strict de SM-4 (gate self-hosting) pourrait légitimement se demander si ces deux services constituent une extension non déclarée du risque résiduel accepté.

### Gap E (mineur) — AD-7 ne restate pas explicitement le silence total pour un chat_id non autorisé
FR-1 exige "ignoré silencieusement (aucune réponse, aucun indice de l'existence du bot)". AD-7 dit seulement que la vérification a lieu "avant tout appel à `core/`" — ce qui empêche le traitement métier, mais n'interdit pas explicitement à `telegram_bot.py` d'envoyer malgré tout un message de rejet ou d'erreur à l'expéditeur non autorisé. La règle telle qu'écrite couvre l'accès à la logique, pas le silence de la réponse elle-même.

### Gap F (mineur, tension plutôt qu'absence) — AD-3 scope la lecture à `read_today()`, mais lie aussi le flux patterns qui a besoin d'historique multi-jours
AD-3 énonce : "un flux ne connaît le contenu produit par un flux précédent que via `ObsidianStore.read_today()`." Le Capability Map lie pourtant `patterns.py` (FR-10-12) à AD-3, alors que FR-11 exige explicitement un accès à plusieurs fiches passées ("recherche ciblée dans l'historique"). Le *comment* est correctement déféré (Open Question #2), mais la règle AD-3 telle que formulée ne prévoit pas d'exception pour ce cas — littéralement, elle l'exclut. Pas un oubli au sens propre (le besoin est reconnu ailleurs et déféré), mais une formulation d'AD qui entre en tension avec ce qu'elle est censée gouverner.

---

## Synthèse

La spine couvre correctement l'écrasante majorité du PRD : les 16 FR ont tous un point d'atterrissage identifiable, les guardrails self-hosting/confidentialité/sécurité des accès/temps de réponse sont bien traités (dont deux via des items Deferred explicites et bien nommés), et 4 des 5 Open Questions sont soit résolues soit explicitement déférées avec renvoi à la question du PRD.

Les manques réels sont concentrés sur :
1. **Une question ouverte structurante jamais traitée** (Gap A — n8n/OpenClaw), la plus significative des six.
2. **Un invariant de premier rang non élevé en règle** (Gap B — pas de réécriture rétroactive/automatisation silencieuse), alors que son pendant (AD-3) reçoit ce traitement.
3. **Des attentes de ton totalement silencieuses** (Gap C), ni traitées ni explicitement mises hors-grain.
4. Deux points mineurs de précision (Gap D nouvelles dépendances réseau non rattachées au guardrail confidentialité, Gap E silence non explicite, Gap F tension de formulation AD-3/patterns).
