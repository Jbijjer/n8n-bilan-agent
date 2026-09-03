---
title: "Assistant Trajet quotidien"
status: final
created: 2026-09-02
updated: 2026-09-03
---

# PRD : Assistant Trajet quotidien

## 0. Document Purpose

Ce PRD s'adresse à Sébastien (porteur de projet et seul utilisateur) et sert d'intrant direct à la phase architecture (`bmad-architecture`). Il définit *ce que* le système doit faire, pas *comment* — les choix techniques déjà tranchés (STT/TTS, LLM, stockage) et le choix d'orchestration encore ouvert (n8n vs OpenClaw) vivent dans l'addendum du brief produit ([`../../briefs/brief-assistant-trajet-quotidien-2026-09-02/addendum.md`](../../briefs/brief-assistant-trajet-quotidien-2026-09-02/addendum.md)), référencé mais non dupliqué ici. Vocabulaire ancré au Glossaire (§3) ; fonctionnalités groupées avec exigences fonctionnelles (FR) numérotées globalement ; hypothèses inférées marquées `[ASSUMPTION]` et indexées en §10.

## 1. Vision

Un assistant conversationnel personnel, self-hosted, qui rencontre Sébastien au moment déjà disponible de sa journée — le trajet en voiture — plutôt que d'exiger un moment dédié à la planification ou au bilan. Trois rendez-vous quotidiens (matin, midi, soir), déclenchés manuellement depuis Telegram, en vocal ou en texte : le matin pose le plan, le midi l'ajuste en quelques échanges, le soir referme la journée avec des questions ciblées sur ce qui était réellement prévu — jamais une liste générique.

Chaque journée devient une fiche structurée et consultable dans un vault Obsidian. Avec le temps, cette continuité permet à Sébastien, à sa demande, de repérer ce qui revient d'un jour à l'autre — et de faire évoluer la structure même de ses bilans en conséquence, toujours sous son contrôle explicite.

## 2. Target User

### 2.1 Jobs To Be Done

- Arriver à mon trajet du midi ou du soir sans avoir à me souvenir moi-même de ce qui était prévu le matin.
- Terminer une journée avec un bilan honnête, sans l'effort de rédaction qu'un TDA rend coûteux.
- Me relire une semaine plus tard et retrouver un historique fiable plutôt qu'un vide ou des entrées bâclées.
- Sur demande, comprendre ce qui revient souvent dans mes journées (frein, distraction, type d'imprévu) sans avoir à relire moi-même des semaines de notes.

*Utilisateur unique : Sébastien lui-même. Pas de section Non-Users — l'usage prévu comme le contexte (secteur public, TDA) sont déjà cadrés au brief.*

### 2.2 Key User Journeys

*Persona unique (Sébastien), usage personnel — forme allégée : une phrase par parcours, cf. §4 pour le détail comportemental.*

- **UJ-1.** Sébastien, au démarrage du trajet du matin, ouvre Telegram, appuie sur "Bilan matin" et confirme en 2-3 échanges les priorités du jour à partir du bilan de la veille.
- **UJ-2.** Sébastien, pendant le trajet du midi (~20 min), appuie sur "Bilan midi" et ajuste en quelques phrases les priorités de l'après-midi si la matinée a dévié du plan.
- **UJ-3.** Sébastien, au trajet du soir, appuie sur "Bilan soir" et répond question par question sur ce qui a été fait, les imprévus et ses suivis pour demain, sans avoir à reformuler ce que l'agent connaît déjà.
- **UJ-4.** Sébastien, un soir où il en ressent le besoin, demande explicitement à l'agent d'analyser ses derniers bilans pour un pattern récurrent, et reçoit soit une observation factuelle, soit la proposition d'un nouveau champ structuré.

## 3. Glossaire

- **Bilan** — Terme générique désignant l'activité de planification ou de compte-rendu propre à un flux (ex. "bilan matin", "bilan soir"). Un bilan se déroule *pendant* un flux et son contenu confirmé est ce qui est écrit dans la fiche du jour — le bilan est l'activité, la fiche du jour en est la trace persistante.
- **Fiche du jour** — Fichier markdown unique par journée dans le vault Obsidian (ex. `2026-08-14.md`), frontmatter YAML structuré + champ libre "Autres". Une fiche par date, jamais réécrite rétroactivement par l'agent sans validation humaine.
- **Flux** — Un des trois rendez-vous quotidiens (Flux Matin, Flux Midi, Flux Soir), déclenché manuellement.
- **Trajet** — Fenêtre de temps réelle (voiture) pendant laquelle un flux se déroule ; contrainte de forme (échanges courts, vocal adapté) mais pas d'objet du système.
- **Mémoire intra-trajet** — Continuité conversationnelle entre les échanges d'un même flux, le même jour.
- **Continuité intra-journée** — Passage d'information entre flux (matin → midi → soir) via lecture de la fiche du jour, jamais via mémoire de session partagée.
- **Champ "Autres"** — Zone libre non structurée de la fiche du jour, pour tout ce qui ne rentre pas dans les champs fixes.
- **Pattern** — Thème ou élément qui revient de façon répétée (pas 1-2 mentions isolées) dans le champ "Autres" et/ou les champs structurés sur plusieurs fiches du jour.
- **chat_id** — Identifiant Telegram unique de Sébastien ; seule identité autorisée à interagir avec le bot.

## 4. Features

### 4.1 Interface conversationnelle et accès

**Description :** Point d'entrée unique du système : Telegram, restreint à Sébastien, avec un déclenchement toujours manuel et une règle stricte de canal (vocal reste vocal, texte reste texte).

**Functional Requirements :**

#### FR-1 : Restriction d'accès

Le bot ne répond qu'aux messages provenant du `chat_id` de Sébastien.

**Consequences (testable) :**
- Tout message d'un `chat_id` différent est ignoré silencieusement (aucune réponse, aucun indice de l'existence du bot), jamais traité par un flux.

