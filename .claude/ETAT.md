# État All-in-One — 2026-07-04

> Curseur du projet. Ne contient que le PRÉSENT et le futur immédiat.
> Historique des jalons terminés → ROADMAP.md (coché) + SESSIONS/ (détail).
> Repos : pilotage · back (NestJS) · front-gestion (Next.js) · app-cliente (Expo).

## Vue d'ensemble
| Phase / Chantier | Repo | État |
|---|---|---|
| 0–2 (hygiène, sécurité, tests) | front+back | ✅ |
| 3 Qualité de type | front | 🟡 traîne `any` hooks/+app/ |
| 4 Robustesse données | back | ✅ mergé + e2e |
| 5 Duplication (ciblée) | front+back | ✅ |
| 6 · POS offline | back+front | ✅ mergé |
| 6 · App cliente M1-a/b (images, API publique, catalogue) | back | ✅ |
| 6 · App cliente — front Expo (Étape 1+2) | app | 🟡 catalogue affiché, bug détail à confirmer |
| 6 · App cliente M1-d (commande retrait) — BACKEND | back | 🟡 codé + tests verts, non commité (relecture) |
| 6 · App cliente M1-c (comptes tél. vérifiés) / M1-d front | back+app | ⬜ à venir |

## En cours / en attente
- [?] **Bug écran détail app** (« produit introuvable » au clic sur établissement) —
      CORRIGÉ ou ENCORE OUVERT ? ← à confirmer, ne pas archiver tant que non résolu.
- [?] **Push / remote** : app Expo poussée sur un remote ? fix images absolues mergé sur main distant ?
      ← à confirmer. Tant que non poussé = travail non sauvegardé (risque disque).
- [ ] Traîne `any` Phase 3 (front) — tâche de fond.

## Prochaine action
1. **Relire M1-d backend** (branche `feat/m1d-commandes-retrait`, non commitée) puis commit +
   déployer la migration `add_commande_retrait`. Restaurer le stash `wip-media-absolute-urls`
   sur `fix/public-media-absolute-urls` (mis de côté pour garder la branche M1-d propre).
2. Confirmer que le catalogue s'affiche correctement sur Expo Go (clic établissement → produits).
3. Backend M1-c : comptes clients (auth téléphone vérifiée) — se branche sur `@CurrentClientId`.
   Puis écrans commande/suivi de l'app (APRÈS ces endpoints).
4. Sourcer le fournisseur SMS/WhatsApp (bloque M1-c) + démarches Wave/Orange (bloque M2).

## Décisions ouvertes
- Majoration « application mobile » : le prix public inclut-il la majoration, ou le prix brut ?

## Pièges ACTIFS
- App Expo : URLs d'images à consommer en ABSOLU (dépend du fix images backend mergé).
- App cliente : ne construire les écrans commande/compte qu'APRÈS les endpoints backend correspondants.
- Multi-repos (4) : parallèle seulement entre repos différents.
- Prisma 5.7 : `where` sur relation to-one dans `include` silencieusement ignoré.
- Règle permanente : aucune trace de Claude/IA dans le git.
- docker rm/prune : jamais de filtre large (incident conteneur passé).