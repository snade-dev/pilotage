# État All-in-One — 2026-07-10

> Curseur du projet. Ne contient que le PRÉSENT et le futur immédiat.
> Historique des jalons terminés → ROADMAP.md (coché) + SESSIONS/ (détail).
> Repos (5) : pilotage · back (NestJS) · front-gestion (Next.js) · app-cliente (Expo) · app-livreur (Expo).

## Vue d'ensemble
| Phase / Chantier | Repo | État |
|---|---|---|
| 0–2 (hygiène, sécurité, tests) | front+back | ✅ |
| 3 Qualité de type | front | 🟡 traîne `any` hooks/+app/ |
| 4 Robustesse données | back | ✅ mergé + e2e |
| 5 Duplication (ciblée) | front+back | ✅ |
| 6 · POS offline | back+front | ✅ mergé |
| 6 · App cliente (catalogue, compte OTP, panier, commande retrait, suivi) | app | ✅ mergé main |
| Billing mobile money (CinetPay) + abonnements | back+front | ✅ (migration + clés env à déployer) |
| Onboarding partenaire (email + essai 14j) | back+front | ✅ |
| Équipes & perfs vendeurs · Rayons · Péremption/lots · Inventaire physique · Conditionnement d'achat | back+front | ✅ mergés main |
| App mobile cliente (repo dédié, étapes 1-3 + kanban commerçant) | app | ✅ mergé main |
| Flux cuisine restaurant (statut PRETE + onglet Cuisine POS) | back+front | ✅ (2 migrations à déployer) |
| **Livraison à domicile + suivi GPS livreur** | 5 repos | ✅ **implémenté, E2E backend vert, login livreur testé device** |
| **Couverture de tests pré-déploiement (Phase B)** | back+front+app | 🟡 **écrite, suites vertes, branches `test/coverage-predeploy` NON commitées (relecture)** |

## Tests pré-déploiement — état détaillé (chantier courant)
- **Backend** : 356 → 412 tests (410 verts + 2 skips documentant un bug). Footgun jest-e2e corrigé
  (les db-specs ne peuvent plus tourner hors conteneur jetable). Ajouts : séparation tokens
  client/employé sur vraies routes (31 tests), expiration/reprise OTP, machine à états commandes
  retrait + décrément défaut + isolation écriture, anti-fuite API publique (clés EXACTES + feed),
  isolation mouvements stock. **BUG BLOQUANT consigné** : `supermarche/mouvement-produit-magasin-stock`
  sans aucune vérification tenant contre le JWT (lecture + création cross-tenant) — 2 tests skippés
  à réactiver après correctif. → `SESSIONS/2026-07-10-phase-b-tests-backend.md`
- **Front Hub** : 23 → 191 tests (188 verts + 3 skips = bugs réels), 24 fichiers spec. Outillage
  jsdom/RTL/fake-indexeddb ajouté, typecheck 0 erreur, suite relancée 3× sans flaky. Bugs confirmés :
  payload POS resto sans options/remises de ligne (bloquant), TVA resto en dur sans tauxTVAId (majeur),
  fallback ligneCommandeId=saleId dans useRefunds (majeur). → `SESSIONS/2026-07-10-phase-b-tests-front-hub.md`
- **App cliente** : 0 → 203 tests (199 verts + 4 skips = bugs réels : clientRequestId partagé
  retrait/livraison, message « session expirée » sur panne réseau du refresh, course d'hydratation
  du panier, détour connexion pendant `chargement`). Couverture totale US-01→US-14. Outillage
  jest-expo 57 + RNTL 13, contrats backend figés dans les mocks, 3 runs sans flaky, tsc 0 erreur.
  → `SESSIONS/2026-07-10-phase-b-tests-app-cliente.md`
- **Phase C FAITE (2026-07-10)** — les 8 bugs corrigés, tous les skips réactivés, plus aucun skip :
  - **Backend** : isolation tenant corrigée sur `/supermarche/mouvement` + jumeau
    `/restaurant/mouvement-plat-stock` (id d'établissement depuis le JWT + seam
    EstablishmentAccess, pattern inventaire) ; `configuration-etablissement` verrouillée pareil
    (8 routes, ids clients vérifiés, contrat Hub préservé) ; `GET /restaurant/plat/recherche`
    renvoie les options (POS resto) ; suites 291+47+85 vertes ×2.
    → `SESSIONS/2026-07-10-phase-c-fix-backend.md`
  - **Front Hub** : options + `tauxTVAId` dans le payload vente resto (online/offline), total UI
    aligné sur le recalcul serveur (TVA sur HT puis remise globale sur TTC), remise par article
    resto refusée à la saisie (DTO backend ne la supporte pas), remboursement sans `lineId`
    refusé + `lineId` mappé depuis la réponse de vente ; 193 tests verts ×2, typecheck 0, build OK.
    → `SESSIONS/2026-07-10-phase-c-fix-front-hub.md`
  - **App cliente** : clientRequestId séparé retrait/livraison, erreur réseau ≠ « session
    expirée » au refresh (tokens conservés), ajout panier pré-hydratation préservé, Commander
    inerte pendant le chargement de session ; 204/204 verts ×2, tsc 0 erreur.
    → `SESSIONS/2026-07-10-phase-c-fix-app-cliente.md`