#### FR-2 : Déclenchement manuel par bouton

Sébastien peut lancer un flux via un clavier Telegram personnalisé à 3 boutons ("Bilan matin", "Bilan midi", "Bilan soir").

**Consequences (testable) :**
- Aucun flux ne démarre sans action explicite de Sébastien (pas d'horaire, pas de géolocalisation en v1).

#### FR-3 : Règle vocal/texte stricte

Le système répond dans le même canal que le message entrant : vocal → réponse vocale, texte → réponse texte.

**Consequences (testable) :**
- Un message vocal ne reçoit jamais une réponse texte, et inversement, quel que soit le flux.

### 4.2 Flux Matin

**Description :** Réalise UJ-1. Bilan bref de la veille (si disponible), état des tâches restantes, et priorités du jour, en séquence fixe.

**Functional Requirements :**

#### FR-4 : Déroulé du flux matin

Sébastien peut obtenir, en une séquence d'étapes sans saut ni répétition d'une étape déjà répondue, un rappel du bilan d'hier, l'état des tâches actives, un contexte scrum si pertinent, et 2-3 priorités du jour proposées puis validées.

**Consequences (testable) :**
- Si une fiche de la veille existe, son contenu est cité dans la première réponse de l'agent.
- Le flux se termine par un résumé de 2-3 lignes soumis à confirmation explicite de Sébastien.
- Le contenu confirmé est écrit dans la fiche du jour (champ plan matin).

### 4.3 Flux Midi

**Description :** Réalise UJ-2. Micro-ajustement borné, pas un nouveau bilan — conçu pour tenir dans ~20 minutes de trajet.

**Functional Requirements :**

#### FR-5 : Déroulé du flux midi

Sébastien peut ajuster les priorités de l'après-midi en au maximum 4 échanges, à partir du plan du matin déjà connu.

**Consequences (testable) :**
- Le flux ne repose jamais une question dont la réponse figure déjà dans la fiche du jour (plan matin).
- Le flux se termine, au plus tard au 4e échange, par un résumé d'une ligne à confirmer.
- Le contenu confirmé est écrit dans la fiche du jour (champ ajustement midi).

### 4.4 Flux Soir

**Description :** Réalise UJ-3. Bilan complet, question par question, basé sur ce que le système connaît déjà (plan matin + ajustement midi).

**Functional Requirements :**

#### FR-6 : Déroulé du flux soir

Sébastien peut faire, dans l'ordre et une question à la fois, le point sur : statut de chaque tâche prévue connue (fait / en cours / pas fait), imprévus, ce qui reste à terminer, suivis pour demain (et avec qui), feeling général, estimation du temps de procrastination — puis un résumé de 3-4 lignes à confirmer.

**Consequences (testable) :**
- Aucune question sur un point déjà couvert par le plan matin ou l'ajustement midi n'est reposée depuis zéro.
- Si une réponse est anormalement courte sur un point qui appelle du détail, l'agent reformule une seule fois, puis accepte la réponse quelle qu'elle soit — jamais de deuxième relance sur le même point.
- Tout contenu hors des 6 points structurés est capturé dans le champ "Autres", non reformaté.
- Le contenu confirmé est écrit dans la fiche du jour (tous les champs structurés + "Autres").

**Notes :** La règle "reformuler une seule fois" s'applique par point du bilan (jusqu'à 6 reformulations possibles dans le flux) — confirmé par Sébastien.

