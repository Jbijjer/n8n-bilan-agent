# Addendum — Assistant Trajet quotidien

Détail technique et décisions déjà validées par Sébastien, extraits du document source [`docs/brief-projet-assistant-trajet.md`](../../../../../docs/brief-projet-assistant-trajet.md). Ce contenu ne va pas dans `brief.md` (trop fin pour un brief produit) mais sert d'intrant direct au PRD et à l'architecture — **à ne pas rouvrir**, sauf indication contraire explicite de Sébastien.

## Choix d'orchestration — Résolu en phase architecture

**Statut : tranché.** Décision : **Python custom + LangGraph**, ni n8n ni OpenClaw (options A et B ci-dessous, écartées). Raison décisive : c'est la seule option où le séquencement strict exigé par le PRD (une question à la fois, jamais sauter d'étape, une seule reformulation par point — FR-4/5/6) est garanti par construction (machine à états native LangGraph — nodes/edges/state) plutôt que porté par la qualité du prompt, comme c'est le cas pour n8n *et* OpenClaw. Décision et détails complets dans le spine d'architecture : `_bmad-output/planning-artifacts/architecture/architecture-assistant-trajet-quotidien-2026-09-03/`.

Options écartées, conservées ici pour mémoire :

**Option A — n8n** (hypothèse de départ, cf. section Orchestration ci-dessous)
- Pour : nodes Telegram + AI Agent (mémoire de session intégrée) déjà bien adaptés au besoin ; self-hosted mature ; debug visuel de l'exécution (atout pour un utilisateur TDA) ; sous-workflows réutilisables qui collent au pattern "Call LLM isolé" (contrainte n°1).
- Contre : la séquence stricte "une question à la fois, ne jamais sauter d'étape, insister une seule fois" repose sur le prompt plutôt que sur une vraie machine à états — fragilité potentielle à travers plusieurs exécutions webhook. JSON de workflow peu diffable/versionnable en git.

