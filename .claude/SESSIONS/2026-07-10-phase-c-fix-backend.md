# Backend Phase C — Correction des bugs consignés en Phase B (2026-07-10)

Repo : `all-in-one-backend`. Branche : `test/coverage-predeploy`
(les tests Phase B y sont commités ; les correctifs Phase C sont
**non commités** — `git status` montré au propriétaire).
Périmètre STRICT : le bug bloquant d'isolation tenant + son jumeau restaurant,
le balayage des suspects analogues, et l'extension validée en cours de mission
(options dans la recherche de plats). Pas de refactor, pas de migration.

## Bug BLOQUANT — mouvements de stock supermarché sans confrontation au JWT

`src/supermarche/mouvement-produit-magasin-stock/` (routes
`/supermarche/mouvement` : POST /, POST /liste, POST /produit,
GET /:supermarcheId/:id) : `create`, `findAllBySupermarche`, `findAllByProduit`
et `findOne` acceptaient un `supermarcheId` arbitraire (body ou URL) sans
jamais le confronter à l'identité du JWT. Un staff d'un tenant B pouvait lire
les mouvements d'un tenant A (achats, ventes, fournisseurs, quantités) et
créer des mouvements chez lui.

Correctif (pattern maison, identique à `supermarche/inventaire`) :
1. Contrôleur : chaque handler prend `@Request() req: AuthenticatedRequest` ;
   le supermarché visé est résolu depuis `req.user.selectedEtablissementId`
   (helper `resolveSupermarcheId` : id présent + type SUPERMARCHE). Le
   `supermarcheId` envoyé par le client (body/URL) est **ignoré** — le JWT
   fait foi. Les routes et DTOs sont inchangés (compat contrat).
2. Service : signatures étendues (`supermarcheId` + `user: UserPayload`) et
   vérification systématique via le seam
   `EstablishmentAccessService.verifySupermarche(user, supermarcheId)` avant
   toute lecture/écriture. `create` épingle le mouvement au supermarché du JWT
   (`restaurantId` forcé à null).
3. `AccessModule` est `@Global` → aucun changement de module.

## Jumeau restaurant — même faille, corrigée pareil

`src/restaurant/mouvement-plat-stock/` (routes
`/restaurant/mouvement-plat-stock`) : même trou, aggravé —
`GET plat/:platId` et `GET variante/:varianteId` n'avaient AUCUN scoping
tenant, `create` acceptait un `restaurantId` arbitraire ET mutait le stock
d'une variante arbitraire (`variante.update` sans vérifier l'appartenance).

Correctif :
1. Contrôleur : même pattern (`resolveRestaurantId` depuis le JWT, routes
   inchangées, param/body ignorés).
2. Service : seam `verifyRestaurant` sur les 4 méthodes ; les historiques par
   plat/variante sont filtrés par `restaurantId` du JWT ; `create` exige que la
   variante appartienne à un plat DU restaurant du JWT (sinon 404, stock
   intact) et attribue l'auteur comme le seam StockMovement
   (`partenaireId` ou `utilisateurId` selon `userType` — l'ancien code écrivait
   `req.user?.sub || 'system'`, toujours 'system' en pratique).
3. Typage : `where: any` → `Prisma.MouvementStockWhereInput` dans
   `findByRestaurant`.

## Balayage des suspects (Phase A)

- `src/common/dto/create-mouvement-stock.dto.ts` : seuls consommateurs = le
  contrôleur/service mouvement supermarché corrigés ci-dessus. Les champs
  `supermarcheId`/`restaurantId` du DTO restent déclarés (whitelist
  `forbidNonWhitelisted`) mais sont **ignorés** côté serveur. → COUVERT.
- `src/configuration-etablissement/dto/*` : faille CONFIRMÉE et exploitable —
  les 8 routes (POST/PUT/GET/DELETE /, verifier-remise, multi-pos GET,
  PATCH remise, PATCH multi-pos) acceptaient un `supermarcheId`/`restaurantId`
  arbitraire ; un tenant B pouvait lire ET modifier la config POS d'un tenant A
  (remise max, multi-POS). CORRIGÉ : le service prend `user: UserPayload` et
  `verifierAccesEtablissement` (seam verifySupermarche/verifyRestaurant)
  remplace le simple test d'existence ; ajouté aussi sur `recuperer` et
  `supprimer` qui ne vérifiaient RIEN. Ici les ids restent acceptés en
  body/query mais sont **vérifiés** (le Hub les envoie —
  `use-establishment-settings.ts` — contrat préservé, aucun changement front
  requis).