### 4.5 Continuité et mémoire

**Description :** Sous-tend les trois flux. Garantit que chaque flux démarre avec le bon contexte sans mémoire de session partagée entre flux, et que les échanges à l'intérieur d'un même flux restent cohérents.

**Functional Requirements :**

#### FR-7 : Fiche journalière structurée

Le système maintient une fiche par jour dans le vault Obsidian, avec des champs fixes (date, plan matin, ajustement midi, prévu vs fait, imprévus, suivis pour demain + avec qui, feeling général, temps de procrastination estimé) et un champ libre "Autres".

**Consequences (testable) :**
- Chaque jour calendaire produit au plus une fiche.
- Les champs fixes sont présents (même vides) dès qu'un flux a écrit dans la fiche du jour.

#### FR-8 : Lecture systématique de la fiche du jour

Chaque flux lit la fiche du jour existante avant de commencer et n'a accès à aucune mémoire de session héritée d'un autre flux.

**Consequences (testable) :**
- Le flux midi voit le plan matin même si le flux matin s'est terminé plusieurs heures plus tôt et dans une session distincte.
- Deux flux du même jour n'ont jamais accès aux échanges bruts l'un de l'autre — uniquement au contenu déjà écrit dans la fiche.

#### FR-9 : Mémoire intra-trajet

À l'intérieur d'un même flux, l'agent garde le contexte des échanges précédents jusqu'à la confirmation finale.

**Consequences (testable) :**
- Une information donnée en début de flux n'a pas besoin d'être répétée plus tard dans le même flux.

### 4.6 Analyse de patterns à la demande

**Description :** Réalise UJ-4. Fonction distincte des trois flux quotidiens, jamais déclenchée automatiquement.

**Functional Requirements :**

#### FR-10 : Déclenchement explicite uniquement

L'analyse de patterns ne s'exécute que sur demande explicite de Sébastien, formulée en langage naturel.

**Consequences (testable) :**
- Aucune tâche planifiée ni déclencheur automatique ne lance cette fonction.

#### FR-11 : Recherche ciblée dans l'historique

L'analyse s'appuie sur une sélection pertinente de fiches passées, jamais sur l'injection de tout l'historique.

**Consequences (testable) :**
- Le système répond en restant factuel : en l'absence de thème réellement récurrent (pas 1-2 mentions isolées), il l'indique explicitement plutôt que d'extrapoler.

`[NOTE FOR PM]` La mécanique exacte de sélection ("pertinente") n'est pas encore définie — voir Open Question #2.

#### FR-12 : Proposition de nouveau champ, jamais application automatique

Quand un thème récurrent net est détecté, le système propose un champ structuré dédié ; il n'applique jamais ce changement seul et ne réécrit jamais les entrées passées.

**Consequences (testable) :**
- Le schéma de la fiche du jour ne change qu'après validation humaine explicite de Sébastien.
- Aucune fiche existante n'est modifiée rétroactivement par cette fonction.

