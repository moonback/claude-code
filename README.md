# Claude Code — Analyse technique et documentation complète (FR)

> **Contexte** : ce dépôt contient le dossier `src/` d’une version fuitée de Claude Code (CLI Anthropic), initialement exposée le **31 mars 2026** via un fichier source map dans un package npm.

---

## 1) README traduit (FR)

### Fuite et origine
Le **31 mars 2026**, le code source de la CLI Claude Code a été rendu accessible à partir d’un fichier `.map` publié dans le registre npm. La map référencait un bundle TypeScript non obfusqué, récupérable depuis un bucket R2.

### Vue rapide
Claude Code est un assistant d’ingénierie logicielle en ligne de commande : il peut lire/éditer des fichiers, lancer des commandes shell, rechercher dans un codebase, gérer des workflows Git, et orchestrer des sous-agents.

Ce dépôt regroupe le dossier `src/` exposé.

- **Date de fuite** : 31/03/2026
- **Langage** : TypeScript
- **Runtime** : Bun
- **UI terminal** : React + Ink
- **Échelle observée** : ~1 900 fichiers, > 500 000 lignes

### Structure générale (traduite)
- `main.tsx` : point d’entrée CLI
- `commands.ts` : registre et routage des commandes
- `tools.ts` : registre des outils
- `Tool.ts` : types et contrats des outils
- `QueryEngine.ts` : moteur d’interaction LLM
- `services/` : intégrations externes (API, OAuth, MCP, LSP, etc.)
- `bridge/` : pont IDE (VS Code / JetBrains)
- `coordinator/` : orchestration multi-agents
- `memdir/` : mémoire persistante
- `plugins/` : extensibilité

### Capacités cœur (traduction + reformulation)
- **Système d’outils** : chaque tool définit son schéma d’entrée, ses permissions et son exécution.
- **Système de commandes** : commandes slash (`/commit`, `/review`, `/doctor`, `/memory`, etc.).
- **Couche services** : API Anthropic, MCP, OAuth, LSP, analytics, politiques d’organisation.
- **Bridge IDE** : communication bidirectionnelle IDE ↔ CLI.
- **Permissions** : validation systématique avant exécution d’un tool.
- **Feature flags** : chargement conditionnel et élimination de code mort au build.

---

## 2) Vue d’ensemble du projet

### C’est quoi ?
Un **copilote terminal “agentique”** : au-delà d’un chat, le système agit sur l’environnement de dev (fichiers, shell, réseau, git, sessions, agents).

### À quoi ça sert ?
- Automatiser des tâches d’ingénierie répétitives.
- Accélérer l’exploration/refactor d’un dépôt volumineux.
- Servir de couche d’assistance cross-outils (CLI + IDE + services externes).

### Pour qui ?
- **Développeurs** : productivité quotidienne.
- **Tech leads / Staff engineers** : coordination de tâches complexes via sous-agents.
- **Platform / DevEx** : outillage standardisé et pilotable.
- **Équipes produit** : prototypage rapide piloté par prompts + commandes.

### Cas d’usage concrets
- Génération d’un plan de migration multi-fichiers, puis exécution contrôlée.
- Analyse de bugs via grep/lsp + exécution de tests + patchs guidés.
- Préparation de commits/reviews avec contexte projet.
- Exécution d’agents spécialisés (frontend, backend, sécurité) dans une équipe d’agents.

---

## 3) Architecture détaillée

### Vue macro (mental model)
```text
Utilisateur/IDE
   │
   ▼
CLI (main.tsx + commands.ts)
   │
   ▼
QueryEngine (raisonnement + boucle tool-calls)
   │
   ├── Tool Runtime (tools.ts / Tool.ts / tools/*)
   │       ├── FS / Shell / Web / Git / Tasks / Agents
   │       └── Permission Gate
   │
   ├── Coordinator (multi-agent)
   │       └── Team agents + messagerie
   │
   ├── Memory (memdir + extraction/sync)
   │
   ├── Plugins
   │
   ├── Bridge IDE (bridge/*)
   │
   └── Services (api, mcp, oauth, lsp, analytics...)
```

### Core engine (LLM / QueryEngine)
- Coordonne les appels modèle (streaming, retries, boucle d’actions).
- Décide quand appeler un tool selon le plan de résolution.
- Suit les coûts/tokens et gère la continuité conversationnelle.

### Tool system
- `Tool.ts` définit le contrat commun (input schema, permissions, résultat).
- `tools.ts` enregistre/rend disponibles les implémentations.
- `tools/*` exécute la logique réelle (fichiers, shell, web, agents, etc.).

