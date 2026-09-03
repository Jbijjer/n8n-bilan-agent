---
name: 'Adversarial Review — Architecture Spine, Assistant Trajet quotidien'
type: review
target: '_bmad-output/planning-artifacts/architecture/architecture-assistant-trajet-quotidien-2026-09-03/ARCHITECTURE-SPINE.md'
created: '2026-09-03'
---

# Revue adversariale — ARCHITECTURE-SPINE.md

Méthode : pour chaque AD, construire une paire concrète d'unités indépendantes (deux graphes de flux,
un adaptateur + le cœur, un futur contributeur écrivant un nouvel adaptateur de port) qui respectent
**chaque AD à la lettre**, et montrer qu'elles produisent quand même un système incohérent à
l'assemblage. 9 scénarios trouvés, classés par AD principal exposé.

---

## 1. AD-3 — Schéma de la fiche du jour non figé : `read_today()` ne garantit rien sur la forme

**Paire :** `core/graphs/matin.py` (écrit) vs `core/graphs/midi.py` (lit).

- `matin.py` respecte AD-3 à la lettre : il écrit exclusivement via `ObsidianStore.write()`, jamais
  via un canal parallèle. Il sérialise l'état du bilan matin en frontmatter YAML avec une clé
  `energie: 7` (entier 0-10).
- `midi.py` respecte AD-3 à la lettre : il lit exclusivement via `ObsidianStore.read_today()`, jamais
  le checkpointer de `matin`. Il a été développé/testé contre un fixture de fiche où l'auteur a choisi
  `energy_level: {score: int, note: str}` (objet imbriqué).
- Les deux graphes n'ont jamais lu le code l'un de l'autre — seul le contrat implicite de la fiche les
  relie, et ce contrat n'est écrit nulle part dans une AD (le spine le renvoie explicitement en
  Deferred : « Schéma YAML exact de la fiche du jour »). Résultat : `midi.py` plante ou lit une valeur
  vide à la première exécution réelle, en prod, malgré une conformité totale aux AD.
- **Gravité :** haute — AD-3 est justement l'AD censée garantir la cohérence inter-flux ; elle
  garantit le *canal* mais pas la *forme*, ce qui vide une partie de son intention.
- **AD à créer/durcir :** une AD (ou un companion "Fiche du jour — schéma") qui fixe le schéma YAML
  des champs partagés (noms, types, unités) et impose que toute nouvelle clé soit ajoutée par un seul
  point (ex. `core/state.py` partagé ou un schéma Pydantic commun importé par tous les graphes), avant
  que `matin`/`midi`/`soir`/`patterns` soient codés indépendamment.

## 2. AD-3 — `read_today()` ne couvre pas la lecture multi-jours qu'exige `patterns.py`

**Paire :** `core/graphs/patterns.py` (FR-10, FR-11, FR-12) vs `adapters/obsidian_fs.py`.

- `ObsidianStore` tel que nommé dans AD-3 n'expose que `read_today()`. `patterns.py` a par nature
  besoin d'un historique (analyse de patterns sur plusieurs jours, FR-11 « recherche ciblée »).
