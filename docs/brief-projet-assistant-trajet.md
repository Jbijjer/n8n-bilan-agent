# Brief de projet — Assistant Trajet quotidien

## Pour l'agent (BMAD)

Ce document est le résultat d'un brainstorm d'architecture complet, déjà validé par le porteur du projet (Sébastien). Il contient des **décisions prises**, pas des options à explorer. Le rôle attendu ici est de pousser vers une spec technique complète (PRD, architecture détaillée, stories) **à partir de** ces décisions — pas de revalider les choix déjà faits, sauf indication contraire explicite du porteur de projet.

Une section "Contraintes non négociables" est fournie plus bas — à respecter strictement. Une section "Ouvert à l'approfondissement" liste ce qui reste à spécifier plus finement (schéma de données, liste de nodes, etc.) — c'est là que la valeur ajoutée de l'agent est attendue.

---

## Contexte

Sébastien est programmeur/analyste dans le secteur public (petite équipe : 4 développeurs, 1 product owner, 1 responsable conception/normes), TDA diagnostiqué. Il veut un assistant conversationnel qui l'accompagne à trois moments de sa journée, arrimés à ses trajets en voiture (matin, midi ~20 min, soir), pour structurer sa planification et ses bilans quotidiens.

## Objectif du système

Un agent conversationnel, accessible par vocal ou texte via Telegram, qui :
- Aide à planifier la journée le matin (à partir du bilan de la veille)
- Permet un micro-ajustement à midi
- Fait un bilan complet le soir, en posant des questions ciblées plutôt que génériques
- Garde une mémoire structurée et consultable des journées passées
- Peut, à la demande, repérer des patterns récurrents et proposer d'évoluer sa propre structure de données

## Contraintes non négociables

1. **Indépendance de modèle LLM** — l'appel au LLM doit être isolé/interchangeable, aucune logique de prompt ou de structure ne doit dépendre d'un modèle en particulier.
2. **Full self-hosted** — orchestration (n8n), stockage (Obsidian), et à terme le LLM et le vocal (STT/TTS) tournent localement, sans dépendance cloud pour les données sensibles.
3. **Confidentialité secteur public** — le contenu des bilans (potentiellement des détails de travail) ne doit pas sortir de l'infrastructure de Sébastien une fois en production. Exception assumée : phase de tests initiale, où un LLM API (Claude/GPT-4o) est utilisé délibérément pour juger la qualité du système.
4. **Pas d'automatisation en arrière-plan** pour l'analyse de patterns — uniquement à la demande explicite, en langage naturel. Aucun job planifié qui modifie la structure de données sans validation humaine.
5. **Accès réseau** via Tailscale (accès point-à-point) + Cloudflare Tunnel (webhook exposé sans port ouvert en clair) — déjà tranché, ne pas proposer d'alternative cloud pour l'exposition du webhook.
6. **Déclenchement manuel** des flux via boutons Telegram — pas de géolocalisation ni d'horaire automatique pour la version initiale (identifié comme évolution future possible, pas un requis actuel).

## Décisions établies, par domaine

### Interface
- Telegram comme canal unique, avec clavier personnalisé (reply keyboard) à 3 boutons : "Bilan matin", "Bilan midi", "Bilan soir".
- Support du vocal natif Telegram : message vocal → réponse vocale ; message texte → réponse texte (règle stricte selon le type de message entrant).
- Restriction du bot au `chat_id` de Sébastien uniquement (sécurité).

### Stockage — fiche du jour
- Un fichier markdown par jour dans un vault Obsidian (ex. `2026-08-14.md`).
- Champs fixes en frontmatter YAML : date, plan matin, ajustement midi, prévu vs fait, imprévus, suivis pour demain (+ avec qui), feeling général, temps de procrastination estimé.
- Un champ libre "Autres" en corps de texte, non structuré.
- Accès depuis n8n : système de fichiers direct, ou plugin Local REST API d'Obsidian (recherche full-text intégrée).

### Mémoire et continuité
- **Intra-journée** (matin → midi → soir) : PAS de mémoire de session partagée entre trajets. Chaque flux lit la fiche du jour existante et injecte son contenu dans le prompt avant de démarrer.
- **Intra-trajet** (plusieurs échanges dans un même flux) : mémoire de session du node AI Agent n8n, clé = `chat_id` + date.
- **Long terme / patterns** : recherche ciblée dans l'historique Obsidian (pas d'injection massive de tout l'historique dans le prompt), déclenchée uniquement à la demande explicite de Sébastien. Si un pattern net est détecté dans le champ "Autres", l'agent propose de créer un champ structuré dédié — sans réécrire les entrées passées.

### Orchestration
- Un workflow "routeur" (réception Telegram, transcription si vocal, routage) + sous-workflows séparés par flux (matin / midi / soir / analyse patterns).
- Sous-workflows communs réutilisables : lecture fiche, appel LLM, TTS, écriture Obsidian.
- Appel LLM isolé dans un sous-workflow dédié ("Call LLM"), point d'entrée unique pour tous les flux — swap de modèle = changer un seul node.

### Vocal
- STT : `faster-whisper` self-hosted (modèle medium/large-v3, français), exposé en service local appelé via HTTP.
- TTS : Piper pour démarrer (léger, rapide) ; upgrade path vers Coqui XTTS v2 si la qualité vocale devient un irritant (~4-6GB VRAM). Les deux doivent rester interchangeables sans impact sur le reste du workflow.

### LLM
- Phase de tests : API (Claude ou GPT-4o).
- Phase de production : local via Ollama, Qwen2.5 14B ou Llama 3.1 8B (quantifiés), compatible avec un GPU 16GB VRAM (RTX 5060 Ti).
- Point de vigilance à valider empiriquement : la finesse du LLM local pour détecter un oubli vs une vraie absence de réponse, et savoir insister une seule fois sans devenir lourd.

### Prompts (validés, réutilisables tels quels ou comme base)

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

## Ouvert à l'approfondissement (rôle attendu de l'agent)

Ces points ont été identifiés mais pas détaillés — c'est ici que l'agent BMAD peut apporter le plus de valeur :

- **Schéma de données précis** de la fiche Obsidian (noms de champs exacts, types, format du frontmatter YAML, convention de nommage des fichiers).
- **Liste des nodes n8n** requis par sous-workflow, avec leurs configurations (credentials, paramètres).
- **Mécanique exacte de la recherche ciblée** dans l'historique Obsidian pour l'analyse de patterns (grep par plage de dates vs requêtes au plugin REST API — critères de sélection des entrées pertinentes).
- **Gestion précise de la mémoire de session** du node AI Agent (durée de vie, nettoyage, format).
- **Détails d'implémentation du switch LLM** (variables d'environnement, structure des credentials multiples).
- **Spécification du mécanisme de proposition de nouveau champ** (comment le champ structuré est réellement ajouté au schéma une fois validé par Sébastien — modification manuelle ou semi-automatisée).
- **Stories/tickets de développement** découpés à partir de la séquence de tests déjà établie (section précédente).

## Évolutions futures (hors scope initial, à ne pas développer maintenant)

- Rappels proactifs spontanés (sans demande explicite).
- Intégration avec Jira (outil de scrum de l'équipe).
- Déclenchement automatique des flux (géolocalisation ou horaire).