### Command system
- `main.tsx` parse les arguments CLI et initialise le runtime.
- `commands.ts` route les commandes slash et choisit les handlers.
- `commands/*` contient la logique métier de chaque commande.

### Multi-agent system
- `coordinator/` orchestre des sous-agents spécialisés.
- Gestion de **teams** d’agents, messages inter-agents, délégation de tâches.
- Permet une parallélisation logique de sous-problèmes.

### Memory system
- `memdir/` conserve des mémoires persistantes localement.
- Services associés : extraction auto de mémoires, synchro mémoire d’équipe.
- Objectif : continuité de contexte à long terme.

### Plugin system
- `plugins/` apporte une extension du comportement sans modifier le cœur.
- Utile pour intégrer des workflows entreprise/custom.

### Bridge system
- `bridge/` connecte CLI et IDE en bidirectionnel.
- Cas : pilotage de session depuis extension, callbacks de permission, auth de session.

### Services layer
- `services/api` : communication backend/LLM.
- `services/mcp` : serveurs MCP et outils distants.
- `services/lsp` : introspection sémantique du code.
- `services/oauth`, `analytics`, `policyLimits`, etc. : couche transverse.

### Data flow (simplifié)
1. Entrée utilisateur (CLI ou IDE).
2. Enrichissement contexte (fichiers, session, mémoire).
3. QueryEngine génère une action.
4. Appel tool(s) si nécessaire (après permissions).
5. Résultats tool injectés dans le contexte.
6. Réponse finale ou itération jusqu’à terminaison.

### Execution flow (simplifié)
```text
Input -> Parse command -> Build context -> LLM decide -> Tool call?
      -> Permission check -> Execute tool -> Observe result
      -> Continue loop -> Final output -> Optional persist memory
```

---

## 4) Explication des composants clés

### `QueryEngine`
- **Rôle** : cerveau opérationnel de la session.
- **Fonctionnement** : boucle pensée/action, streaming, retries, suivi tokens.
- **Importance** : détermine robustesse, qualité des décisions, coût et latence.

### `Tool.ts`
- **Rôle** : contrat standard de tous les outils.
- **Fonctionnement** : types d’entrée/sortie, permissions, états de progression.
- **Importance** : garantit cohérence, sécurité et composabilité des tools.

### `commands.ts`
- **Rôle** : catalogue + routage des commandes slash.
- **Fonctionnement** : enregistrement conditionnel selon environnement/flags.
- **Importance** : surface UX principale de la CLI.

### `main.tsx`
- **Rôle** : bootstrap global.
- **Fonctionnement** : parse CLI, initialise runtime, renderer Ink/React, préfetch parallèle.
- **Importance** : impact direct sur performance de démarrage et stabilité.

### `services/*`
- **Rôle** : isoler les intégrations externes.
- **Fonctionnement** : API, auth, protocoles, analytics, limites de policy.
- **Importance** : séparation des responsabilités + testabilité + maintenabilité.

### `tools/*`
- **Rôle** : exécution concrète d’actions dans le monde réel.
- **Fonctionnement** : implémentations indépendantes sous contrat commun.
- **Importance** : transforme le LLM en agent exécutable, pas seulement conversationnel.

### `coordinator/*`
- **Rôle** : orchestrer plusieurs agents.
- **Fonctionnement** : dispatch des tâches, suivi d’état, messages, fusion des sorties.
- **Importance** : scaler la résolution de problèmes complexes.

---

## 5) Système d’agents (très important)

### Comment fonctionnent les agents
- Un agent principal reçoit l’objectif global.
- Il délègue des sous-objectifs à des sub-agents (via `AgentTool` / team tools).
- Chaque sub-agent opère avec son contexte, puis renvoie ses résultats.

### Sub-agents
- Spécialisables par rôle (ex. “refactor”, “tests”, “documentation”).
- Peuvent être temporaires (one-shot) ou persister selon le flux.

### Coordination
- Le coordinator agit comme **scheduler + arbitre**.
- Il évite les collisions de tâches et garde une vue d’ensemble.

### Team agents
- Des équipes d’agents peuvent être créées/supprimées dynamiquement.
- Utiles pour paralléliser des streams de travail (analyse, implémentation, validation).

### Cas d’usage
- **Automation pipeline** : analyser backlog → proposer plan → exécuter patchs.
- **Copilote autonome** : traiter une issue complète avec checkpoints humains.
- **Ops de codebase massive** : exploration/recherche/triage distribués.

---

## 6) Système de tools

### Comment un tool fonctionne
1. Déclaration du schéma d’entrée (validation stricte).
2. Définition des permissions requises.
3. Exécution de la logique.
4. Retour d’un résultat structuré + logs/progress.