- Un contributeur qui suit AD-1 à la lettre (« core n'importe que des symboles de ports/ ») mais ne
  trouve pas de méthode d'historique sur le port a deux issues également « conformes » : (a) il ajoute
  `read_range()`/`read_history()` au port et à `obsidian_fs.py` de sa propre initiative, avec sa propre
  convention de retour (liste de strings brutes ? liste de dicts parsés ?) ; (b) il contourne le
  problème en importable `pathlib`/`glob` directement dans `core/graphs/patterns.py` pour lister les
  fichiers du vault — ce qui viole l'esprit d'AD-1 mais pas forcément sa lettre stricte, puisque
  `pathlib` n'est pas cité dans la liste d'exemples interdits (« Telegram, SDK LLM, whisper,
  filesystem » — filesystem l'est en fait, mais rien ne bloque techniquement l'import).
- Une équipe qui prend la voie (a) et une autre qui prend (b) — ou même deux devs qui prennent (a)
  avec des signatures différentes — ne s'assemblent pas.
- **Gravité :** haute — FR-10/11/12 sont un livrable explicite du spine et le port tel que décrit ne
  les couvre pas.
- **AD à créer/durcir :** étendre AD-3 (ou une nouvelle AD-3b) pour spécifier la surface complète du
  port `ObsidianStore` (au minimum `read_today()` et une méthode d'historique bornée), et rappeler
  explicitement que `core/graphs/*` n'importe jamais de module d'accès fichier, y compris `pathlib`.

## 3. AD-1 / AD-4 — Aucun contrat de forme sur les ports (sync/async, type de retour, exceptions)

**Paire :** `adapters/llm_litellm.py` (existant) vs un futur `adapters/llm_<autre>.py` écrit par un
nouveau contributeur pour un besoin non prévu (ex. appel direct à un modèle de secours si LiteLLM est
down).

- Le nouveau contributeur respecte AD-1 à la lettre : son adaptateur implémente `ports.LLMClient`, il
  est le seul à importer le SDK externe, `core/` ne connaît toujours que l'interface.
- Rien dans AD-1 ni AD-4 ne fixe : méthode `async def` vs `def`, forme du retour (`str` brut vs objet
  `{content, usage, finish_reason}`), taxonomie d'exceptions (une `LLMTimeoutError` maison vs laisser
  fuiter `httpx.ConnectError`/`openai.APIError`), ni support garanti du function-calling/streaming
  (AD-4 dit que le routage Ollama/Claude est « une config LiteLLM… jamais un changement de code côté
  orchestrateur » — mais ne garantit pas que tous les modèles routables exposent les mêmes capacités
  qu'utilisent les graphes).
- Concrètement : `matin.py` développé/testé contre GPT-4o (phase de test, function-calling fiable) et
  `soir.py` développé/testé contre un modèle Ollama local (JSON mode dégradé) partagent le même port —
  les deux « obéissent » à AD-4, mais l'un plante en prod dès que LiteLLM route vers l'autre backend,
  précisément le scénario qu'AD-4 prétend rendre indolore.
- **Gravité :** haute — c'est systémique : ça touche les 4 ports (`LLMClient`, `STTClient`,
  `TTSClient`, `ObsidianStore`), pas seulement LLM.
- **AD à créer :** une AD « Contrat de port » qui fixe pour chaque port : signature (sync/async),
  forme du type de retour, taxonomie d'exceptions communes (ex. toutes les erreurs adaptateur sont
  wrappées dans un type maison avant de remonter à `core/`), et pour `LLMClient` spécifiquement un
  sous-ensemble de capacités garanti (pas de dépendance à function-calling si tous les backends routés
  ne le supportent pas de façon homogène).

## 4. AD-2 — La source de « date » n'est pas fixée, et peut diverger en cours d'exécution d'un même flux

**Paire :** `core/graphs/soir.py` (nœud d'entrée) vs `core/graphs/soir.py` (nœud tardif du même
graphe, ou un deuxième flux qui recalcule `date` indépendamment).

- AD-2 dit seulement : `thread_id = f"{chat_id}:{date}:{flow_type}"`, jamais réutilisé entre flow_type
  ou dates différentes. Rien n'impose que `date` soit calculée **une fois**, au début du flow, et
  propagée dans l'état.
- Un flow `soir` lancé à 23:58 qui enchaîne plusieurs appels LLM (latence machine GPU + Tailscale) peut
  franchir minuit. Si un nœud tardif recalcule `datetime.now().date()` au lieu de lire la date figée
  dans l'état, le même run logique change de `thread_id` en plein milieu — le checkpointer voit deux
  threads distincts pour une seule conversation, chacun conforme à la lettre d'AD-2 (jamais réutilisé
  entre dates différentes — ici ce ne sont *pas* deux checkpointer réutilisés, ce sont deux créés).
  Le nom de fiche (`{YYYY-MM-DD}.md`, cf. Consistency Conventions) hérite de la même ambiguïté :
  `matin.py` du lendemain peut écrire dans la fiche de la veille ou l'inverse selon quel horodatage
  chaque flow choisit.
