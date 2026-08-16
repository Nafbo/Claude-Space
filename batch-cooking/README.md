# Batch Cooking — routine hebdomadaire GitHub Actions

Envoie automatiquement, chaque vendredi 18h00 (heure de Paris), le planning des repas
de la semaine à venir et la liste de courses (PDF en pièce jointe) à
`victor.bonnaf@gmail.com`.

Workflow : [`.github/workflows/batch-cooking-weekly.yml`](../.github/workflows/batch-cooking-weekly.yml)
Logique métier complète : [`skill/SKILL.md`](skill/SKILL.md) (copie du skill Claude
`batch-cooking`, à garder synchronisée manuellement si le skill évolue).

## ⚠️ Point de blocage connu — non résolu

Les connecteurs **Notion** et **Gmail** utilisés par le skill sont des serveurs MCP
distants à authentification **OAuth interactive** (callback `localhost` + navigateur).
Il n'existe pas de méthode officiellement supportée pour authentifier ce type de
connecteur dans un runner headless comme GitHub Actions.

Piste à tester (best-effort, sans garantie de durabilité) :

1. En local, connecter les serveurs MCP Notion et Gmail interactivement une fois
   (session Claude Code normale).
2. Repérer où Claude Code persiste les credentials obtenus pour ces serveurs
   (`claude mcp list` et doc officielle — l'emplacement/format exact varie selon la
   version).
3. Copier ce credential dans un secret GitHub (`CLAUDE_MCP_CREDENTIALS` ou équivalent),
   et compléter l'étape commentée "Restaurer les credentials MCP Notion/Gmail" dans le
   workflow pour le réinjecter au démarrage du job.
4. Tester plusieurs exécutions successives avant de faire confiance à la routine : ces
   tokens ont un cycle de vie/refresh qui peut nécessiter une ré-autorisation
   interactive à intervalle inconnu — **la routine peut casser silencieusement après un
   certain temps**.

Tant que cette étape n'est pas complétée, le job échoue de façon visible à l'appel des
outils Notion/Gmail — c'est le comportement voulu (pas d'échec silencieux).

### Procédure de ré-authentification en cas de panne

Si le job échoue avec une erreur d'authentification Notion ou Gmail (une issue GitHub
est créée automatiquement dans ce cas, voir plus bas) :

1. Reproduire l'étape 1 ci-dessus en local (reconnexion interactive des deux
   connecteurs).
2. Reproduire l'étape 2-3 : récupérer le nouveau credential et mettre à jour le secret
   GitHub correspondant (Settings → Secrets and variables → Actions).
3. Relancer manuellement le workflow (`workflow_dispatch`) pour confirmer que ça
   repasse, avant d'attendre le prochain vendredi.

## Secrets GitHub requis

| Secret | Usage |
|---|---|
| `ANTHROPIC_API_KEY` | Authentification de Claude Code lui-même (appel du modèle) — bien supportée en CI. |
| `CLAUDE_MCP_CREDENTIALS` (nom indicatif, à ajuster) | Credential(s) MCP Notion/Gmail obtenus via la procédure ci-dessus. Non créé automatiquement — à toi de le configurer une fois la piste testée. |

`GITHUB_TOKEN` (fourni automatiquement par Actions) sert uniquement à créer l'issue de
notification d'échec — indépendant de l'auth Notion/Gmail, donc fiable même si celle-ci
casse.

## Ordonnancement

Cron GitHub Actions en UTC uniquement, sans gestion automatique du changement d'heure.
Le workflow programme les deux horaires possibles (`0 16 * * 5` été / `0 17 * * 5`
hiver) et un job de garde vérifie l'heure réelle de Paris au démarrage pour ne laisser
passer que le bon déclenchement — au prix d'un run "à vide" (quelques secondes, job
`guard` seulement) deux fois par vendredi au lieu d'un.

## Fichiers copiés dans ce repo

- `skill/SKILL.md` — logique métier complète (recettes, planning, règles de
  conservation/anti-répétition, format du PDF).
- `skill/references/pdf-template.html` — gabarit HTML/CSS du PDF (converti via
  `wkhtmltopdf`, polices locales uniquement : pas d'accès réseau garanti dans le
  runner).

Claude Code n'a pas accès automatiquement aux skills utilisateur d'une conversation
interactive : cette copie est nécessaire pour que le job headless ait la même logique
sous les yeux.
