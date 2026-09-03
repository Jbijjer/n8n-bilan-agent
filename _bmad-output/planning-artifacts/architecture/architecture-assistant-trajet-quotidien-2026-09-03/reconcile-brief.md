# Réconciliation — brief/addendum vs ARCHITECTURE-SPINE.md

Date : 2026-09-03
Sources lues intégralement :
- `_bmad-output/planning-artifacts/briefs/brief-assistant-trajet-quotidien-2026-09-02/brief.md`
- `_bmad-output/planning-artifacts/briefs/brief-assistant-trajet-quotidien-2026-09-02/addendum.md`
- `_bmad-output/planning-artifacts/architecture/architecture-assistant-trajet-quotidien-2026-09-03/ARCHITECTURE-SPINE.md`

Objectif : vérifier que les contraintes non négociables et décisions établies du brief/addendum (interface, stockage/fiche du jour, mémoire, vocal, LLM, gestion d'erreurs, sécurité) sont bien reflétées dans le spine, en tenant compte du fait que l'addendum a lui-même été édité en cours de route pour abandonner n8n au profit de Python+LangGraph+LiteLLM+topologie deux hôtes.

## Verdict global

La majorité des décisions techniques (topologie deux hôtes, LiteLLM comme point de swap LLM unique, STT réseau sur la machine GPU, TTS Piper in-process, Tailscale/Cloudflare Tunnel, restriction chat_id, écriture atomique + notification d'erreur) sont fidèlement portées dans le spine, avec des AD explicites et une correspondance claire au niveau des FR. Cependant, il y a **un gap substantiel** et **deux gaps mineurs/silencieux**, plus **une incohérence documentaire résiduelle dans l'addendum lui-même** (non corrigée lors du pivot n8n → Python).

---

## 1. Gap substantiel — la garantie de séquencement strict n'est pas codifiée comme AD

**Ce que dit l'addendum (raison décisive du choix d'architecture) :**
> « le séquencement strict exigé par le PRD (une question à la fois, jamais sauter d'étape, une seule reformulation par point — FR-4/5/6) est garanti par construction (machine à états native LangGraph — nodes/edges/state) plutôt que porté par la qualité du prompt »

C'est littéralement la raison pour laquelle n8n et OpenClaw ont été écartés au profit de Python+LangGraph. C'est donc la décision d'architecture la plus centrale du document source.

**Ce que fait le spine :**
- Le Design Paradigm dit seulement : « chaque étape du bilan est un node, chaque transition une arête, l'état est explicite et persisté (voir AD-2) » — une description structurelle, pas une règle.
- La Capability → Architecture Map lie FR-4/5/6 (flux matin/midi/soir) à AD-1 (direction de dépendance), AD-2 (portée du checkpointer) et AD-3 (fiche du jour = canal de continuité). **Aucune de ces trois AD n'énonce la règle de séquencement elle-même** (une étape à la fois, ne jamais sauter, une seule reformulation puis acceptation de la réponse).
- Rien dans `core/state.py` ni dans les graphes (`matin.py`, `midi.py`, `soir.py`) n'est décrit comme portant un compteur de relance ou une garde anti-saut d'étape — ce qui est justement le mécanisme concret qui rendrait la garantie « par construction » vraie plutôt qu'aspirationnelle (une topologie de graphe n'empêche pas à elle seule qu'un node relance deux fois ou saute une étape ; il faut un champ d'état + une arête conditionnelle explicite pour ça).

**Pourquoi c'est un problème :** le spine sert de socle qui doit garder tout le développement futur cohérent. Sans une AD dédiée (ex. « AD-8 — chaque étape d'un flux est un node distinct ; aucune arête ne saute un node ; l'état porte un compteur de relance par point, plafonné à 1 »), rien n'empêche une implémentation future de recréer exactement le problème que le pivot vers LangGraph était censé éliminer (séquence portée par la qualité du prompt plutôt que garantie par la structure).

**Recommandation :** ajouter une AD explicite liée à FR-4/5/6 qui formalise : (a) un node par étape, pas de branchement libre qui saute un node ; (b) un champ d'état type `retry_count` par point du bilan soir, plafonné à 1 avant acceptation de la réponse telle quelle.

---

## 2. Gap silencieux — la contrainte non négociable #4 (pas d'automatisation pour l'analyse de patterns) n'a pas d'AD dédiée

**Ce que dit l'addendum (contrainte non négociable n°4, et repris dans le brief comme différenciateur central) :**
> « Pas d'automatisation en arrière-plan pour l'analyse de patterns — uniquement à la demande explicite, en langage naturel. Aucun job planifié modifiant la structure de données sans validation humaine. »

Le brief la reprend comme trait distinctif : « aucune automatisation silencieuse qui modifierait ses données » (section *What Makes This Different*), et la vision insiste : « sans jamais agir seul sur les données passées ».

**Ce que fait le spine :**
- La Capability Map lie FR-10/11/12 (analyse de patterns) à AD-1 et AD-3 uniquement — ni l'une ni l'autre n'énonce « déclenchement à la demande explicite uniquement » ni « aucune modification de schéma sans validation humaine ».
- Le Deferred mentionne bien « Mécanisme précis d'ajout d'un nouveau champ structuré (FR-12) — Open Question #4 du PRD » — mais cela ne diffère que le *comment* (modalités d'implémentation du mécanisme de proposition), pas *l'invariant* que la proposition doit rester une proposition (jamais une application automatique) et que rien ne doit tourner en tâche de fond ou planifiée pour l'analyse de patterns.
- Rien dans la stack ou la structure de dossiers ne mentionne explicitement l'absence de scheduler/cron pour ce flux (ce qui est cohérent avec la stack proposée, qui n'introduit aucun ordonnanceur — mais ce n'est nulle part affirmé comme règle protégée).

**Recommandation :** ajouter une AD (ou étendre AD-3) qui rend explicite : `core/graphs/patterns.py` n'est invoqué que par une requête utilisateur explicite (jamais par un trigger planifié) ; toute proposition de nouveau champ structuré est un message à valider par Sébastien, jamais une écriture directe dans le schéma ou les entrées passées.

---

## 3. Gap mineur — déclenchement manuel uniquement (contrainte non négociable #6 / hors-scope) non repris comme AD

Le brief liste explicitement en hors-scope : « Déclenchement automatique des flux (géolocalisation ou horaire) », et la contrainte #6 de l'addendum : « Déclenchement manuel des flux via boutons Telegram — pas de géolocalisation ni d'horaire automatique en v1 ».

Le spine ne contredit pas cela (la stack ne comporte aucun mécanisme de géolocalisation ou d'ordonnanceur), mais AD-7 — qui gouverne la frontière transport — ne porte que sur le contrôle d'accès `chat_id` et la règle vocal-in/vocal-out, texte-in/texte-out. Le déclenchement exclusivement manuel par les 3 boutons du reply keyboard Telegram (Interface, section brief/addendum) n'apparaît nulle part comme règle protégée — c'est une absence structurelle de mécanisme plutôt qu'une règle explicite. Risque faible (rien dans le spine n'invite à ajouter un trigger automatique), mais à nommer si on veut que le spine reste la source de vérité auto-suffisante.

**Recommandation (optionnelle, faible priorité) :** mentionner explicitement dans AD-7 ou la table de conventions que les 3 flux ne sont invocables que via une action utilisateur explicite (bouton reply keyboard), jamais par un event externe (webhook cron, géoloc).

---

## 4. Gap mineur — règle d'erreur « transcription vide/incohérente » non reprise

L'addendum liste 3 règles sous *Gestion des erreurs* :
1. Transcription vide/incohérente → redemander par vocal plutôt que d'envoyer au LLM. **Non repris dans le spine.**
2. Message hors-sujet/ambigu → le LLM doit demander clarification, ne pas halluciner. (Règle de prompt, pas d'architecture — omission acceptable.)
3. Échec technique → notification Telegram explicite. **Repris fidèlement dans AD-6.**

La règle #1 touche la frontière `adapters/telegram_bot.py` ↔ `ports.STTClient` (que faire quand `STTClient` renvoie une transcription vide ou incohérente avant même d'atteindre `core/`), donc elle est de niveau architecture, pas seulement prompt. Elle n'apparaît ni dans AD-5 (STT/TTS) ni AD-7 (frontière transport) ni dans Deferred.

**Recommandation (faible priorité) :** ajouter une clause à AD-5 ou AD-7 : une transcription vide/incohérente renvoyée par `STTClient` déclenche une nouvelle demande vocale à l'utilisateur avant tout appel à `core/`, sans jamais transmettre un texte vide/incohérent au LLM.

---

## 5. Incohérence documentaire résiduelle dans l'addendum (pas dans le spine) — références n8n non nettoyées après le pivot

Lors du pivot n8n → Python+LangGraph, seule la section **Orchestration** de l'addendum a été explicitement corrigée (texte barré + note de remplacement). Deux autres sections gardent une formulation n8n-spécifique qui n'a pas été mise à jour, alors que le spine, lui, a bien pivoté sur le fond :

- **Stockage — fiche du jour** : « Accès depuis n8n : système de fichiers direct, ou plugin Local REST API d'Obsidian (recherche full-text intégrée). » Le spine retient l'accès filesystem direct (`adapters/obsidian_fs.py`), ce qui correspond à une des deux options listées — donc pas de contradiction de fond, mais la phrase elle-même référence encore n8n littéralement.
- **Sécurité** : « Credentials via le système de credentials chiffrés de n8n, jamais en clair, jamais commit en git. » Le spine remplace cela par des variables d'environnement (`.env`, gitignored) — cohérent en esprit (jamais en clair, jamais commit), mais le mécanisme concret cité dans l'addendum (credentials n8n) n'existe plus dans l'architecture retenue, et l'addendum ne le signale pas comme obsolète (contrairement à la section Orchestration, qui a reçu ce traitement).

**Ce n'est pas une divergence de fond** entre le spine et l'intention du brief (le principe « jamais en clair, jamais commit » est préservé), mais c'est une incohérence *documentaire* : l'addendum affirme explicitement en en-tête « à ne pas rouvrir, sauf indication contraire explicite de Sébastien », alors que deux de ses sections sont désormais factuellement obsolètes suite au pivot acté ailleurs dans le même document. Un lecteur futur qui prend l'addendum au pied de la lettre (sans croiser le spine) recevrait une information technique fausse sur le mécanisme de credentials et d'accès au vault.

**Recommandation :** une passe de nettoyage légère sur l'addendum (même traitement par strikethrough que la section Orchestration) pour les sections Stockage et Sécurité, avec renvoi vers les conventions du spine (variables d'environnement, `adapters/obsidian_fs.py`).

---

## Ce qui est correctement reflété (contrôlé, pas de gap)

- **Interface** : canal Telegram unique, restriction `chat_id`, règle vocal-in→vocal-out / texte-in→texte-out → AD-7, fidèle.
- **Mémoire intra-journée** (pas de mémoire partagée entre matin/midi/soir, lecture de la fiche du jour au démarrage de chaque flux) → AD-3, fidèle.
- **Mémoire intra-trajet** (checkpointer par flux) → AD-2 ; la clé passe de `chat_id+date` (n8n) à `chat_id:date:flow_type` (LangGraph), changement nécessaire et cohérent avec AD-3, explicitement laissé à l'architecture par l'addendum (« voir spine d'architecture »).
- **Indépendance LLM** (contrainte non négociable #1) → AD-4, fidèle ; LiteLLM comme point de swap unique bien capturé.
- **Full self-hosted + confidentialité secteur public** (contraintes #2/#3) → topologie deux hôtes, diagramme Structural Seed qui isole explicitement le risque résiduel Telegram et la phase de test API LLM comme « hors infra Sébastien » — le spine va même au-delà du texte source en nommant ce risque explicitement.
- **Accès réseau** (contrainte #5, Tailscale + Cloudflare Tunnel) → diagramme Structural Seed, fidèle, présenté comme déjà tranché tel que l'addendum l'exige.
- **Vocal** (STT réseau sur machine GPU « corrigé en architecture », TTS Piper in-process CPU, upgrade path Coqui XTTS v2) → AD-5 + Deferred, fidèle, y compris la correction apportée en cours d'architecture.
- **LLM** (phase test API / phase prod Ollama local, modèles cités) → AD-4 + Stack, fidèle.
- **Écriture atomique + notification d'erreur technique** → AD-6, fidèle.
- **Schéma exact de la fiche du jour, mécanique de recherche ciblée pour patterns, détails credentials, rétention checkpointer** → correctement laissés en Deferred avec renvoi aux Open Questions du PRD, comme l'addendum le demande.

---

## Résumé des gaps (par priorité)

| # | Gap | Sévérité | Domaine |
| --- | --- | --- | --- |
| 1 | La garantie de séquencement strict (une étape à la fois, jamais sauter, une seule relance) — raison décisive du choix LangGraph — n'est codifiée par aucune AD ; les FR-4/5/6 ne sont liées qu'à AD-1/AD-2/AD-3, qui ne portent pas cette règle | Élevée | Orchestration / core graphs |
| 2 | Contrainte non négociable #4 (analyse de patterns à la demande explicite uniquement, jamais planifiée ; propositions de champs jamais auto-appliquées) sans AD dédiée | Moyenne | Analyse de patterns |
| 3 | Déclenchement strictement manuel des flux (pas de géoloc/horaire) non nommé comme règle protégée dans une AD | Faible | Interface / transport |
| 4 | Règle d'erreur « transcription vide/incohérente → redemander par vocal » absente du spine | Faible | Vocal / gestion d'erreurs |
| 5 | Addendum non nettoyé : sections Stockage et Sécurité gardent des références n8n obsolètes après le pivot (incohérence documentaire, pas de fond) | Faible (documentaire) | Stockage, Sécurité |