### Lifecycle d’un tool
```text
Register -> Validate input -> Permission gate -> Execute -> Return -> Trace
```

### Permissions
- Contrôle systématique avant action sensible.
- Modes observables : interactif, planifié, auto/bypass (selon policy/config).
- Peut demander approbation utilisateur ou appliquer des règles automatiques.

### Exemples concrets
- **FileRead** : lecture de code, images, PDF, notebooks.
- **Bash** : exécution shell encadrée.
- **WebSearch/WebFetch** : recherche et récupération web.
- **FileEdit/FileWrite** : mutation locale de fichiers.
- **MCP/LSP** : accès outils distants et intelligence sémantique.

---

## 7) Commandes CLI importantes

### Commandes à fort impact
- `/commit` : prépare et crée un commit.
- `/review` : revue de changements.
- `/doctor` : diagnostic environnement.
- `/memory` : consultation/gestion mémoire persistante.
- `/skills` : gestion des compétences/outils spécialisés.
- `/tasks` : suivi de tâches.
- `/config` : réglages runtime.
- `/diff` : inspection des modifications.
- `/resume` : reprise de session.

### Exemples d’usage réel
- **Avant session** : `/doctor` puis `/config` pour sécuriser le contexte.
- **Pendant implémentation** : `/tasks` + `/diff` + `/review`.
- **Finalisation** : `/commit` puis partage/synchronisation.

---

## 8) Stack technique expliquée

- **Bun** : runtime rapide, bundling moderne, startup faible latence.
- **TypeScript strict** : sécurité de typage sur une base très large.
- **React + Ink** : paradigme UI composant, appliqué au terminal.
- **Commander.js** : parsing CLI robuste.
- **Zod** : validation runtime fiable des schémas tools/commandes.
- **ripgrep** : recherche code ultra-performante.
- **MCP + LSP** : extension protocolée + compréhension sémantique.
- **OpenTelemetry + gRPC** : observabilité et transport standardisés.
- **GrowthBook** : feature flags pour livraisons progressives.

---

## 9) Patterns d’architecture

### Lazy loading
- Chargement différé de modules coûteux pour réduire TTI en CLI.

### Parallel prefetch
- Pré-initialisation parallèle (settings, keychain, flags) au démarrage.

### Feature flags
- Activation ciblée de capacités (et élimination de code mort au build).

### Agent orchestration
- Décomposition d’un problème en tâches parallélisables + agrégation.

### Diagramme de décision (simplifié)
```text
Need capability?
 ├─ stable core path -> direct tool
 ├─ optional/expensive -> lazy load
 ├─ experiment rollout -> feature flag
 └─ complex task -> multi-agent orchestration
```

---

## 10) Sécurité & risques

### Implications d’une fuite de source
- Exposition d’implémentations internes (surfaces d’attaque mieux connues).
- Risque de rétro-ingénierie des flux, permissions, endpoints et comportements.
- Accélération d’imitations concurrentes (copie de patterns/UX).

### Risques potentiels
- Mauvaise configuration des permissions tools (exécution non maîtrisée).
- Utilisation agressive du shell/web sans sandbox/policies fortes.
- Fuites de secrets dans logs, mémoires persistantes ou plugins tiers.

### Bonnes pratiques si réutilisation
- Mettre une **politique de permissions deny-by-default**.
- Isoler l’exécution shell (sandbox, allowlists, quotas).
- Chiffrer/segmenter mémoire persistante.
- Activer audit logs + traçabilité tool-level.
- Faire revue sécurité des plugins avant activation.

---

## 11) Opportunités produit (angle startup)

### Transformer en SaaS
- **Positionnement** : “Agent Engineering Platform” pour équipes software.
- **Modèle** : seat-based + usage-based (tools, tokens, automations).

### Idées de features monétisables
- Workspaces multi-projets avec governance centralisée.
- Templates d’agents métier (SRE, SecOps, QA, Migration).
- “Approval center” (workflow conformité + RBAC avancé).
- Observabilité business (temps gagné, MTTR, qualité PR).

### Différenciation possible
- Plus orienté “ops de codebase” que copilote inline type Copilot.
- Plus orchestrateur terminal/agents que simple éditeur assisté type Cursor.
- Avantage clé : combinaison **CLI + IDE + mémoire + équipes d’agents**.

### Trajectoire MVP → Scale
1. **MVP** : assistant terminal + 6 tools cœur + mémoire locale.
2. **V1 équipe** : RBAC, audit, templates agents, dashboards.
3. **Scale** : multi-tenant, marketplace plugins, politiques org avancées.
4. **Enterprise** : SOC2, SSO/SCIM, data residency, policy engine complet.