- **Décisions ouvertes issues de la Phase C** :
  - Remise par ligne resto : `LigneVentePlatDto` ne la supporte pas (parité supermarché à faire
    côté back si voulue) — d'ici là le POS resto renvoie vers la remise globale.
  - `GET /configuration-etablissement/liste` (« admin ») liste toutes les configs à tout staff —
    intention à trancher avant d'ajouter un guard.

## Livraison — état détaillé (chantier courant)
- **Backend** : `StatutLivraison` (RECUE→EN_PREPARATION→PRETE→EN_LIVRAISON→LIVREE), `PositionLivreur`,
  lat/lng adresse, frais/rayon sur Supermarché (parité resto). Module `common/commande-livraison/`,
  contrôleurs client / commerçant / `livreur/*`. Permissions `GERER_LIVRAISONS` + `EFFECTUER_LIVRAISONS`
  seedées. Encaissement cash + décrément stock à LIVREE, code confirmation 4 chiffres.
- **Nouveau repo `all-in-one-livreur`** (Expo) : login staff, tournée, carte OSM, démarrer/livrer,
  suivi GPS foreground (polling). Bug worklets/reanimated transitif corrigé (overrides pnpm, comme l'app cliente).
- **Fix login insensible à la casse** (back) : `partenaire`/`utilisateur`/`client` cherchés en ILIKE +
  trim — les claviers mobiles capitalisent l'email. Backend rebuildé + relancé.
- Compte de test livreur : `livreur.test@aio.dev` / `LivreurTest123!` (resto snade flash, frais 1000 FCFA,
  rayon 10 km). Détail complet → mémoire `project_livraison`.

## En cours / en attente
- [ ] **Relecture + commit des correctifs Phase C** (3 branches `test/coverage-predeploy`,
      working trees non commités) puis merge des branches sur main.
- [ ] **Test manuel bout-en-bout livraison sur devices** : login livreur OK ; reste le flux complet
      (commande client → kanban Hub → assigner → démarrer → suivi carte live → livrer avec code).
- [ ] **Migrations à déployer en prod** (appliquées en local seulement) :
      `add_livraison`, cuisine (×2), conditionnement, billing, rayons, et antérieures (compte tél., stats).
- [ ] **Push des repos** vers leurs remotes (merges locaux seulement).
- [ ] Traîne `any` Phase 3 (front) — tâche de fond.

## Prochaine action
1. Dérouler le test manuel bout-en-bout de la livraison (2 devices, ou livreur simulé en curl).
2. Regrouper et déployer les migrations Prisma en attente.
3. Pousser les repos.

## Décisions ouvertes
- Majoration « application mobile » : le prix public inclut-il la majoration, ou le prix brut ?
- Tracking GPS livreur en arrière-plan : reporté (dev build EAS + expo-task-manager, hors Expo Go).

## Pièges ACTIFS
- Apps Expo : bug worklets/reanimated transitif (0.10.2 vs natif 0.10.0) → épingler via `pnpm.overrides`
  + deps directes, sinon crash au lancement sans écran d'erreur.
- App Expo : URLs d'images à consommer en ABSOLU.
- Erreurs backend côté app : PAS de champ `code` (avalé par le filtre) → statut + message.
- Multi-repos (5) : parallèle seulement entre repos différents.
- Prisma 5.7 : `where` sur relation to-one dans `include` silencieusement ignoré.
- Migration : piège DROP INDEX trgm parasite dans le SQL généré (à retirer à la main avant apply).
- Whitelist : jamais de supermarcheId/restaurantId en body/query — le JWT fait foi.
- `prisma generate`/`build` EPERM : un `node dist/main` en cours verrouille le query engine → l'arrêter d'abord.
- Règle permanente : aucune trace de Claude/IA dans le git.
- docker rm/prune : jamais de filtre large (incident conteneur passé) — base dev = conteneur `aio-db`.