- **Gravité :** moyenne-haute — silencieux, dépend du fuseau horaire du conteneur Unraid et de la
  latence LLM, donc difficile à reproduire en test mais frappe en prod à l'heure exacte où `soir`
  tourne.
- **AD à durcir :** ajouter à AD-2 la règle « `date` est calculée une seule fois, à l'entrée du graphe
  (premier nœud), stockée dans l'état, et jamais recalculée par un nœud ultérieur » — plus fixer le
  fuseau horaire de référence (celui du conteneur, explicitement documenté, pas « l'heure système »
  implicite).

## 5. AD-2 — SQLite checkpointer partagé, aucune garantie de concurrence entre flows

**Paire :** un run `matin` déclenché par planification (cron/scheduler) et un run `patterns`
déclenché en tâche de fond, tous deux actifs en même temps sur le même fichier SQLite.

- Les deux respectent AD-2 à la lettre : `thread_id` distincts (`flow_type` différent), jamais
  réutilisés. Le Structural Seed montre un seul fichier SQLite (`ckpt`) partagé par tout le processus.
- SQLite a un modèle d'écriture concurrente limité (verrou fichier). Rien dans AD-2 n'impose un mode
  WAL, une connexion partagée avec sérialisation, ou une politique de retry sur `database is locked`.
  Un contributeur écrivant `checkpoint/sqlite_checkpointer.py` peut ouvrir une connexion par flow sans
  se soucier du voisin — chaque module est individuellement correct, l'ensemble lève des erreurs de
  verrouillage aléatoires dès que deux flows tournent en parallèle (webhook Telegram entrant pendant
  qu'un cron `soir` est en cours, cas réaliste vu le topology mono-processus).
- **Gravité :** moyenne — dégrade la fiabilité plutôt que la cohérence des données, mais viole l'esprit
  d'AD-6 (pas d'échec technique silencieux) si le retry n'est pas géré et que l'exception SQLite fuit
  sans notification.
- **AD à durcir :** AD-2 devrait fixer le mode d'accès SQLite (WAL + retry/backoff, ou connexion
  unique sérialisée process-wide) comme faisant partie du contrat, pas un détail d'implémentation
  laissé à l'auteur de `sqlite_checkpointer.py`.

## 6. AD-6 — Le contrat de notification d'erreur ne couvre que « avant une écriture réussie »

**Paire :** deux implémentations de la gestion d'erreur dans `core/graphs/matin.py`, toutes deux
lisant AD-6 à la lettre.

- AD-6 : « Toute exception levée **avant une écriture réussie** déclenche une notification Telegram…
  avant la fin du flux ». Lu strictement, ceci ne couvre pas une exception survenant *après* que
  `ObsidianStore.write()` a réussi (ex. un nœud tardif qui échoue en générant la réponse vocale finale,
  ou `patterns.py` qui écrit une fiche puis plante en tentant d'envoyer une synthèse).
- Implémentation A (prudente) : notifie sur **toute** exception non gérée dans le flow, qu'il y ait eu
  écriture ou non. Implémentation B (littérale) : ne notifie que si l'exception précède la première
  écriture réussie ; une fois la fiche écrite, les erreurs suivantes sont juste loguées. Les deux
  citent AD-6 comme justification. B laisse l'utilisateur sans notification pour une classe entière
  d'échecs (tout ce qui suit l'écriture), silencieusement — exactement ce qu'AD-6 prétend interdire
  (« aucun flux ne se termine sur erreur sans passer par ce chemin » — la deuxième phrase de la
  règle est plus large que la première, ce qui crée la contradiction interne).