- `GET /configuration-etablissement/liste` (« admin ») : CONSIGNÉ, non touché —
  liste les configs de TOUS les tenants à tout staff authentifié, sans guard
  admin ni paramètre d'établissement à vérifier. Fuite faible (remise/multiPOS
  + ids), mais l'intention (espace /admin plateforme ?) est à trancher avant de
  poser un guard.
- `src/supermarche/commande/dto/create-commande.dto.ts` : SAIN / non
  exploitable — `CreateCommandeDto` et `UpdateCommandeDto` ne sont référencés
  par AUCUN contrôleur ni service (DTO mort) ; le contrôleur commande n'expose
  pas de POST de création. Rien à faire.

## Extension validée en cours de mission — options dans la recherche de plats

`GET /restaurant/plat/recherche` (`findAll` de `plat.service.ts`) n'incluait
pas les `options` des plats (seul `getPlatById` le faisait) → le correctif POS
resto du Hub (envoi `options: [{optionId, quantite}]`) était inerte, faute de
catalogue avec options. Ajouté `options: { include: { option: true } }` à
l'include du findAll (même forme que `getPlatById`), rien d'autre dans le
module.

Effet de bord réglé : le nouveau spec compile `plat.service.ts` sous ts-jest
(`esModuleInterop: true`) alors que `tsc` du projet est sans interop —
`import * as namer from 'color-namer'` + appel direct ne compile pas sous
interop. Shim d'interop CJS local (`.default ?? module`, typé, sans `any`),
valide et fonctionnel sous les deux configs. `jest-e2e-db.json` inchangé.

NON traité (décision ouverte, consigné par le front) : remise par ligne resto
(`remise?` dans `LigneVentePlatDto` + calcul serveur) — parité fonctionnelle
avec le supermarché, pas un correctif de bug.

## Tests

- `test/e2e-db/mouvement-stock-isolation.db-e2e-spec.ts` : les 2 skips Phase B
  réactivés (staff B ne liste pas / ne crée pas chez A → Forbidden) ; les 2
  tests qui figeaient la faille inversés (rejet attendu) ; + couverture jumeau
  restaurant (lecture/écriture cross-tenant refusées, historiques scopés,
  variante d'un autre restaurant → 404 sans mutation de stock).
- NOUVEAU `test/e2e-db/configuration-etablissement-isolation.db-e2e-spec.ts` :
  lecture/modification/suppression/vérif-remise cross-tenant → Forbidden,
  config du tenant A intacte après tentatives.
- NOUVEAU `test/e2e-db/plat-recherche-options.db-e2e-spec.ts` : la recherche
  renvoie `options[].option` (forme getPlatById) ; plat sans option → `[]`.
- Front vérifié en lecture : AUCUN appel à `/supermarche/mouvement` ni
  `/restaurant/mouvement-plat-stock` dans Partner-Hub/client/livreur → aucun
  contrat front à adapter pour les mouvements.

## Suites (2 runs complets, résultats identiques, zéro flaky)

- `pnpm test` : 29 suites / **291 tests verts**, 0 skip (×2).
- `pnpm test:e2e` : 3 suites / **47 tests verts**, 0 skip (×2).
- `pnpm test:e2e:db` : 17 suites / **85 tests verts**, 0 skip (×2) — avant :
  16 suites / 81 verts + 2 skips. Conteneur jetable `aio-e2e-pg` uniquement,
  base dev `aio-db` non touchée.
- `npx tsc --noEmit` : 0 erreur. Lint : aucun nouveau problème introduit
  (le legacy `catch(error)`/imports inutilisés préexistants de
  `plat.service.ts` et `configuration-etablissement` reste tel quel).

## Fichiers applicatifs modifiés (aucun commit)

- `src/supermarche/mouvement-produit-magasin-stock/mouvement-produit-magasin-stock.{controller,service}.ts`
- `src/restaurant/mouvement-plat-stock/mouvement-plat-stock.{controller,service}.ts`
- `src/configuration-etablissement/configuration-etablissement.{controller,service}.ts`
- `src/restaurant/plat/plat.service.ts` (include options + shim color-namer)
- Specs : `mouvement-stock-isolation` (réécrit),
  `configuration-etablissement-isolation` (nouveau),
  `plat-recherche-options` (nouveau)
