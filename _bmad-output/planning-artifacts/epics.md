---
stepsCompleted: [step-01, step-02]
inputDocuments:
  - '_bmad-output/planning-artifacts/prds/prd-assistant-trajet-quotidien-2026-09-02/prd.md'
  - '_bmad-output/planning-artifacts/architecture/architecture-assistant-trajet-quotidien-2026-09-03/ARCHITECTURE-SPINE.md'
  - '_bmad-output/specs/spec-assistant-trajet-quotidien/SPEC.md'
  - '_bmad-output/specs/spec-assistant-trajet-quotidien/glossary.md'
  - '_bmad-output/specs/spec-assistant-trajet-quotidien/prompts.md'
  - '_bmad-output/specs/spec-assistant-trajet-quotidien/test-sequence.md'
---

# Assistant Trajet quotidien - Epic Breakdown

## Overview

This document provides the complete epic and story breakdown for Assistant Trajet quotidien, decomposing the requirements from the PRD and Architecture spine (via SPEC.md/CAP-N as the canonical contract) into implementable stories. Pas de document UX — canal Telegram uniquement, pas d'écrans à concevoir.

## Requirements Inventory

### Functional Requirements

FR1 (CAP-1): Le bot ne répond qu'aux messages du `chat_id` de Sébastien.
FR2 (CAP-1): Déclenchement manuel via clavier Telegram à 3 boutons (Bilan matin/midi/soir).
FR3 (CAP-1): Réponse dans le même canal que le message entrant (vocal→vocal, texte→texte).
FR4 (CAP-2): Flux matin — bilan bref d'hier, tâches actives, priorités du jour, séquence fixe sans saut.
FR5 (CAP-3): Flux midi — micro-ajustement, max 4 échanges, basé sur le plan du matin.
FR6 (CAP-4): Flux soir — bilan complet question par question, une reformulation max par point.
FR7 (CAP-5): Fiche journalière structurée (frontmatter YAML + champ "Autres") dans le vault Obsidian.
FR8 (CAP-5): Chaque flux lit la fiche du jour existante avant de démarrer ; pas de mémoire partagée entre flux.
FR9 (CAP-5): Mémoire intra-trajet (contexte gardé à l'intérieur d'un même flux).
FR10 (CAP-6): Analyse de patterns déclenchée uniquement à la demande explicite.
FR11 (CAP-6): Recherche ciblée dans l'historique (pas d'injection massive).
FR12 (CAP-6): Proposition de nouveau champ structuré, jamais appliquée automatiquement, jamais de réécriture des fiches passées.
FR13 (CAP-7): Transcription vide/incohérente → redemander par vocal plutôt qu'envoyer au LLM.
FR14 (CAP-7): Message hors-sujet/ambigu → demander clarification, ne pas halluciner.
FR15 (CAP-7): Échec technique → écriture atomique + notification Telegram explicite.
FR16 (CAP-8): Appel LLM isolé derrière un point d'entrée unique — swap de modèle = un seul point de changement.

### NonFunctional Requirements

