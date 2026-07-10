# Front Hub Phase C — Correction des 3 bugs consignés en Phase B (2026-07-10)

Repo : `All-in-One-Partner-Hub`. Branche : `test/coverage-predeploy`
(les tests Phase B y sont commités ; les correctifs Phase C sont
**non commités** — `git status` montré au propriétaire).
Périmètre STRICT : les 3 correctifs ci-dessous + leurs tests. Pas de refactor,
pas d'extraction des pages POS, backend NON modifié (contrats vérifiés en
lecture dans `all-in-one-backend`).

## Bug 1 — BLOQUANT (argent) : POS resto n'envoyait ni options ni remises de ligne

`src/app/(app)/pos-restaurant/components/table-grid-tab.tsx`.

Contrat backend vérifié (`src/restaurant/pos-session/dto/vente/ligne-vente-plat.dto.ts`) :
- `LigneVentePlatDto = { varianteId, quantite, options?: [{ optionId, quantite }] }`.
  Les options sont identifiées par **l'ID BD de l'entité `Option`** (plate :
  `{id, nom, prix}`, liée au plat via `OptionGroupe`) ; le serveur facture
  `option.prix × option.quantite × quantite du plat`.
- **AUCUNE remise par ligne** dans le DTO resto (contrairement au supermarché
  `LigneVenteDto.remise`) → voir « Manques backend » plus bas.
- Remise globale : le serveur resto applique `calculerRemise(totalTTC, remise)`
  **sur le TTC** (étape 6 de `vente-pos-restaurant.service.ts`) — le
  supermarché, lui, l'applique sur le HT avant TVA.

Correctifs front :
1. `types/pos.ts` : `ProductOptionChoice.id?` (ID BD de l'option).
2. `mapPlatsToProducts` : mappe `plat.options` (associations OptionGroupe) en
   groupe unique « Suppléments » à choix multiples, id BD conservé.
3. Payload de vente : chaque ligne plat porte
   `options: [{ optionId, quantite: 1 }]` construit depuis `selectedOptions`.
   Le payload est **commun aux branches online et offline** (même objet
   `venteData` mis en file) → le rejeu ne perd rien.
4. Option sans id BD dans le panier → paiement REFUSÉ (toast explicite),
   aucune vente envoyée (pas d'écart d'encaissement possible).
5. Remise par article → REFUSÉE à la saisie (toast « utilisez la remise
   globale ») + garde-fou au paiement. Le DTO resto ne sait pas la transporter.
6. Alignement du total UI sur le recalcul serveur : TVA calculée sur le HT,
   puis remise globale appliquée sur le TTC (avant : remise fixe appliquée
   avant TVA → total UI < total serveur de `remise × taux` → rejet
   INSUFFICIENT_PAYMENT).

## Bug 2 — MAJEUR : TVA resto codée en dur

Même fichier, l.107 : `[18, 10, 5.5, 0]` en dur, pas de `tauxTVAId` envoyé.

Contrat backend vérifié :
- `GET /restaurant/taux-tva` existe (`src/restaurant/taux-tva/taux-tva.controller.ts`),
  déjà consommé par le front via `use-tva-rates` / `restaurant-tva.ts`
  (`useTvaRates("RESTAURANT")`).
- `CreateVenteDto.tauxTVAId?: string | null` accepté par la vente resto ; sans
  lui le serveur résout un taux « standard » qui peut différer de l'affiché.

Correctif : pattern supermarché reproduit — taux BD via
`useTvaRates("RESTAURANT")` + `selectActiveTvaRates`, sélection par valeur avec
suivi de l'ID (`vatRateId`), taux par défaut BD auto-sélectionné, payload
`tauxTVAId: isVatApplied ? vatRateId : undefined` (online **et** offline, même
objet). Plus aucun taux codé en dur.

## Bug 3 — MAJEUR : fallback `ligneCommandeId: item.lineId || saleId`

`src/app/(app)/pos/hooks/useRefunds.ts` l.73 : quand `lineId` manquait, l'ID de
COMMANDE partait comme ID de LIGNE (`LigneRemboursementDto.ligneCommandeId`).

Cause racine remontée : `lineId` n'était renseigné que par le rechargement de
l'historique (`page.tsx` → `loadHistory`, `lignes[].id`). Une vente en ligne
fraîchement encaissée stockait `items: [...cartItems]` **sans** `lineId` → tout
remboursement immédiat déclenchait le fallback fautif.

Correctifs :
1. `pos/page.tsx` : après `processVente`, les `lineId` sont mappés depuis la
   réponse serveur (`venteResponse.lignes[]`, id par `varianteId`) → une vente
   fraîche est remboursable immédiatement, sans repasser par l'historique.
2. `useRefunds.ts` : fallback supprimé — si `lineId` manque, ERREUR explicite
   (« identifiant de ligne de commande introuvable… rechargez l'historique »),
   aucun appel API, vente locale intacte. Jamais d'id de commande envoyé.
3. `use-pos-session.ts` : `VenteResponse.lignes[].id` typé `string` (était `any`).

Cas restant couvert par l'erreur explicite : les ventes offline
(`pending_sync`) n'ont pas de lignes serveur → remboursement refusé proprement
tant que la vente n'est pas synchronisée (comportement voulu).

## Manques backend consignés (backend NON touché)

1. **Remise par ligne resto** : ajouter `remise?: { type: POURCENTAGE |
   MONTANT_FIXE; valeur }` à `LigneVentePlatDto` + prise en compte dans
   `calculerMontantsAvecOptions` (parité avec le supermarché). Tant que ce
   champ n'existe pas, le POS resto refuse les remises par article.
2. **Options absentes de `/restaurant/plat/recherche`** : le `findAll` de
   `plat.service.ts` n'inclut pas `options` (seul `getPlatById` le fait). Le
   POS resto charge son catalogue par cette route → les options d'un plat ne
   peuvent pas être proposées à la vente aujourd'hui. Ajouter
   `options: { include: { option: true } }` à l'include du findAll ; le front
   les mappera automatiquement (mapping déjà en place).

## Tests

- 3 skips Phase B réactivés et verts : `table-grid-tab.spec.tsx` ×2 (payload
  options `{optionId, quantite}` + `tauxTVAId` BD), `useRefunds.spec.tsx` ×1
  (refus explicite sans `lineId`, aucun appel API).
- Assertions ajustées : remise de ligne → refus (subTotal inchangé, toast
  destructif) ; remise globale fixe appliquée sur le TTC (3500 → 4130 − 150 =
  3980) ; mock BD des taux TVA dans le spec resto.
- Nouveaux tests : option sans id BD → paiement refusé sans appel serveur ;
  branche offline → payload en file porte `tauxTVAId` et les options.
- **Suite complète 2×** : `24 fichiers / 193 tests verts, 0 skip` (avant :
  188 verts + 3 skips), résultats identiques, zéro flaky.
- `pnpm typecheck` : **0 erreur**. `pnpm build` : **OK**.

## Fichiers applicatifs modifiés

- `src/types/pos.ts` (id optionnel sur `ProductOptionChoice`)
- `src/app/(app)/pos-restaurant/components/table-grid-tab.tsx` (bugs 1 + 2)
- `src/app/(app)/pos/hooks/useRefunds.ts` (bug 3)
- `src/app/(app)/pos/page.tsx` (bug 3 — cause racine, `lineId` depuis la
  réponse de vente)
- `src/hooks/use-pos-session.ts` (typage `lignes[].id`)
- Specs : `table-grid-tab.spec.tsx`, `useRefunds.spec.tsx`
