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
| 6 · App cliente M1-d (commande retrait) — BACKEND | back | ✅ commité (branche `feat/m1d-commandes-retrait`) |
| 6 · App cliente M1-c (comptes tél. vérifiés) — BACKEND | back | 🟡 codé, unit verts (291/291), e2e-BD à rejouer (Docker HS), non commité (relecture) |
| 6 · App cliente M1-d front (écrans commande/suivi) | app | ⬜ à venir (après M1-c) |

## En cours / en attente
- [?] **Bug écran détail app** (« produit introuvable » au clic sur établissement) —
      CORRIGÉ ou ENCORE OUVERT ? ← à confirmer, ne pas archiver tant que non résolu.
- [?] **Push / remote** : app Expo poussée sur un remote ? fix images absolues mergé sur main distant ?
      ← à confirmer. Tant que non poussé = travail non sauvegardé (risque disque).
- [ ] Traîne `any` Phase 3 (front) — tâche de fond.

## Prochaine action
1. **Relire M1-c backend** (branche `feat/m1c-comptes-clients`, partant de M1-d, non commitée) :
   auth téléphone vérifiée (démarrer/vérifier code, session client), `VerificationCodeSender`
   (dev = log, garde-fou prod), rattachement fidélité par téléphone, `@CurrentClientId` déjà
   branché. Design : `.claude/DESIGN/2026-07-04-m1c-comptes-clients.md`.
   **Rejouer `pnpm test:e2e:db` après redémarrage de Docker Desktop** (le backend Docker est
   passé en lecture seule en cours de session → conteneur de test `aio-e2e-pg` bloqué ; 1er run
   48/49, l'unique échec = bug de test depuis corrigé). Puis commit + déployer la migration
   `add_compte_client_telephone`.
2. Confirmer que le catalogue s'affiche correctement sur Expo Go (clic établissement → produits).
3. Écrans commande/suivi + compte de l'app cliente (APRÈS relecture/merge M1-c).
4. Sourcer le fournisseur SMS/WhatsApp (impl réelle du `VerificationCodeSender`) + démarches
   Wave/Orange (bloque M2).

## Décisions ouvertes
- Majoration « application mobile » : le prix public inclut-il la majoration, ou le prix brut ?

## Pièges ACTIFS
- App Expo : URLs d'images à consommer en ABSOLU (dépend du fix images backend mergé).
- App cliente : ne construire les écrans commande/compte qu'APRÈS les endpoints backend correspondants.
- Multi-repos (4) : parallèle seulement entre repos différents.
- Prisma 5.7 : `where` sur relation to-one dans `include` silencieusement ignoré.
- Règle permanente : aucune trace de Claude/IA dans le git.
- docker rm/prune : jamais de filtre large (incident conteneur passé).