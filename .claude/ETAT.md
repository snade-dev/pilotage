# État All-in-One — 2026-07-04 (soir)

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
| 6 · App cliente — front Expo (Étape 1+2 consultation) | app | ✅ commité sur main |
| 6 · M1-c (comptes tél.) + M1-d (commande retrait) — BACKEND | back | ✅ mergés main, migration appliquée en local |
| 6 · App cliente Étape 3 (compte, panier, commande, suivi) | app | 🟡 CODÉ (branche `feat/etape3-transactionnel`), typecheck vert, **non commité — attente tests Expo Go + relecture** |
| 6 · Feed public `/public/v1/feed` (accueil produits) | back | 🟡 codé + testé en réel, non commité |
| 6 · Kanban « Commandes en ligne » (maquette 2j) | front-gestion | 🟡 codé, typecheck vert, non commité |

## En cours / en attente
- [ ] **Tests manuels Étape 3** sur Expo Go + Partner Hub (parcours complet : compte →
      panier → créneau → suivi ↔ kanban commerçant ; commande de test #00DkLEY en file).
- [ ] **Commits après relecture** : 3 repos (app branche étape 3 ; back feed ; hub kanban).
- [?] **Push / remote** de l'app Expo — à confirmer (travail non poussé = risque disque).
- [ ] Traîne `any` Phase 3 (front) — tâche de fond.

## Prochaine action
1. Tester l'Étape 3 bout-en-bout (détail : SESSIONS/2026-07-04-client-etape3-transactionnel.md).
2. Relecture + commits (app, back, hub) — messages sans trace IA.
3. Sourcer le fournisseur SMS/WhatsApp (impl prod `VerificationCodeSender`) + démarches
   Wave/Orange (bloque M2).
4. Backend (améliorations relevées) : exposer le `code` d'erreur (filtre global l'avale),
   valider `creneauRetrait` futur, porter le nom d'établissement sur la commande client.

## Décisions ouvertes
- Majoration « application mobile » : le prix public inclut-il la majoration, ou le prix brut ?

## Pièges ACTIFS
- App Expo : URLs d'images à consommer en ABSOLU (dépend du fix images backend mergé).
- Erreurs backend côté app : PAS de champ `code` (avalé par le filtre) → statut + message.
- Multi-repos (4) : parallèle seulement entre repos différents.
- Prisma 5.7 : `where` sur relation to-one dans `include` silencieusement ignoré.
- Règle permanente : aucune trace de Claude/IA dans le git.
- docker rm/prune : jamais de filtre large (incident conteneur passé).
