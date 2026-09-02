---
title: "Assistant Trajet quotidien"
status: draft
created: 2026-09-02
updated: 2026-09-02
---

# Product Brief : Assistant Trajet quotidien

## Executive Summary

Un assistant conversationnel personnel, self-hosted, accessible par Telegram (vocal ou texte), qui accompagne Sébastien à trois moments de sa journée — arrimés à ses trajets en voiture du matin, du midi (~20 min) et du soir. Le matin, il aide à planifier ; le midi, il permet un micro-ajustement ; le soir, il fait un bilan complet en posant des questions ciblées plutôt que génériques. Il garde une mémoire structurée et consultable des journées passées et peut, à la demande, repérer des patterns récurrents dans ces bilans.

Le besoin part d'un contexte concret : Sébastien, programmeur/analyste dans le secteur public et diagnostiqué TDA, cherche un outil qui structure sa planification et ses bilans sans lui demander un effort de rédaction ou d'ouverture d'appli — le trajet en voiture, déjà présent dans sa routine, devient le moment naturel pour ce rituel.

## The Problem

Structurer sa planification quotidienne et faire un bilan honnête de sa journée demande un effort d'initiation et de rédaction que le TDA rend coûteux : ouvrir un outil, se souvenir de ce qui était prévu, formuler ce qui a été fait ou non, sans y penser sur le moment. Sans ce rituel, les tâches non finies, les imprévus et les suivis à faire se perdent d'une journée à l'autre, et les patterns récurrents (ce qui revient souvent comme frein ou distraction) restent invisibles faute d'être capturés de façon exploitable.

## The Solution

Un agent conversationnel qui vient au moment déjà disponible — le trajet en voiture — plutôt que d'exiger un moment dédié. Interaction vocale native (parle, on te répond vocalement) ou texte, questions posées une à la fois et adaptées au contexte déjà connu de la journée (pas de liste de questions génériques à chaque fois), déclenchement manuel simple (un bouton Telegram par moment de la journée). Chaque journée est une fiche structurée dans un vault Obsidian ; l'agent peut, sur demande explicite, analyser l'historique pour proposer d'évoluer sa propre structure de données quand un pattern net apparaît.

## What Makes This Different

Pas un journal ou un to-do générique : le point d'entrée est le trajet, pas l'écran. La différenciation n'est pas technique (aucune ambition de nouveauté algorithmique) mais d'ajustement — le système est pensé pour ne jamais alourdir l'usage (une question à la fois, pas de jugement sur ce qui n'est pas fait, arrêt après une seule relance) et pour rester entièrement sous le contrôle de Sébastien (self-hosted, aucune automatisation silencieuse qui modifierait ses données).

## Who This Serves

Utilisateur unique : Sébastien — programmeur/analyste, secteur public, petite équipe (4 développeurs, 1 product owner, 1 responsable conception/normes), TDA diagnostiqué. Succès pour lui : arriver à ses trajets suivants avec le contexte du précédent déjà en main, sans avoir eu à s'en souvenir ; finir une semaine avec un historique de bilans assez fiable pour s'y fier plutôt que pour se juger.

## Success Criteria

- [ASSUMPTION] Usage soutenu : les 3 trajets (ou au moins matin + soir) sont utilisés la majorité des jours ouvrés, sans ressenti de corvée.
- [ASSUMPTION] Continuité perçue : au trajet du midi et du soir, l'agent n'a jamais à reposer une question dont la réponse est déjà dans la fiche du jour.
- [ASSUMPTION] Fiabilité du bilan du soir : Sébastien retrouve, en se relisant une semaine plus tard, un historique qui reflète honnêtement sa journée (pas de trous, pas de bilans bâclés par lassitude).
- [ASSUMPTION] Confidentialité respectée dès la mise en production : aucun contenu de bilan ne transite par un service cloud une fois la phase de test terminée.

*(Ces critères sont proposés à partir du contexte fourni — à confirmer ou ajuster par Sébastien.)*

## Scope

**Dans le scope initial :**
- Flux matin / midi / soir, déclenchés manuellement via boutons Telegram.
- Vocal natif Telegram (in/out) et texte, mémoire de session intra-trajet, fiche journalière Obsidian comme continuité inter-trajets.
- Analyse de patterns à la demande explicite, avec proposition (jamais application automatique) de nouveaux champs structurés.
- Phase de test sur LLM API (Claude/GPT-4o) avant bascule en LLM local self-hosted.

**Hors scope initial** *(évolutions futures identifiées, pas à développer maintenant)* :
- Rappels proactifs spontanés (sans demande explicite).
- Intégration avec l'outil de scrum de l'équipe (Jira).
- Déclenchement automatique des flux (géolocalisation ou horaire).

## Vision

Si l'outil tient l'usage quotidien, l'étape suivante naturelle est la bascule complète vers l'infrastructure locale (LLM + vocal self-hosted, sans dépendance API), condition pour l'utiliser sereinement sur de vraies données de travail secteur public. Au-delà, la valeur qui se dégage le plus est la détection de patterns : un historique de bilans assez riche pour que l'agent commence à repérer, de lui-même sur demande, ce qui revient — et à faire évoluer sa propre structure de données en conséquence, sans jamais agir seul sur les données passées.

---

*Contraintes techniques non négociables, décisions d'architecture détaillées (stockage, mémoire, orchestration, vocal, LLM, prompts validés, gestion d'erreurs, sécurité), et points ouverts à approfondir : voir [`addendum.md`](./addendum.md) et le document source [`docs/brief-projet-assistant-trajet.md`](../../../../../docs/brief-projet-assistant-trajet.md).*
