# Glossaire — Assistant Trajet quotidien

- **Bilan** — Terme générique désignant l'activité de planification ou de compte-rendu propre à un flux (ex. "bilan matin", "bilan soir"). Se déroule *pendant* un flux ; son contenu confirmé est écrit dans la fiche du jour, qui en est la trace persistante.
- **Fiche du jour** — Fichier markdown unique par journée dans le vault Obsidian (`{YYYY-MM-DD}.md`), frontmatter YAML structuré (schéma : voir spine, AD-10) + champ libre "Autres". Une fiche par date, jamais réécrite rétroactivement sans validation humaine.
- **Flux** — Un des trois rendez-vous quotidiens (Flux Matin, Flux Midi, Flux Soir), déclenché manuellement. `flow_type ∈ {matin, midi, soir, patterns}`.
- **Trajet** — Fenêtre de temps réelle (voiture) pendant laquelle un flux se déroule ; contrainte de forme (échanges courts, vocal adapté), pas un objet du système.
- **Mémoire intra-trajet** — Continuité conversationnelle entre les échanges d'un même flux, le même jour.
- **Continuité intra-journée** — Passage d'information entre flux (matin → midi → soir) via lecture de la fiche du jour, jamais via mémoire de session partagée.
- **Champ "Autres"** — Zone libre non structurée de la fiche du jour, pour tout ce qui ne rentre pas dans les champs fixes.
- **Pattern** — Thème ou élément qui revient de façon répétée (pas 1-2 mentions isolées) dans le champ "Autres" et/ou les champs structurés sur plusieurs fiches du jour.
- **chat_id** — Identifiant Telegram unique de Sébastien ; seule identité autorisée à interagir avec le bot.
