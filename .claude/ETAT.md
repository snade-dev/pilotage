# État du projet all-in-one — dernière mise à jour : 2026-07-01

> Suivi centralisé des DEUX repos. Lire ce fichier en premier à chaque session.
> Repos : Frontend (Next.js, All-in-One-Partner-Hub) · Backend (NestJS).

## Vue d'ensemble
| Phase | Repo | État |
|---|---|---|
| 0 Hygiène | front + back | ✅ |
| 1 Sécurité | back | ✅ |
| 2 Tests « argent » | front + back | ✅ |
| Module Statistiques (backend) | back | ✅ mergé |
| Module Statistiques (frontend) | front | 🔗 en cours (chantier parallèle) |
| 3 Qualité de type | front | ⬜ à venir (après le front stats) |
| 4 Robustesse données | back | 🔗 en cours (chantier parallèle) |
| 5 Duplication | front + back | ⬜ |
| 6 Fonctionnalités | front + back | ⬜ (stats = 1re, en cours) |

## Chantiers actifs EN PARALLÈLE (repos différents → sûrs)
- [ ] **Frontend Statistiques** (repo FRONT, branche `feat/stats-supermarche-frontend`) :
      page consommant les 4 endpoints stats. Code neuf typé proprement (pas de nouveau `any`).
- [ ] **Phase 4 Robustesse données** (repo BACK, branche `feat/phase-4-data-robustness`) :
      transactions Prisma, audit d'isolation multi-tenant, index, journal d'audit.

## Fait (validé + mergé)
### Backend
- [x] Phase 0 Hygiène — voir SESSIONS/2026-07-01-phase-0-1.md
- [x] Phase 1 Sécurité (auth globale, CORS, secrets, throttler, Helmet) — voir SESSIONS/2026-07-01-phase-0-1.md
- [x] Phase 2 Tests « argent » (105 unit + 8 e2e guards) — voir SESSIONS/2026-07-01-phase-2-money.md
- [x] Module Statistiques agrégation (4 endpoints : resume, chiffre-affaires,
      classement-produits, classement-vendeurs ; scope établissement/consolidé, CA net,
      pagination/tri ; 134 unit + 12 e2e) — voir SESSIONS/2026-07-01-stats-supermarche.md
### Frontend
- [x] Phase 2 socle Vitest (3 fichiers, 15 tests : totaux document, promotions, TVA)

## Prochaine action
1. (BACK) Appliquer la migration d'index du module stats (`prisma migrate deploy`) — NON FAIT.
2. Mener en parallèle les deux chantiers actifs ci-dessus.
3. Une fois le frontend stats mergé → démarrer la Phase 3 (qualité de type) sur le repo front, seul.

## Décisions ouvertes / à trancher
- Le webhook Stripe existe-t-il ? (à confirmer si pas déjà tranché en Phase 1)

## Pièges connus
- Multi-repos : front (Next.js) et back (NestJS) séparés. Ne PAS mener deux chantiers du
  même repo en parallèle (conflits). Parallèle OK seulement entre repos différents.
- Migration d'index stats non appliquée en base → endpoints stats lents tant que non déployée.
- Backend : login via LocalAuthGuard, pas JWT — ne pas casser avec l'auth globale (traité Phase 1).
- Frontend : `ignoreBuildErrors: true` encore actif + ~247 `any` (chantier Phase 3).
- Règle permanente : aucune trace de Claude/IA dans le git (commits, auteurs, commentaires).