**Out of Scope :** Le mécanisme exact d'ajout du champ au schéma (manuel ou semi-automatisé) est un point ouvert — voir §9.

`[NOTE FOR PM]` Voir Open Question #4 pour le mécanisme précis d'ajout de champ.

### 4.7 Gestion des erreurs

**Description :** Comportements attendus quand une entrée est inexploitable ou qu'un composant échoue, pour ne jamais faire perdre un bilan silencieusement.

**Functional Requirements :**

#### FR-13 : Transcription vocale inexploitable

Si une transcription vocale est vide ou incohérente, le système redemande par vocal plutôt que d'envoyer le contenu au LLM.

**Consequences (testable) :**
- Un audio inexploitable ne produit jamais d'appel LLM avec un contenu vide ou aberrant.

#### FR-14 : Message hors-sujet ou ambigu

Si un message ne correspond à aucune étape attendue du flux, le système demande une clarification plutôt que d'halluciner une réponse.

**Consequences (testable) :**
- Aucune information non fournie par Sébastien n'apparaît dans la fiche du jour.

#### FR-15 : Échec technique

Si un composant technique échoue (accès fichier, service indisponible), Sébastien reçoit une notification Telegram explicite et le bilan n'est pas enregistré partiellement sans qu'il le sache.

**Consequences (testable) :**
- Un échec technique en cours de flux ne produit jamais une fiche du jour incomplète sans notification associée.

### 4.8 Indépendance du modèle LLM

**Description :** Contrainte non négociable élevée au rang de capacité testable : aucune partie du produit ne doit dépendre d'un modèle LLM particulier.

**Functional Requirements :**

#### FR-16 : Point d'appel LLM isolé

Tous les flux passent par un point d'appel LLM unique et isolé du reste de la logique produit.

**Consequences (testable) :**
- Remplacer le modèle ou le fournisseur LLM ne nécessite de modifier qu'un seul point du système, jamais la logique ou les prompts propres à chaque flux.

## 5. Constraints and Guardrails

*Frontières contraignantes pour toute conception downstream — indépendantes du choix d'orchestration (§9, Open Question #1).*

- **Self-hosting.** Le système doit fonctionner sans dépendance à un service cloud tiers pour les données sensibles en production (orchestration, stockage, et à terme LLM et vocal). Exception assumée et temporaire : phase de test, où un LLM API externe (Claude/GPT-4o) sert délibérément à juger la qualité du système avant la bascule en production. Valide SM-4.
- **Confidentialité.** Le contenu des bilans (potentiellement des détails de travail secteur public) n'est **stocké et traité** que sur l'infrastructure de Sébastien une fois en production — aucun service tiers ne conserve ni n'analyse ce contenu. **Exception explicitement acceptée :** le transport réseau via Telegram (message et vocal) est un risque résiduel assumé, justifié par le chiffrement en transit, la restriction au `chat_id` unique (FR-1), et l'absence d'alternative self-hosted pour ce canal — à rouvrir en architecture si ce risque devenait inacceptable. Valide SM-4.
- **Pas d'automatisation silencieuse.** Aucune modification de la structure de données (nouveau champ, réécriture d'une entrée passée) sans validation humaine explicite — voir FR-12.
- **Sécurité des accès.** Identifiants et secrets ne sont jamais exposés en clair ni versionnés dans le dépôt de code, quel que soit l'outil retenu. Mécanisme précis laissé à l'architecture.
- **Temps de réponse (sécurité, pas seulement confort).** Le trajet se fait en conduisant : le temps de réponse de l'agent (STT → LLM → TTS) ne doit jamais donner l'impression d'attendre activement la machine au volant. Critère qualitatif, pas de borne chiffrée dans ce PRD — à valider empiriquement en phase architecture, en particulier après bascule vers un LLM local (latence GPU différente de l'API).

## 6. Non-Goals (Explicit)

