# Session 2026-07-01 — Mise en place du repo de pilotage centralisé

## Contexte de départ
`.claude/` existait uniquement dans le repo backend. Passage imminent à deux chantiers en
parallèle (front stats + back Phase 4) → besoin d'un état unique couvrant les deux repos.

## Fait
- Création du repo `all-in-one-pilotage` avec README + `.claude/` (ROADMAP, ETAT, DECISIONS, SESSIONS).
- ETAT.md refondu pour suivre les DEUX repos en un seul tableau.

## Décidé
- Option B : pilotage centralisé (voir DECISIONS.md).

## Laissé de côté (volontairement)
- Suppression effective de l'ancien `.claude/` du repo backend : à faire côté utilisateur
  pour éviter deux sources de vérité (noté comme action).

## Prochaine action recommandée
- Geler/retirer l'ancien `.claude/` du backend.
- Appliquer la migration d'index stats en base.
- Lancer les deux chantiers parallèles.
