# Assistant Trajet quotidien

Assistant conversationnel (Telegram, vocal/texte) arrimé aux trajets en voiture de Sébastien (matin / midi / soir), pour structurer la planification et les bilans quotidiens. Orchestration n8n, stockage Obsidian, full self-hosted à terme.

Voir [`docs/brief-projet-assistant-trajet.md`](docs/brief-projet-assistant-trajet.md) pour le brief de projet complet (contexte, contraintes non négociables, décisions établies, points ouverts).

## Méthode de travail — BMAD-METHOD

Ce repo utilise [BMAD-METHOD](https://bmadcode.com/) pour aller du brief à la spec technique (PRD, architecture, epics/stories), via des skills Claude Code installés dans `.claude/skills/`.

- `_bmad/` — installation BMAD (agents, config, scripts). Généré par l'installeur, ne pas éditer à la main sauf dans `_bmad/custom/`.
- `_bmad-output/planning-artifacts/` — brief, PRD, architecture produits par les skills BMAD.
- `_bmad-output/implementation-artifacts/` — epics, stories, statut de sprint.
- `docs/` — connaissance long terme du projet (le brief source, futures notes de référence).

### Prochaine étape

Invoquer le skill `bmad-product-brief` (mode *Update*, à partir de `docs/brief-projet-assistant-trajet.md`) ou directement `bmad-prd` pour produire le PRD à partir des décisions déjà prises dans le brief.
