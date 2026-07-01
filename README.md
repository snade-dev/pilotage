# all-in-one-pilotage

Poste de commandement du projet **all-in-one**. Ce dépôt ne contient PAS de code : il
centralise le suivi de l'avancement, la feuille de route et les décisions, pour les deux
dépôts de code du projet :

- **Frontend** — Next.js (repo `All-in-One-Partner-Hub`)
- **Backend** — NestJS (repo backend)

## Pourquoi un repo séparé
Les sessions Claude (Chat ou Code) n'ont pas de mémoire fiable entre elles. La source de
vérité du projet doit donc vivre dans git. Comme le projet est réparti sur deux repos de
code, un état éclaté (un `.claude/` par repo) diverge vite. Ce repo unique règle le problème :
un seul `ETAT.md`, une seule feuille de route.

## Contenu
```
.claude/
├── ROADMAP.md    ← plan global (les 7 phases). Change rarement.
├── ETAT.md       ← LE fichier vivant : où on en est, sur CHAQUE repo. À lire en premier.
├── DECISIONS.md  ← journal des décisions structurantes et de leur « pourquoi ».
└── SESSIONS/     ← une entrée par session de travail.
```

## Comment l'utiliser avec Claude Code
Claude Code travaille dans un repo de code à la fois et ne voit pas automatiquement un
`.claude/` situé ailleurs. Deux façons de faire :

1. **Dossier parent commun (recommandé)** — place les trois dépôts côte à côte :
   ```
   all-in-one/
   ├── all-in-one-pilotage/   (ce repo)
   ├── All-in-One-Partner-Hub/  (front)
   └── backend/                 (back)
   ```
   Ouvre Claude Code depuis `all-in-one/` : il accède aux trois.

2. **Chemin explicite dans le prompt** — indique à Claude Code le chemin du `.claude/` de
   pilotage à lire au début et à mettre à jour à la fin.

## Règle d'or
Le suivi se met à jour **dans le même commit** que le travail qu'il décrit (côté code) ou
juste après validation (côté pilotage). Un suivi qui diverge de la réalité est pire que pas
de suivi.

> Note : ce dépôt documente le projet, jamais le fait qu'une IA y contribue.