NFR1 (Self-hosting): Aucune dépendance cloud pour les données sensibles en production ; exception assumée : phase de test sur LLM API.
NFR2 (Confidentialité): Contenu des bilans stocké/traité uniquement sur l'infra de Sébastien ; transport réseau Telegram accepté comme risque résiduel explicite.
NFR3 (Pas d'automatisation silencieuse): Aucune modification de schéma ni réécriture de fiche passée sans validation humaine.
NFR4 (Sécurité des accès): Identifiants/secrets jamais en clair ni versionnés.
NFR5 (Temps de réponse = sécurité): Qualitatif — ne jamais donner l'impression d'attendre activement en conduisant ; instrumenté, mesuré empiriquement, pas de borne chiffrée a priori.
NFR6 (Contre-indicateur SM-C1): Ne jamais optimiser la fréquence d'usage au prix de la règle "jamais insister deux fois" (FR6).

### Additional Requirements

**Starter Template : aucun.** Service Python sur mesure (pas de scaffold/générateur) — impacte Epic 1 Story 1 (mise en place du squelette du projet).

- Paradigme hexagonal (ports & adapters), cœur en graphes d'états **LangGraph** (spine AD-1, AD-8) — le séquencement strict des flux est une machine à états, pas un effet de prompt.
- Topologie **deux hôtes** : orchestrateur Python en Docker sur Unraid (CPU) ; machine séparée avec GPU (GTX 5060 16GB) pour Ollama + LiteLLM + service STT (faster-whisper). Liaison via **Tailscale** (spine, Structural Seed).
- **LiteLLM** (proxy sur machine GPU) comme point d'appel LLM unique compatible OpenAI, unifiant Ollama (local) et Claude/GPT-4o (phase de test) — spine AD-4.
- **Checkpointer LangGraph** sur SQLite, clé `chat_id:date:flow_type`, monté en volume Docker (spine AD-2, Consistency Conventions — persistance).
- **Vault Obsidian** monté en volume Docker (jamais dans le filesystem éphémère du conteneur).
- **Schéma YAML de la fiche du jour** fixé — spine AD-10 (champs : `date`, `plan_matin`, `ajustement_midi`, `taches[]`, `imprevus`, `suivis_demain[]`, `feeling_general`, `procrastination_estimee_min`, + corps "Autres").
- **`ObsidianStore`** expose `read_today()` (continuité inter-flux, matin/midi/soir) et `read_range()` (historique multi-jours, patterns uniquement) — spine AD-3.
- Transport **python-telegram-bot 22.8** en mode webhook, exposé via **Cloudflare Tunnel** (déjà décidé, non négociable) sans port ouvert en clair.
- TTS **Piper** (OHF-Voice/piper1-gpl, GPL-3.0) in-process sur l'orchestrateur ; STT **faster-whisper 1.2.1** en service réseau sur la machine GPU — spine AD-5.
- Écriture atomique de la fiche du jour + notification Telegram sur tout échec technique du cycle de vie complet d'un flux — spine AD-6 (distinct des boucles de validation conversationnelles FR13/FR14, spine AD-8).
- Séquence de tests voulue par Sébastien (companion `test-sequence.md`) : (1) briques isolées, (2) flux complet en texte, (3) même flux en vocal, (4) journée simulée 3 trajets, (5) usage réel sur vacances avant données de travail sensibles — sert de colonne vertébrale au séquencement des epics ci-dessous.

### UX Design Requirements

Aucun — pas de document UX produit (interface Telegram uniquement, pas d'écrans à concevoir).

### FR Coverage Map

FR1: Epic 1 - Restriction d'accès chat_id
FR2: Epic 1 - Déclenchement manuel par bouton
FR3: Epic 2 - Règle vocal/texte stricte
FR4: Epic 1 - Déroulé du flux matin
FR5: Epic 1 - Déroulé du flux midi
FR6: Epic 1 - Déroulé du flux soir
FR7: Epic 1 - Fiche journalière structurée
FR8: Epic 1 - Lecture systématique de la fiche du jour
FR9: Epic 1 - Mémoire intra-trajet
FR10: Epic 3 - Déclenchement explicite de l'analyse de patterns
FR11: Epic 3 - Recherche ciblée dans l'historique
FR12: Epic 3 - Proposition de nouveau champ, jamais auto-appliquée
FR13: Epic 2 - Transcription vocale inexploitable
FR14: Epic 1 - Message hors-sujet ou ambigu
FR15: Epic 1 - Échec technique (écriture atomique + notification)
FR16: Epic 1 - Point d'appel LLM isolé

## Epic List

### Epic 1: Bilans quotidiens en texte
Sébastien peut faire ses trois bilans (matin/midi/soir) par écrit sur Telegram, avec continuité entre eux et un historique fiable dans Obsidian. Cœur du système, le plus gros risque technique (séquencement LangGraph, LiteLLM, topologie deux hôtes) validé avant d'ajouter la complexité vocale.
**FRs covered:** FR1, FR2, FR4, FR5, FR6, FR7, FR8, FR9, FR14, FR15, FR16

### Epic 2: Interaction vocale
Sébastien peut faire les mêmes trois bilans entièrement en vocal (parle, on lui répond en vocal), sans jamais avoir à basculer vers le texte.
**FRs covered:** FR3, FR13
**Dépend d'Epic 1.**

### Epic 3: Analyse de patterns à la demande
Sébastien peut demander à l'assistant d'analyser ses bilans passés et de lui proposer, quand c'est pertinent, un nouveau champ structuré — jamais appliqué sans son accord.
**FRs covered:** FR10, FR11, FR12
**Dépend d'Epic 1.** Indépendant d'Epic 2.

### Epic 4: Bascule production sécurisée
Sébastien peut utiliser le système en confiance sur ses vraies données de travail secteur public — bascule du LLM de l'API de test vers Ollama local, validation que rien ne transite hors de son infra (hors risque Telegram déjà accepté).
**FRs covered:** aucune nouvelle — valide NFR1, NFR2 (gate confidentialité), NFR5 (latence réelle)
**Dépend d'Epic 1** (idéalement 2 et 3 aussi livrés avant bascule finale).