---

## 12) Résumé exécutif (TL;DR)

1. Claude Code est un copilote terminal orienté action (pas seulement chat).
2. Son moteur central est une boucle LLM + outils.
3. Les tools sont typés, validés, et gouvernés par permissions.
4. Les commandes slash structurent l’expérience utilisateur.
5. La couche services encapsule API/protocoles/auth/analytics.
6. Le bridge IDE relie les usages terminal et éditeur.
7. Le système multi-agents permet la délégation et la parallélisation.
8. La mémoire persistante améliore la continuité inter-sessions.
9. Les feature flags facilitent rollout progressif et expérimentation.
10. Le potentiel produit est fort pour une plateforme SaaS DevEx/AgentOps.

---

## Annexes (bonus)

### Comparaison mentale rapide
- **Copilot** : fort dans l’assistance inline éditeur.
- **Cursor** : expérience IDE augmentée orientée agent.
- **Ce design CLI-agent** : plus proche d’un **orchestrateur programmable** qui opère sur tout l’environnement de développement.

### Modèle mental utile
Pensez le système comme un **“OS d’agents pour ingénierie logicielle”** :
- le LLM = planificateur cognitif,
- les tools = appels système,
- les permissions = sécurité kernel,
- le coordinator = scheduler distribué,
- la mémoire = disque contextuel.


---

## 13) Installation, exécution et réutilisation du code

> ⚠️ **Important** : ce dépôt est une base de code fuitée. Son exécution peut être incomplète selon les dépendances internes non publiées.

### Prérequis
- **OS** : macOS / Linux recommandé
- **Runtime** : Bun (version récente)
- **Node.js** : utile pour certains outils de l’écosystème
- **Git** : pour cloner et gérer les modifications

### Installation (mode exploration)
```bash
git clone <votre-fork-ou-copie>
cd claude-code
bun install
```

### Exécution locale (selon entrypoints disponibles)
Comme le dépôt contient surtout `src/`, le point d’entrée réel dépend de la structure de build présente dans votre copie.

Essais usuels :
```bash
# Vérifier les scripts disponibles
cat package.json

# Exécuter le mode dev si disponible
bun run dev

# Exécuter la CLI si un script existe
bun run start
# ou
bun run cli
```

Si aucun script n’est défini :
```bash
# Vérification TypeScript (si tsconfig présent)
bunx tsc --noEmit

# Exécution directe d’un entrypoint potentiel
bun run src/main.tsx
```

### Vérifications minimales recommandées
```bash
# Lint / typecheck / tests (si disponibles)
bun run lint
bun run typecheck
bun test
```

### Réutiliser le code dans un autre projet (approche pragmatique)

#### Option A — Réutilisation du pattern d’architecture
- Reprendre les concepts plutôt que copier tel quel :
  - `QueryEngine` (boucle LLM + tools)
  - `Tool.ts` (contrat unique des outils)
  - `coordinator/` (orchestration multi-agents)
  - `services/` (intégrations isolées)
- Avantage : faible dette légale/technique, adaptation plus rapide.

#### Option B — Extraction modulaire contrôlée
- Extraire d’abord les modules peu couplés (ex. utilitaires, schémas, patterns tools).
- Encapsuler chaque module derrière des interfaces stables.
- Ajouter tests de non-régression avant intégration.

#### Option C — “Clean-room reimplementation” (recommandé en contexte produit)
- Spécifier les comportements attendus.
- Réimplémenter depuis zéro avec vos propres conventions.
- Conserver seulement les idées architecturales, pas le code brut.

### Plan de réutilisation en 5 étapes
1. **Cartographier** les dépendances (`services`, `tools`, `query`).
2. **Prioriser** 3 capacités cœur (lecture fichier, bash, recherche).
3. **Isoler** un contrat de tool unifié + permission gate.
4. **Implémenter** un QueryEngine minimal avec boucle tool-call.
5. **Durcir** sécurité (sandbox, audit logs, RBAC, quotas).

### Risques à anticiper en réutilisation
- Couplage fort à des APIs internes ou endpoints non accessibles.
- Manques de configuration (secrets, policies, feature flags internes).
- Ambiguïtés légales/licence selon provenance du code.

### Recommandations opérationnelles
- Commencer par un **prototype local non connecté** aux systèmes sensibles.
- Remplacer les dépendances externes par des abstractions mockées.
- Mettre en place observabilité et permissioning dès le premier sprint.
- Faire une revue sécurité et légale avant toute mise en production.