- Deuxième angle, même AD : l'échec **STT** (transcription vocale) se produit dans
  `adapters/telegram_bot.py`, **avant** même l'invocation de `core/` — donc avant qu'aucun « flow » au
  sens d'AD-6 n'ait commencé. AD-6 dit « tous les flux » mais son mécanisme (notifier avant la fin du
  flux) présuppose qu'un flow est en cours. Un échec STT (ex. Tailscale coupé vers la machine GPU)
  n'est couvert par aucune AD : un contributeur peut légitimement choisir de logguer et abandonner
  silencieusement le message vocal, une autre équipe choisit de notifier — les deux sont hors du
  périmètre littéral d'AD-6.
- **Gravité :** haute — erreurs silencieuses envers l'utilisateur sont exactement le risque qu'AD-6 a
  été écrite pour éliminer ; le trou est dans l'AD elle-même, pas dans son application.
- **AD à durcir :** réécrire AD-6 pour lier la notification à « toute exception non interceptée
  survenant à n'importe quel moment du traitement d'un message entrant (y compris avant l'invocation de
  `core/`, ex. échec STT) », et non à la seule fenêtre « avant écriture réussie ».

## 7. AD-6 — Le port `Notifier` est nommé « abstrait » mais la règle le fige sur Telegram

**Paire :** `ports.Notifier` (interface) vs un futur adaptateur `adapters/notifier_email.py` ou
`notifier_pushover.py` écrit pour un besoin de secours (ex. notifier même si le webhook Telegram est
down).

- Le nouvel adaptateur respecte AD-1 à la lettre : il implémente `ports.Notifier`, seul lui importe le
  SDK email/pushover, `core/` ne change pas.
- Mais AD-6 dit littéralement « déclenche une notification **Telegram** via `ports.Notifier` » — le nom
  du canal est câblé dans le texte de la règle censée gouverner un port abstrait. Un contributeur qui
  implémente un `Notifier` non-Telegram respecte AD-1 mais contredit la lettre d'AD-6. Deux équipes
  peuvent légitimement se disputer sur laquelle est « conforme ».
- **Gravité :** basse-moyenne — plutôt une incohérence rédactionnelle qu'un risque d'exécution
  immédiat, mais elle signale que le port `Notifier` n'est pas vraiment traité comme point de swap
  malgré l'architecture hexagonale revendiquée.
- **AD à durcir :** reformuler AD-6 pour dire « … déclenche une notification via `ports.Notifier`
  (dont l'unique adaptateur v1 est Telegram) » — cohérent avec le traitement d'AD-4 pour LLMClient.

## 8. AD-1 — `config.py` n'est pas nommée comme hors-limites pour `core/`, alors qu'elle porte des secrets/adresses

**Paire :** `core/graphs/matin.py` (une implémentation disciplinée) vs `core/state.py` ou un nœud de
graphe écrit par un contributeur pressé.

- AD-1 interdit à `core/` d'importer « une librairie externe (Telegram, SDK LLM, whisper, filesystem) »
  et dit que seul `adapters/telegram_bot.py` importe `core/graphs/*`. Elle ne dit rien sur
  `config.py` (module interne au projet, pas une « librairie externe » au sens des exemples donnés).
- Un flow a besoin de `chat_id` pour construire `thread_id` (AD-2) et pour appeler `Notifier` (AD-6).
  Rien dans le spine ne dit comment `chat_id`/`date`/config arrivent dans l'état du graphe. Un
  contributeur discipliné les fait passer explicitement dans l'état initial construit par
  `telegram_bot.py` avant d'invoquer le graphe (respecte l'esprit d'AD-1). Un autre, pressé, fait
  `from config import AUTHORIZED_CHAT_ID` directement dans `core/graphs/matin.py` — ne viole la lettre
  d'aucun exemple cité par AD-1, obtient le même résultat fonctionnel, mais couple durablement `core/`
  à la mécanique de chargement d'environnement (et, si `config.py` un jour lit un secret ou une adresse
  réseau spécifique à un adaptateur, ce couplage devient un vrai trou dans la frontière hexagonale).