- Pas de rappels proactifs spontanés (le système ne parle jamais sans que Sébastien ait initié un flux).
- Pas d'intégration avec un outil de scrum d'équipe (Jira ou autre).
- Pas de déclenchement automatique par géolocalisation ou horaire en v1.
- Pas d'ambition d'innovation algorithmique ou de moat technique — la valeur est dans l'ajustement à l'usage réel, pas dans la nouveauté.
- Pas de correction ou de jugement sur le contenu des bilans (tâches non faites, procrastination, imprévus) — le système capture, il ne critique pas.

## 7. MVP Scope

### 7.1 In Scope

- Flux matin, midi, soir — déclenchement manuel uniquement.
- Vocal natif Telegram (in/out) et texte, selon la règle stricte du canal.
- Fiche journalière Obsidian comme mécanisme de continuité inter-flux ; mémoire de session intra-trajet.
- Analyse de patterns à la demande, avec proposition (jamais application automatique) de nouveaux champs.
- Point d'appel LLM isolé ; phase de test sur LLM API (Claude/GPT-4o) avant bascule en LLM local self-hosted.
- Gestion d'erreurs explicite (transcription, ambiguïté, échec technique).

### 7.2 Out of Scope for MVP

- Rappels proactifs, intégration Jira, déclenchement automatique — différés, non priorisés (cf. §6).
- Bascule complète vers infrastructure 100% locale (LLM + vocal self-hosted sans API) — prévue mais postérieure au MVP, cf. brief §Vision.
- Choix final de l'outil d'orchestration (n8n vs OpenClaw) — **hors scope du présent PRD**, tranché en phase architecture (voir addendum du brief).

## 8. Success Metrics

*Reprises et précisées à partir des critères de succès confirmés au brief produit.*

**Primary**
- **SM-1** : Usage soutenu — Sébastien se sert régulièrement des flux matin et soir, sans que ce soit ressenti comme une corvée. Critère qualitatif, pas de quota chiffré (délibéré : un seuil rigide pénaliserait vacances, télétravail, semaines sans trajet — sans que ce soit un échec du produit). Valide FR-4, FR-6.
- **SM-2** : Continuité parfaite — aucune question reposée par les flux midi/soir sur un point déjà répondu dans la fiche du jour. Cible : 0 occurrence observée. Valide FR-5, FR-6, FR-8.

**Secondary**
- **SM-3** : Fiabilité perçue du bilan du soir — auto-évaluation hebdomadaire de Sébastien (le contenu relu reflète honnêtement la journée). Valide FR-6, FR-7.

**Counter-metrics (à ne pas optimiser)**
- **SM-C1** : Taux de relance ("insister deux fois") ne doit jamais augmenter dans le temps — un système qui pousse plus fort pour faire monter l'usage (SM-1) casse la contrainte "n'insiste pas une seconde fois" (FR-6). Contrebalance SM-1.

**Gate (binaire, à valider avant bascule production)**
- **SM-4** : Aucun contenu de bilan n'est stocké ni traité hors de l'infrastructure de Sébastien une fois en production (exceptions explicitement acceptées : phase de test sur LLM API, et transport réseau via Telegram — voir §5 Confidentialité). Vérifié par revue technique avant bascule, pas mesuré en continu.

## 9. Open Questions

1. **Outil d'orchestration** (n8n vs OpenClaw) — non tranché, cf. addendum du brief produit. À trancher via spike en phase `bmad-architecture`.
2. Mécanique exacte de la recherche ciblée dans l'historique pour FR-11 (grep par plage de dates vs requêtes à un plugin/API Obsidian) — architecture.
3. Détails d'implémentation du switch LLM pour FR-16 (variables d'environnement, structure des credentials multiples) — architecture.
4. Mécanisme concret d'ajout du nouveau champ structuré une fois validé (FR-12) — modification manuelle du schéma ou semi-automatisée — architecture.
5. Schéma YAML exact de la fiche du jour (FR-7) — noms de champs, types, convention de nommage des fichiers — architecture.

## 10. Assumptions Index

*Toutes les hypothèses inférées ont été confirmées avec Sébastien (voir `.memlog.md`) — aucune ouverte à ce stade.*
