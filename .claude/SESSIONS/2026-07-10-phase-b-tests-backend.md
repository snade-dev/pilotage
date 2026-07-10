# Backend Phase B — Couverture de tests pré-déploiement (2026-07-10)

Repo : `all-in-one-backend`. Branche : `test/coverage-predeploy`.
**Non commité** (attente relecture — `git status` montré au propriétaire).
Périmètre STRICT : fichiers de test + config de test uniquement — **zéro fichier
applicatif modifié**. Base de dev (`aio-db`) jamais touchée : e2e-BD exclusivement
via `pnpm test:e2e:db` (conteneur jetable `aio-e2e-pg`, port 5434).

## Comptes (avant → après)
- Unit (`pnpm test`) : 291 → **291** (inchangé, tous verts).
- e2e sans BD (`pnpm test:e2e`) : 16 → **47** (+31, tous verts).
- e2e-BD (`pnpm test:e2e:db`) : 49 → **74** (72 verts + 2 skips documentant un bug
  réel, cf. plus bas). Suite relancée **2×** : résultats identiques, zéro flaky.
- Total : 356 → **412** (+56, dont 2 skips volontaires).

## Correctif de config prioritaire (footgun)
`test/jest-e2e.json` : la regex `.e2e-spec.ts$` attrapait aussi les
`.db-e2e-spec.ts` SANS le garde-fou `setup-env.ts` → risque d'exécution des
suites BD contre la base de dev. Ajout de `testPathIgnorePatterns:
["\\.db-e2e-spec\\.ts$"]`. Vérifié : `--listTests` ne liste plus que les suites
sans BD.

## Couverture ajoutée
1. **Séparation des tokens client/employé sur les VRAIES routes**
   (`test/token-separation.e2e-spec.ts`, 31 tests, sans BD) : contrôleurs réels
   montés avec les 3 guards globaux dans l'ordre de prod (JwtAuthGuard →
   UserTypeGuard → PermissionsGuard), services métier bouchonnés. Token client
   → 403 sur mouvements de stock, POS supermarché, commandes de retrait
   commerçant (SM + resto) ; token employé (partenaire ET utilisateur) → 403 sur
   `/v1/commandes-retrait`, `/v1/favoris`, profil client ; sans token → 401
   partout sauf routes OTP `@Public` ; contrôles positifs (bon token → 200/201,
   y compris utilisateur avec/sans permission `GERER_INVENTAIRE`).
2. **OTP client** (`test/e2e-db/client-auth.db-e2e-spec.ts`, +2 tests) :
   code périmé refusé même correct (réponse neutre `CODE_INVALIDE`, aucun compte
   créé) ; après verrouillage (5 essais) + cooldown, un nouveau code est émis et
   fonctionne (le verrou ne condamne pas le numéro), l'ancien code restant mort.
   (Épuisement des essais et cooldown étaient déjà couverts.)
3. **Commandes retrait — machine à états**
   (`test/e2e-db/commande-retrait-transitions.db-e2e-spec.ts`, 7 tests) :
   sauts interdits (RECUE→PRETE, RECUE→RETIREE) ; retours interdits depuis
   PRETE ; RETIREE/REFUSEE/ANNULEE terminaux (double-clic « Retirer » = 409,
   stock décrémenté UNE seule fois, 1 seul mouvement) ; annulation client OK
   depuis RECUE/EN_PREPARATION, refusée depuis PRETE ; retrait d'une variante
   SPÉCIFIQUE décrémente aussi la variante par défaut (invariant Σ) ; isolation
   d'écriture (un client ne peut pas annuler la commande d'un autre) et de liste
   (file du commerçant B, `listerMes` du client).
4. **API publique /public/v1 — anti-fuite**
   - `test/e2e-db/public-exposure.db-e2e-spec.ts` (6 tests) : ensemble **EXACT**
     des clés exposées figé pour chaque forme (carte/fiche établissement SM et
     resto, carte/détail item produit et plat, variantes, images, catégories) —
     toute clé ajoutée par erreur casse le test ; item soft-deleted et item
     `statut≠ACTIF` absents de la liste + 404 au détail.
   - `test/e2e-db/public-feed.db-e2e-spec.ts` (5 tests) : le feed n'expose que
     les établissements visibles, jamais un item supprimé ni celui d'un
     établissement masqué ; épuisé badgé `EPUISE` (jamais de stock chiffré) ;
     clés exactes de l'item + établissement imbriqué ; filtre par type étanche.
5. **Isolation tenant mouvements de stock**
   (`test/e2e-db/mouvement-stock-isolation.db-e2e-spec.ts`) : 2 tests verts qui
   FIGENT le constat de la faille + 2 tests `it.skip` décrivant le comportement
   attendu après correctif (voir bug ci-dessous).

## Bugs consignés (aucun corrigé — hors périmètre)
1. **BLOQUANT (sécurité, fuite cross-tenant)** —
   `src/supermarche/mouvement-produit-magasin-stock/mouvement-produit-magasin-stock.service.ts`
   (toutes les méthodes) + son contrôleur : `findAllBySupermarche`,
   `findAllByProduit`, `findOne`, `create` acceptent un `supermarcheId`
   ARBITRAIRE sans jamais le confronter au JWT (aucun `UserPayload`, aucun appel
   à `EstablishmentAccessService`). Routes `POST /supermarche/mouvement/liste`,
   `POST /produit`, `GET /:supermarcheId/:id`, `POST /` : tout staff d'un tenant
   B peut lire les mouvements (achats, quantités, fournisseurs) d'un tenant A et
   CRÉER des mouvements chez A. Prouvé par test vert « constat de la faille » ;
   correctif attendu décrit dans les 2 tests skippés (passer l'utilisateur au
   service + vérification d'appartenance). C'est le même patron déjà corrigé
   ailleurs (bug contrat whitelist du 2026-07-06) — module oublié.
2. **Mineur (robustesse de test, corrigé côté test)** — les tests de
   `public-catalog.db-e2e-spec.ts` faisaient des assertions de présence sur le
   listing public global paginé à 20 : avec la base e2e partagée qui grossit
   (nouvelles suites), résultat dépendant de l'ordre des suites (flaky observé
   1 run sur 2). Rendues déterministes via le filtre `q` (nom unique).
3. **Mineur (flaky environnemental, corrigé côté test)** —
   `image-migration.db-e2e-spec.ts` : le `beforeAll` (premier chargement du
   binaire natif sharp) dépassait parfois 60 s sur poste chargé → timeout de
   hook. Porté à 180 s.

## Fichiers touchés (aucun commit)
- Modifiés : `test/jest-e2e.json`, `test/e2e-db/client-auth.db-e2e-spec.ts`,
  `test/e2e-db/public-catalog.db-e2e-spec.ts`,
  `test/e2e-db/image-migration.db-e2e-spec.ts`
- Créés : `test/token-separation.e2e-spec.ts`,
  `test/e2e-db/commande-retrait-transitions.db-e2e-spec.ts`,
  `test/e2e-db/public-feed.db-e2e-spec.ts`,
  `test/e2e-db/public-exposure.db-e2e-spec.ts`,
  `test/e2e-db/mouvement-stock-isolation.db-e2e-spec.ts`

## Reste / suite
- Relecture + commit (attente propriétaire).
- Corriger le bug bloquant d'isolation des mouvements de stock, puis réactiver
  (et inverser le constat) les tests skippés de
  `mouvement-stock-isolation.db-e2e-spec.ts`.