- **Gravité :** moyenne — dette silencieuse plutôt que panne immédiate, mais elle mine la promesse
  « remplacement peu coûteux » qu'AD-1 existe pour garantir.
- **AD à durcir :** étendre AD-1 pour dire explicitement que `core/` n'importe *que* `ports/` — pas
  `config.py`, pas de module d'accès environnement — et que tout paramètre requis par un graphe
  (chat_id, date, chemins) doit lui être injecté via l'état initial construit par l'adaptateur
  d'entrée.

## 9. AD-7 — Aucune règle de modalité pour les messages proactifs (flows déclenchés par planification)

**Paire :** un run `matin`/`soir` déclenché par un scheduler (pas par un message Telegram entrant) vs
la règle de branchement d'AD-7.

- AD-7 dit : « La règle vocal-in→vocal-out / texte-in→texte-out est appliquée par branchement dans cet
  adaptateur ». Cette règle présuppose un message entrant dont la modalité peut être détectée. Or les
  flows `matin`/`midi`/`soir` sont, par nature d'un « bilan quotidien », vraisemblablement initiés par
  le système (heure programmée) et non par un message utilisateur — il n'y a alors **aucune modalité
  entrante** sur laquelle brancher.
- Implémentation A : par défaut, un message proactif est toujours envoyé en texte (safe, simple).
  Implémentation B : le système retient la dernière modalité utilisée par l'utilisateur (stockée où ?
  aucune AD ne le prévoit) et la reproduit pour le message proactif. Les deux sont compatibles avec la
  lettre d'AD-7 puisque celle-ci ne couvre que le cas réactif ; elles produisent un comportement
  utilisateur différent selon quelle équipe a implémenté quel flow, et aucune des deux n'est «
  fausse » au regard du spine.
- **Gravité :** moyenne — expérience utilisateur incohérente entre flows plutôt que corruption de
  données, mais révèle un angle mort réel dans la seule AD qui gouverne la frontière transport.
- **AD à créer/durcir :** étendre AD-7 (ou ajouter une AD dédiée) pour couvrir le cas des messages
  initiés par le système : modalité par défaut explicite, et si une préférence de modalité doit
  persister, dire où elle vit (config statique `AUTHORIZED_CHAT_ID`-scoped ? fiche du jour ? état
  séparé ?) plutôt que de laisser chaque flow inventer sa propre réponse.

---

## Récapitulatif

| # | Scénario | AD principal exposé | Gravité |
| - | -------- | -------------------- | ------- |
| 1 | Schéma fiche du jour non figé (écrivain/lecteur divergent) | AD-3 | Haute |
| 2 | `read_today()` insuffisant pour `patterns.py` (multi-jours) | AD-3 | Haute |
| 3 | Pas de contrat de forme sur les ports (sync/async, retour, exceptions, capacités LLM) | AD-1 / AD-4 | Haute |
| 4 | Source de « date » non figée, dérive en cours de flow | AD-2 | Moyenne-haute |
| 5 | SQLite checkpointer partagé sans garantie de concurrence | AD-2 | Moyenne |
| 6 | Fenêtre de notification d'erreur trop étroite + échec STT hors périmètre | AD-6 | Haute |
| 7 | `Notifier` abstrait mais figé « Telegram » dans le texte d'AD-6 | AD-1 / AD-6 | Basse-moyenne |
| 8 | `config.py` non exclue des imports `core/`, couplage silencieux | AD-1 | Moyenne |
| 9 | Aucune règle de modalité pour les messages proactifs (flows planifiés) | AD-7 | Moyenne |

**9 scénarios de build incompatible trouvés**, chacun avec les deux unités conformes citées et l'AD à
créer/durcir en regard.