**Option B — OpenClaw** ([openclaw/openclaw](https://github.com/openclaw/openclaw)) — assistant IA personnel self-hosted, multi-canaux (dont Telegram), découvert lors de cette remise en question.
- Pour : vocal Telegram natif qui correspond quasi mot pour mot à la règle du brief (transcription auto via Whisper local en priorité, réponses en bulles vocales Telegram, mode `tts.auto: "inbound"` — ne parle que si le message entrant était vocal) ; accès fichiers via MCP filesystem server (lecture/écriture Obsidian sans plugin tiers) ; modèles hébergés ou locaux interchangeables (contrainte n°1) ; skills/plugins MCP (TypeScript/Python) pour les flux matin/midi/soir.
- Contre : projet très récent (inconnu de l'agent avant recherche web, malgré ~388k étoiles GitHub — croissance très rapide, donc maturité et historique de sécurité à vérifier indépendamment avant d'y confier du contenu secteur public confidentiel, contrainte n°3) ; surface multi-canaux (WhatsApp/Telegram/Slack/Discord/Signal/iMessage) plus large qu'un workflow n8n isolé ; même limite que n8n sur la séquence stricte (portée par le prompt, pas une vraie FSM) ; boutons Telegram personnalisés (reply keyboard) à vérifier concrètement, pas confirmé en documentation.

*(Le chemin de décision par spike envisagé initialement n'a pas été nécessaire — la contrainte de séquencement strict du PRD départageait déjà les options sans ambiguïté.)*

Sources consultées : [openclaw/openclaw (GitHub)](https://github.com/openclaw/openclaw) · [OpenClaw docs — Plugin SDK](https://docs.openclaw.ai/plugins/sdk-overview) · [OpenClaw docs — Tools/Skills](https://docs.openclaw.ai/tools) · [Stack Junkie — OpenClaw Voice Mode](https://www.stack-junkie.com/blog/openclaw-voice-mode-telegram) · [LumaDock — Add voice to OpenClaw](https://lumadock.com/tutorials/openclaw-voice-tts-stt-talk-mode)

## Contraintes non négociables

1. **Indépendance de modèle LLM** — appel LLM isolé/interchangeable, aucune logique de prompt ne dépend d'un modèle particulier.
2. **Full self-hosted** — orchestration, stockage (Obsidian), et à terme LLM + vocal (STT/TTS), sans dépendance cloud pour les données sensibles.
3. **Confidentialité secteur public** — contenu des bilans jamais hors infrastructure de Sébastien en production. Exception assumée : phase de tests initiale avec LLM API (Claude/GPT-4o).
4. **Pas d'automatisation en arrière-plan** pour l'analyse de patterns — uniquement à la demande explicite, en langage naturel. Aucun job planifié modifiant la structure de données sans validation humaine.
5. **Accès réseau** via Tailscale (point-à-point) + Cloudflare Tunnel (webhook exposé sans port ouvert en clair) — déjà tranché, pas d'alternative cloud à proposer.
6. **Déclenchement manuel** des flux via boutons Telegram — pas de géolocalisation ni d'horaire automatique en v1.

## Décisions établies, par domaine

### Interface
- Telegram comme canal unique, clavier personnalisé (reply keyboard) à 3 boutons : "Bilan matin", "Bilan midi", "Bilan soir".
- Vocal natif Telegram : message vocal → réponse vocale ; message texte → réponse texte (règle stricte selon le type de message entrant).
- Bot restreint au `chat_id` de Sébastien uniquement.

### Stockage — fiche du jour
- Un fichier markdown par jour dans un vault Obsidian (ex. `2026-08-14.md`).
- Frontmatter YAML fixe : date, plan matin, ajustement midi, prévu vs fait, imprévus, suivis pour demain (+ avec qui), feeling général, temps de procrastination estimé.
- Champ libre "Autres" en corps de texte, non structuré.
- Accès depuis n8n : système de fichiers direct, ou plugin Local REST API d'Obsidian (recherche full-text intégrée).

### Mémoire et continuité
- **Intra-journée** (matin → midi → soir) : pas de mémoire de session partagée entre trajets. Chaque flux lit la fiche du jour existante et l'injecte dans le prompt au démarrage.
- **Intra-trajet** : mémoire de session du node AI Agent n8n, clé = `chat_id` + date.
- **Long terme / patterns** : recherche ciblée dans l'historique Obsidian (pas d'injection massive), déclenchée uniquement à la demande explicite. Si un pattern net est détecté dans "Autres", l'agent propose un champ structuré dédié — sans réécrire les entrées passées.

### Orchestration
*(Description historique, pensée pour n8n — remplacée par la décision Python+LangGraph ci-dessus. Voir le spine d'architecture pour la conception actuelle : services/modules, checkpointer, frontières.)*
- ~~Workflow "routeur" (réception Telegram, transcription si vocal, routage) + sous-workflows par flux (matin / midi / soir / analyse patterns).~~
- ~~Sous-workflows communs réutilisables : lecture fiche, appel LLM, TTS, écriture Obsidian.~~
- ~~Appel LLM isolé dans un sous-workflow dédié ("Call LLM"), point d'entrée unique — swap de modèle = changer un seul node.~~
- Principe conservé sous une autre forme : appel LLM toujours isolé derrière un point d'entrée unique (interface `LLMClient` dans l'architecture Python) — swap de modèle = changer un adaptateur, pas la logique des flux.

### Vocal
- STT : `faster-whisper` self-hosted (medium/large-v3, français). **Mis à jour en architecture :** utilisé **in-process** (librairie Python native), pas de service HTTP séparé — simplification permise par le paradigme Python, moins de pièces à faire tourner et surveiller.
- TTS : Piper pour démarrer (bindings Python, in-process également) ; upgrade path vers Coqui XTTS v2 si besoin (~4-6GB VRAM). Interchangeables sans impact sur le reste du système.

### LLM
- Phase de tests : API (Claude ou GPT-4o).
- Phase de production : local via Ollama, Qwen2.5 14B ou Llama 3.1 8B (quantifiés), GPU 16GB VRAM (RTX 5060 Ti).
- Point de vigilance à valider empiriquement : finesse du LLM local à détecter un oubli vs une vraie absence de réponse, et à insister une seule fois sans devenir lourd.

### Prompts validés (réutilisables tels quels ou comme base)

**Système commun :**
```
Tu es l'assistant de bilan quotidien de Sébastien, lié à ses trajets en voiture.
Réponds de façon concise, une question à la fois, jamais de liste de questions.
Aucun jugement sur les tâches non faites, le temps de procrastination ou les imprévus.
Adapte tes phrases à une écoute/réponse vocale : court, direct, pas de formulations complexes.

Contexte du jour (extrait du fichier Obsidian) :
{{contenu_fiche_du_jour}}
```

**Flux matin :**
```
C'est le trajet du matin. Objectif : bilan bref d'hier + tâches restantes + plan du jour.
Étapes, une à la fois :
1. Rappelle en une phrase le bilan d'hier si présent dans le contexte.
2. Demande quelles tâches d'hier sont encore actives.
3. Demande si un scrum d'équipe est prévu aujourd'hui et si ça influence le plan.
4. Propose 2-3 priorités pour la journée, fais-les valider.
5. Termine par un résumé de 2-3 lignes à confirmer.
Ne saute pas d'étape. Si Sébastien répond déjà à une étape à venir, ne la repose pas.
```

**Flux midi (max 4 échanges) :**
```
C'est le trajet du midi (~20 minutes). Objectif : mini-ajustement, pas un nouveau bilan.
1. Rappelle en une phrase le plan établi ce matin (depuis le contexte).
2. Demande où ça en est / ce qui a changé.
3. Ajuste les priorités de l'après-midi si besoin.
4. Résumé d'une ligne à confirmer.
Maximum 4 échanges. Ne pas étirer la conversation.
```

**Flux soir :**
```
C'est le trajet du soir. Objectif : bilan complet. Le contexte contient le plan du matin et
l'ajustement du midi — base tes questions dessus, ne redemande pas ce qui est déjà connu.
Couvre ces points dans l'ordre, un à la fois :
1. Pour chaque tâche prévue connue, demande le statut (fait / en cours / pas fait).
2. Imprévus survenus.
3. Ce qui reste à terminer.
4. Suivis à faire demain, et avec qui.
5. Feeling général de la journée.
6. Estimation du temps de procrastination.

Si une réponse est très courte sur un point qui a normalement du contenu (ex. juste "non" sans
détail), reformule la question une seule fois différemment. N'insiste pas une deuxième fois —
accepte la réponse.

Tout ce qui ne rentre pas dans les points 1-6 va dans un champ "Autres" séparé, sans le structurer.

Termine par un résumé de 3-4 lignes à confirmer.
```

**Flux analyse de patterns (à la demande) :**
```
Sébastien te demande d'analyser des patterns dans ses bilans passés. Le contexte injecté contient
les entrées récentes pertinentes (champ "Autres" et/ou champs structurés selon la recherche
effectuée).
Repère les thèmes ou éléments qui reviennent souvent. Si un thème récurrent apparaît clairement
(pas juste 1-2 mentions), propose de créer un champ structuré dédié pour ce thème. Sinon, dis
simplement qu'aucun pattern net ne ressort encore.
Reste factuel — ne pas extrapoler au-delà de ce qui est présent dans les données.
```

### Gestion des erreurs
- Transcription vide/incohérente → redemander par vocal plutôt que d'envoyer au LLM.
- Message hors-sujet/ambigu → le LLM doit demander clarification, ne pas halluciner.
- Échec technique (node, fichier inaccessible) → `Error Trigger` global n8n, notification Telegram explicite ("Erreur technique, bilan non enregistré").

### Sécurité
- Credentials via le système de credentials chiffrés de n8n, jamais en clair, jamais commit en git.
- Restriction d'accès au bot par `chat_id`.

### Séquence de tests prévue
1. Chaque brique isolément (Telegram, Whisper, lecture/écriture Obsidian).
2. Un flux complet en texte.
3. Le même flux en vocal.
4. Journée simulée complète (3 trajets) pour valider la continuité via la fiche du jour.
5. Usage réel sur journées de vacances avant d'exposer le système à de vraies données de travail.

## Ouvert à l'approfondissement (intrant pour PRD / architecture)

- Schéma de données précis de la fiche Obsidian (noms de champs exacts, types, format du frontmatter YAML, convention de nommage des fichiers).
- ~~Liste des nodes n8n requis par sous-workflow~~ — obsolète (architecture Python+LangGraph retenue).
- Mécanique exacte de la recherche ciblée dans l'historique Obsidian pour l'analyse de patterns (grep par plage de dates vs requêtes au plugin REST API — critères de sélection des entrées pertinentes).
- Gestion précise du checkpointer LangGraph (rétention, nettoyage des anciens threads, format) — voir spine d'architecture.
- Détails d'implémentation du switch LLM (variables d'environnement, structure des credentials multiples).
- Spécification du mécanisme de proposition de nouveau champ (comment le champ structuré est réellement ajouté au schéma une fois validé par Sébastien — modification manuelle ou semi-automatisée).
- Stories/tickets de développement découpés à partir de la séquence de tests déjà établie.
