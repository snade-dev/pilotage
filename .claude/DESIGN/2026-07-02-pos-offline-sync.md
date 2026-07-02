# Design — Mode offline robuste du POS (Phase 6, jalon O0)

> **Statut : PROPOSITION.** Aucun code, aucune migration appliquée. Ce document fixe le
> modèle de synchronisation AVANT toute implémentation. Rien n'est décidé au-delà des
> décisions déjà arrêtées (voir plus bas) tant que le propriétaire n'a pas relu.
>
> Périmètre O0 : **supermarché d'abord** (le socle offline existant est côté supermarché).
> Le restaurant réutilisera le même modèle dans un jalon ultérieur.

---

## 0. Décisions déjà arrêtées (cadre, non rediscutées ici)

1. **Synchronisation par file d'opérations idempotentes.** UUID générés côté caisse ;
   rejeu à la reconnexion **sans double effet**.
2. **Conflit de stock = on accepte toutes les ventes + alerte d'écart.** Stock négatif
   **toléré à la réconciliation**, journalisé via le seam d'audit existant
   (`AuditJournalService.record` → table `historique`). **En ligne**, la vente sous zéro
   **peut rester interdite (mode strict)**.

Tout ce qui suit découle de ce cadre.

---

## 1. Contexte : ce qui existe déjà (à réutiliser, pas à refaire)

### Frontend (`All-in-One-Partner-Hub`)
- **IndexedDB** `all-in-one-pos` (`pos/lib/offline-pos-store.ts`), 3 object stores :
  `catalog` (snapshot catalogue par `supermarcheId`), `sales` (file des ventes en attente,
  clé = `localId` UUID), `clients`.
- **`QueuedSale`** : `{ localId, supermarcheId, sessionId?, payload, createdAt, status,
  lastError?, retries, lastAttemptAt? }`, `status ∈ {pending_sync, syncing, failed}`.
- **`useSalesSync`** : rejeu **séquentiel**, verrou synchrone anti-course (`syncLock`),
  backoff exponentiel (`syncBackoffMs`, plafond 60 s), plafond `MAX_SYNC_RETRIES = 5`,
  auto-sync au retour en ligne + retry périodique (20 s), interruption propre sur 401/403.

### Backend (`all-in-one-backend`)
- **Idempotence déjà en place** : `Commande.clientRequestId String? @unique`. Dans
  `PosSessionService.effectuerVente` : (a) court-circuit si une commande existe déjà pour
  ce `clientRequestId` (renvoie `construireReponseVenteIdempotente`), (b) rattrapage de la
  **course** via `catch P2002` → renvoie la commande gagnante. VenteDto porte déjà
  `clientRequestId?`.
- **Seam stock** : `StockMovementService.apply(tx, params)` — module profond détenant
  l'invariant `stock(défaut) = Σ stock(spécifiques)`. **Rejette `stockApres < 0`**
  (`INSUFFICIENT_STOCK`) et idem sur la variante par défaut.
- **Seam audit** : `AuditJournalService.record(tx, entry)` → `historique` (immuable,
  écriture seule), champs `entite / entiteId / action / ancienne/nouvelleValeur /
  etablissementId / etablissementType / acteur`.
- **Atomicité Phase 4** : la vente s'exécute dans `prisma.$transaction`, les deux seams
  s'exécutent DANS ce `tx` (réussite/échec en bloc).
- **Caisse = `PointDeVente`** ; `PosSession` est liée à `pointDeVenteId` + `supermarcheId`.

> **Ce que O0 change conceptuellement** : on généralise la file *ventes* en file
> *opérations* (pour préparer remboursement/clôture) et on ajoute au seam stock un **mode
> permissif** (négatif toléré + alerte) pour le **contexte de rejeu offline uniquement**.
> Le chemin **en ligne** reste strict.

---

## 2. Point 1 — Modèle d'opération offline

### 2.1 Structure d'une opération

Enveloppe commune, indépendante du type métier :

```ts
type OfflineOpType =
  | "VENTE"            // POS supermarché (O1/O2) — SEUL type offline en O0
  | "CLOTURE_SESSION"; // (ultérieur, hors O0)
  // NB: REMBOURSEMENT hors périmètre offline (décision 2026-07-02, cf. §8-Q2).
  //     Un remboursement exige d'être EN LIGNE.

interface OfflineOperation {
  opId: string;            // UUID v4 généré côté caisse — clé d'idempotence GLOBALE
  type: OfflineOpType;
  supermarcheId: string;   // établissement cible
  caisseId: string;        // = pointDeVenteId (identifie la caisse d'origine)
  sessionId?: string;      // PosSession locale/serveur si connue
  deviceId: string;        // identifiant stable de l'appareil (voir §6)
  createdAt: string;       // ISO — horodatage LOCAL de saisie (heure de vente réelle)
  seq: number;             // compteur monotone par caisse (ordre de rejeu, voir §3)
  payload: unknown;        // corps métier (ex. VenteDto SANS le montant de stock serveur)
  // --- état de sync (local, jamais envoyé) ---
  status: "pending_sync" | "syncing" | "synced" | "failed";
  retries: number;
  lastError?: string;
  lastAttemptAt?: string;
  serverId?: string;       // id serveur renvoyé après acceptation (ex. commandeId)
}
```

Notes :
- `opId` **est** la clé d'idempotence transmise au backend. Pour `VENTE`, on **mappe
  `opId → VenteDto.clientRequestId`** : on réutilise l'unicité déjà en base, zéro nouvelle
  infra pour le cas vente.
- `createdAt` porte l'**heure réelle de la vente** (offline), pas l'heure de rejeu — pour
  que les rapports/journal reflètent le moment de l'encaissement.
- `payload` **ne contient jamais** de valeur dérivée du stock serveur : le prix/TVA/promo
  sont figés au moment de la vente (déjà calculés côté client à partir du snapshot
  catalogue), le stock est décidé **au rejeu** par le serveur.

### 2.2 Garantir l'idempotence au rejeu

Trois couches, de la moins à la plus forte :

1. **Clé unique en base** (`clientRequestId @unique`, ré-utilisée pour `VENTE`). Le premier
   rejeu crée ; tout rejeu ultérieur retombe sur le court-circuit ou le `catch P2002`.
2. **Court-circuit applicatif** : `effectuerVente` renvoie la commande existante sans re-
   décrémenter le stock (déjà codé). => **le rejeu ne rejoue jamais le mouvement de stock**.
3. **Table d'idempotence générique** (proposée, pour les types SANS `@unique` métier —
   remboursement, clôture) : voir §3.4. Elle mémorise `opId → résultat` pour renvoyer la
   même réponse à un rejeu, même si l'entité créée n'a pas de clé unique naturelle.

> Invariant d'idempotence : **une opération d'`opId` donné produit au plus un effet
> métier, quel que soit le nombre de rejeus, sur n'importe quelle caisse.**

---

## 3. Points 2 & 3 — File locale + protocole de synchronisation

### 3.1 File locale côté caisse (Point 2)

- **Stockage** : IndexedDB (déjà en place). On **renomme conceptuellement** le store `sales`
  en store `operations` (clé `opId`), avec index `by_status`, `by_supermarche`,
  `by_caisse`. Migration IDB via bump `DB_VERSION` (3) + `onupgradeneeded` (copie
  `sales → operations`, `localId → opId`). Rétro-compat : garder l'API `enqueueSale` comme
  fine surcouche de `enqueueOperation` le temps de la transition.
- **Persistance** : IndexedDB survit au rechargement, à la coupure et au redémarrage de
  l'app (PWA/desktop). Aucune opération n'est perdue tant qu'elle n'est pas `synced` +
  supprimée.
- **Ordre** : rejeu **séquentiel par caisse**, trié par `seq` croissant (le socle actuel
  rejoue déjà en série). L'ordre inter-caisses n'a pas à être garanti (chaque caisse est
  indépendante ; le stock partagé est réconcilié côté serveur, cf. §4).
- **Reprise après coupure** : au montage du POS et à chaque retour `online`, on relit la
  file et on rejoue les opérations `pending_sync` + `failed` éligibles (backoff). Une
  opération `syncing` orpheline (app tuée en plein rejeu) est **re-tentée** sans risque
  grâce à l'idempotence serveur.

### 3.2 Endpoint(s) de rejeu (Point 3)

Deux options, **je recommande l'option A pour O1, l'option B pour O2** :

**Option A — Rejeu unitaire (réutilise l'existant).**
Chaque opération est POST individuellement sur l'endpoint métier déjà idempotent :
`POST /supermarche/pos-session/vente` avec `clientRequestId = opId`. Le socle `useSalesSync`
fait déjà exactement ça. **Coût nul, dispo immédiatement.** Limite : N requêtes = N aller-
retours réseau.

**Option B — Rejeu par lot (batch), pour O2.**
Nouvel endpoint dédié :

```
POST /supermarche/pos-session/sync-operations
Body: { operations: OfflineOperationWire[] }   // triées par (caisseId, seq)
```

Traitement : **une transaction par opération** (PAS une transaction géante — on veut des
échecs partiels indépendants), en réutilisant exactement le code de `effectuerVente` (donc
l'atomicité Phase 4 par vente est préservée). Réponse détaillée par opération :

```ts
interface SyncOperationResult {
  opId: string;
  outcome: "APPLIED" | "DUPLICATE" | "REJECTED";
  serverId?: string;          // commandeId créé (APPLIED) ou existant (DUPLICATE)
  stockNegatif?: boolean;     // true si le rejeu a fait passer une variante < 0
  error?: { code: string; message: string };
}
interface SyncOperationsResponse { results: SyncOperationResult[]; }
```

- `APPLIED` → l'opération a créé l'effet métier ; le client passe l'op à `synced` et la
  supprime.
- `DUPLICATE` → déjà appliquée (idempotence) ; **même traitement client** que `APPLIED`
  (on nettoie), sans nouvel effet.
- `REJECTED` → refus **définitif et légitime** (ex. session fermée côté serveur, moyen de
  paiement supprimé, variante supprimée). Le client **ne re-tente pas** : passage en état
  terminal `failed` + remontée à l'opérateur (voir §7 risques). À distinguer d'une erreur
  **transitoire** (réseau/500) qui, elle, reste `pending_sync` et sera re-tentée.

### 3.3 Gestion des échecs partiels

- Le lot n'est **jamais** « tout ou rien ». Chaque `SyncOperationResult` est traité
  isolément côté client (le socle sait déjà marquer une vente `failed` et continuer).
- Classification d'erreur (à centraliser) :
  - **transitoire** (réseau perdu, 5xx, 429) → rester en file, backoff, re-tenter.
  - **auth** (401/403) → interrompre le lot, ne pas consommer les retries, reprendre après
    reconnexion (comportement actuel de `useSalesSync` conservé).
  - **définitive** (4xx métier hors 401/403) → état terminal, escalade opérateur.
- **Plafond de retries** (`MAX_SYNC_RETRIES`) conservé : au-delà, l'op sort de l'auto-sync
  et exige une action manuelle (déjà le cas).

### 3.4 Idempotence côté serveur pour les types sans clé unique naturelle

**Décision 2026-07-02 : le remboursement offline est INTERDIT.** Comme la `VENTE` est le
seul type d'opération offline en O0, et qu'elle s'appuie déjà sur `Commande.clientRequestId
@unique`, **aucune infrastructure d'idempotence supplémentaire n'est nécessaire pour O0/O2**.
Rien à ajouter côté schéma.

> Reporté (hors O0) : si un jour `CLOTURE_SESSION` (ou un autre type sans clé unique
> naturelle) devait partir offline, on introduirait une table d'idempotence générique
> `OperationIdempotente { opId @id, type, etablissementId, resultatJson, entiteId,
> createdAt }`, écrite dans la transaction de l'op, renvoyant `resultatJson` au rejeu.
> **Non requis tant que seule la vente est offline.**

---

## 4. Point 4 — Résolution du stock (cœur du sujet)

### 4.1 Application des mouvements au rejeu

Le rejeu d'une `VENTE` réutilise `effectuerVente` dans la transaction Phase 4. Rien ne change
pour le calcul prix/TVA/promo : ils sont figés dans le payload.

> **Découverte de code (2026-07-02) — à intégrer en O2b.** Le POS **supermarché** n'applique
> PAS le stock de la vente via `StockMovementService.apply`. `effectuerVente` fait :
> 1. un **pré-check applicatif** (`pos-session.service.ts:1730` : `if (variante.stockActuel <
>    quantite) throw INSUFFICIENT_STOCK`), puis
> 2. un **`decrement` atomique inline** de la variante ET de la variante par défaut
>    (`~2141` / `~2193`).
>
> Conséquences :
> - Le param `negativeStockPolicy` **du seam ne suffit pas** pour le rejeu de la vente SM :
>   O2b devra **neutraliser le pré-check (1730)** et **lire `stockApres` après décrément** pour
>   détecter le négatif, DANS le contexte de rejeu uniquement.
> - Bonne nouvelle : le décrément inline est **déjà atomique** (`{ decrement: q }`), donc **pas
>   de lost-update** sur le chemin de vente SM (le durcissement atomique évoqué en §8-Q3 ne
>   concerne que les autres appelants du seam : ajustements, remboursements, réceptions).
> - Le seam et son `TOLERATE_NEGATIVE` (livrés en O2a) restent utiles pour les **autres**
>   mouvements et pour le chemin restaurant, qui, lui, passe par le seam.

### 4.2 Détection du passage négatif + alerte d'écart

Le seam `StockMovementService.apply` **rejette aujourd'hui** `stockApres < 0`. Proposition :
lui ajouter un **mode** explicite, passé par l'appelant, JAMAIS déduit implicitement.

```ts
// PROPOSITION — extension de ApplyStockMovementParams
interface ApplyStockMovementParams {
  // ...existant...
  /** Politique de stock négatif.
   *  - "STRICT" (défaut)  : throw INSUFFICIENT_STOCK  → chemin EN LIGNE inchangé.
   *  - "TOLERATE_NEGATIVE": autorise stockApres < 0, ne throw pas, signale l'écart. */
  negativeStockPolicy?: "STRICT" | "TOLERATE_NEGATIVE";
}

interface ApplyStockMovementResult {
  varianteAvant: number;
  stockApres: number;         // peut être < 0 en mode TOLERATE_NEGATIVE
  mouvement: MouvementStock;
  ecartNegatif?: number;      // = |stockApres| si stockApres < 0, sinon absent
}
```

- **Défaut = STRICT** : tout le code en ligne existant conserve son comportement
  (rejet sous zéro). Aucune régression.
- Le **rejeu offline** passe `TOLERATE_NEGATIVE`. Quand `stockApres < 0` (ou stock défaut
  < 0), le seam **applique quand même** le mouvement et **retourne `ecartNegatif`**.
- L'appelant (chemin de rejeu) déclenche alors l'**alerte d'écart** via
  `AuditJournalService.record(tx, …)` DANS la même transaction :

```ts
await audit.record(tx, {
  entite: "Variante",
  entiteId: varianteId,
  action: "ECART_STOCK_OFFLINE",       // nouveau code d'action
  etablissementId: supermarcheId,
  etablissementType: "SUPERMARCHE",
  ancienneValeur: { stockAvant: res.varianteAvant },
  nouvelleValeur: {
    stockApres: res.stockApres,        // négatif
    ecart: res.ecartNegatif,
    opId, caisseId, commandeId,
  },
  ...audit.acteurFromUser(user),
});
```

=> L'écart est **tracé, attribuable à une caisse et à une vente**, et immuable. Le tableau
`/owner/audit-log` (déjà branché sur `historique`) peut lister ces écarts sans nouvelle
plomberie backend d'affichage.

### 4.3 Cas multi-caisses simultanées (détaillé)

Scénario : caisses C1 et C2, hors-ligne toutes deux, stock serveur d'une variante = 5.
- C1 vend 4 (offline) → file C1. C2 vend 3 (offline) → file C2.
- Retour en ligne (ordre quelconque) :
  - 1er rejeu (ex. C1, 4) : `5 → 1`. OK, pas d'écart.
  - 2e rejeu (C2, 3) : `1 → -2`. `TOLERATE_NEGATIVE` : mouvement appliqué, stock = **-2**,
    `ECART_STOCK_OFFLINE(ecart=2, caisseId=C2)` journalisé.
- **Les deux ventes sont conservées** (décision arrêtée). Le stock négatif matérialise le
  survente réel ; l'opérateur régularise par un inventaire/ajustement (mouvement
  `AJUSTEMENT_*` existant), qui laissera lui aussi sa trace.
- **Aucune collision d'écriture** : chaque rejeu est sa propre transaction Prisma ;
  l'invariant `stock(défaut) = Σ` est maintenu par le seam (le delta est répercuté sur la
  variante par défaut, même en négatif). Les décréments concurrents sont sérialisés par la
  base (transactions), pas par l'ordre des caisses.
- **Idempotence croisée** : deux caisses ne peuvent pas produire le même `opId` (UUID v4 +
  `deviceId`, cf. §5), donc pas d'écrasement mutuel ni de dédoublonnage accidentel entre
  caisses.

> **Tranché (§8-Q3)** : le seam fait aujourd'hui un `read-then-write` applicatif (lost-update
> possible). Décision → écritures **atomiques** : `increment` en mode `TOLERATE_NEGATIVE`, et
> `updateMany` **conditionnel** (`where stockActuel gte -delta` + `increment`, throw si
> `count===0`) en mode `STRICT`. Ni `SELECT FOR UPDATE` ni verrou applicatif : la base
> sérialise les deltas. Corrige aussi un lost-update latent du chemin en ligne actuel.

---

## 5. Point 5 — Identifiants (UUID caisse vs IDs serveur)

- **Toute entité créée offline est identifiée par un UUID généré côté caisse** (`opId`), qui
  sert de clé d'idempotence. Les **IDs serveur** (cuid/uuid des `Commande`, `MouvementStock`)
  restent **autoritaires** et sont générés **au rejeu** ; le client les récupère via
  `serverId` et met à jour sa file.
- **Pas de génération d'ID métier serveur côté caisse** : la caisse ne fabrique jamais un
  `commandeId`. Elle ne fabrique que des `opId` (espace de noms distinct, jamais confondu
  avec un id serveur).
- **Anti-collision inter-caisses** : `opId = UUID v4` (2^122 d'entropie) — collision
  négligeable. On ajoute `deviceId` (identifiant stable stocké en IndexedDB, généré au 1er
  lancement) pour la **traçabilité** (savoir quelle caisse a produit l'op) et comme garde-
  fou de débogage, pas comme mécanisme d'unicité.
- **Anti-écrasement** : aucune opération ne fait d'`UPDATE` aveugle sur une entité d'une
  autre caisse. Les ventes ne font que des **créations** (Commande) + **deltas** de stock.
  Deux caisses touchant la même variante appliquent deux deltas additifs, jamais un « set ».
- **Numéro de reçu / ticket** : s'il existe une numérotation séquentielle serveur (à
  vérifier), elle est attribuée **au rejeu** (pas offline), pour éviter les trous/doublons
  entre caisses. À confirmer (§8-Q4).

---

## 6. Point 6 — Impact sur l'existant (ce qui bouge / ce qui NE bouge PAS)

### Ce qui change (proposé)
| Zone | Changement | Ampleur |
|---|---|---|
| Front `offline-pos-store.ts` | Store `sales` → `operations` (clé `opId`), `deviceId`, `seq`, `type` ; bump `DB_VERSION` + migration IDB | Moyen |
| Front `useSalesSync` | Généralisation « ventes » → « opérations » ; classification d'erreur transitoire/définitive | Moyen |
| Back `StockMovementService.apply` | Param `negativeStockPolicy` (défaut STRICT) + `ecartNegatif` en retour | **Petit, ciblé** |
| Back chemin de rejeu vente | Passe `TOLERATE_NEGATIVE` + `record(ECART_STOCK_OFFLINE)` sur écart | Petit |
| Back (O2) endpoint `sync-operations` (batch) | Nouveau contrôleur, réutilise `effectuerVente` | Moyen |
| Prisma | **Aucune table nouvelle en O0** — remboursement offline interdit ⇒ `clientRequestId @unique` suffit | Nul |

### Ce qui NE DOIT PAS bouger (garde-fous)
- **Atomicité Phase 4** : chaque vente/rejeu reste dans SA `prisma.$transaction`. Pas de
  transaction géante multi-ventes.
- **Isolation multi-tenant** : le scope `supermarcheId` vient toujours du JWT + garde
  d'accès existante. Le rejeu ne fait pas confiance au `supermarcheId` du payload sans
  contrôle d'appartenance. (Rappel Phase 4 : ne pas s'appuyer sur un `where` de relation
  to-one dans un `include` — Prisma 5.7 l'ignore silencieusement.)
- **Chemin de vente EN LIGNE** : reste **STRICT** (rejet sous zéro). Le mode permissif est
  strictement réservé au contexte de rejeu offline.
- **Invariant de stock** `stock(défaut) = Σ spécifiques` : toujours maintenu par le seam,
  y compris en négatif.
- **Journal immuable** : on **ajoute** des lignes (`ECART_STOCK_OFFLINE`), on ne modifie
  jamais l'historique.
- **`clientRequestId @unique`** : conservé tel quel ; on ne le remplace pas, on le nourrit
  avec `opId`.

---

## 7. Point 7 — Découpage en jalons d'implémentation proposé

### O1 — File locale généralisée + rejeu simple (unitaire)
- Front : migrer `sales → operations` (opId/deviceId/seq/type), conserver le rejeu série.
- Back : **aucun changement backend** — on rejoue sur `POST …/vente` avec
  `clientRequestId = opId` (déjà idempotent).
- **Livrable** : plusieurs ventes offline d'une même caisse rejouées sans doublon.
- **Risques** : stock encore STRICT → une vente offline qui ferait passer sous zéro serait
  **rejetée au rejeu** (perte de vente). Acceptable en O1 mono-caisse « stock large », mais
  **ne couvre pas encore la décision “accepter toutes les ventes”**. À signaler à
  l'opérateur. C'est précisément ce que O2 corrige.

### O2 — Sync robuste : idempotence complète + tolérance négative + écarts
- **O2a — FAIT (backend, branche `feature/pos-offline-o2-stock-seam`)** : `negativeStockPolicy`
  (`STRICT` défaut / `TOLERATE_NEGATIVE`) + `ecartNegatif` sur `StockMovementService.apply` ;
  STRICT inchangé (219 unit verts) ; 3 tests TOLERATE ajoutés.
- **O2b — FAIT (back `feature/pos-offline-o2-stock-seam` 01b937b ; front `feature/pos-offline-o1`
  b5d8001)** :
  - Back : `effectuerVente({ offlineReplay })` neutralise le pré-check 1730 + `validateVente({
    skipStockCheck })`, applique la vente sous zéro et journalise `ECART_STOCK_OFFLINE`
    (variante spécifique + par défaut) dans la transaction ; renvoie `stockNegatif`. Nouvel
    endpoint **`POST /supermarche/pos-session/sync-operations`** (rejeu par lot, un résultat
    par op : `APPLIED`/`REJECTED`/`RETRY` via `classifyReplayError`). `/vente` en ligne reste
    strict (mode tolérant NON exposé). Pas de table d'idempotence.
  - Front : `syncOfflineOperations` + la file de sync passe par `sync-operations` (au lieu de
    `/vente`) et interprète le résultat (APPLIED → supprimée ; REJECTED/RETRY → échec/retry).
  - Tests : back 226/226 unit + `replay-error.spec` ; front 28/28 ; tsc propre des 2 côtés.
- **O2b — reste** : preuve d'intégration **e2e-BD (Docker)** — double effet (rejeu ×2 = 1
  commande), écart tracé, échec partiel. Non faite ici (Docker indisponible). Affichage
  opérateur des écarts (écran dédié) : à câbler (les écarts sont déjà dans `historique`).
- **Risque résiduel** : `REJECTED` n'est pas encore terminal côté file (retombe dans le
  retry/backoff plafonné) — acceptable, à affiner en O3.

### O3 — Multi-caisses simultanées durci
- Seam stock déjà atomique (fait en O2) → ici on **prouve** la concurrence : tests de charge
  multi-caisses (pas de lost update), tableau d'écarts par caisse, régularisation guidée
  (ajustement d'inventaire).
- **Livrable** : scénario 2+ caisses offline sur stock partagé prouvé par e2e.
- **Risques** : conditions de course subtiles ; d'où O3 séparé, avec tests concurrents.

> Ordre imposé par la décision « accepter toutes les ventes » : **O2 est le vrai cœur**.
> O1 est un socle sûr mais incomplet vis-à-vis de la décision ; ne pas s'arrêter à O1.

---

## 8. Point 8 — Décisions retenues (choix par défaut « le plus intelligent », 2026-07-02)

Toutes les questions ouvertes ont été tranchées avec le choix le plus sûr/simple. Elles font
foi pour O1→O3 sauf objection du propriétaire.

1. **Périmètre offline O0 = supermarché seul.** Restaurant plus tard (le socle IDB est côté
   supermarché ; ne pas éparpiller avant d'avoir prouvé le modèle sur une verticale).
2. **Remboursement offline INTERDIT.** Exige d'être en ligne. ⇒ `VENTE` = seul type offline,
   pas de table d'idempotence à créer. **Front** : griser le remboursement quand `isOffline`
   + message clair.
3. **Delta de stock atomique — CONSTAT + décision.** Le seam fait aujourd'hui un
   **read-then-write** (`findUnique` → calcul → `update` d'une valeur absolue), donc
   *lost-update* possible entre deux rejeus/ventes concurrents. **Décision** :
   - **Rejeu offline (`TOLERATE_NEGATIVE`)** : passer à l'**`increment` atomique** Prisma —
     `update({ data: { stockActuel: { increment: delta } } })` — puis relire `stockApres`
     pour détecter le négatif. La base sérialise les deltas → aucun *lost update*.
   - **Chemin en ligne (`STRICT`)** : garder le rejet sous zéro **sans** lost update, via un
     **update conditionnel atomique** — `updateMany({ where: { id, stockActuel: { gte:
     -delta } }, data: { stockActuel: { increment: delta } } })` ; si `count === 0`, throw
     `INSUFFICIENT_STOCK`. (Remplace le `findUnique`+`update` actuel.)
   - Réalisé en **O2** (le mode négatif l'impose déjà), consolidé/prouvé en **O3**.
4. **Numérotation des reçus — SANS OBJET.** `Commande.id` est un **UUID** (aucune séquence
   numérotée par établissement). Rien à réserver ni à attribuer au rejeu. (Si une
   numérotation lisible est ajoutée un jour, elle sera attribuée **au rejeu**, jamais offline.)
5. **Rejeu unitaire (O1) puis batch (O2).** On démarre unitaire — **zéro changement
   backend**, rejeu sur `POST …/vente` avec `clientRequestId = opId`. Le batch
   `sync-operations` n'arrive qu'en O2, quand il apporte un gain réel.
6. **Pas d'expiration temporelle des opérations.** Une op reste re-tentable tant que l'échec
   est **transitoire** (réseau/5xx). Elle ne devient terminale (`REJECTED` + escalade
   opérateur) que sur cause **métier définitive** : session serveur clôturée, variante/moyen
   de paiement supprimé, appartenance invalide. Pas de perte silencieuse par « péremption ».
7. **Session de caisse : ouverte EN LIGNE avant bascule.** On **n'ouvre pas** de `PosSession`
   hors-ligne et on **ne clôture pas** hors-ligne en O0. Offline = uniquement des `VENTE` sur
   une session déjà ouverte. Si la session a été clôturée côté serveur entre-temps, le rejeu
   renvoie `REJECTED` (§3.2) → escalade. (`CLOTURE_SESSION` reste hors O0.)
8. **Réalignement catalogue + écran d'écarts.** Après sync : recharge du catalogue
   (`onSynced`, déjà là) **et** affichage explicite des variantes passées en **négatif**
   (issues des `ECART_STOCK_OFFLINE`) pour régularisation immédiate par ajustement
   d'inventaire. Ne pas se contenter du rechargement silencieux.

> Aucune de ces décisions n'ouvre de nouvelle table en O0 (cf. §6). La seule vraie évolution
> backend est la **réécriture du seam stock en écritures atomiques** (point 3), qui bénéficie
> aussi au chemin en ligne (correction d'un lost-update latent).

---

## Annexe — Emplacement du livrable
Ce document vit dans le repo **pilotage** (`.claude/DESIGN/`) car la conception est
**inter-repos** (front + back) et de niveau décision. À la validation, une entrée résumée
sera ajoutée à `DECISIONS.md` (section « POS offline ») et l'implémentation suivra les
jalons O1→O3 ci-dessus.

---

## 11. Référence visuelle (maquette Claude Design)

**Direction visuelle retenue : « 1b — Frais épuré »** (vert oklch ~0.55 0.11 155, encre vert
foncé #16211c, fond blanc froid #fbfcfb, police Hanken Grotesk, listes denses). Écartées : 1a
(Marché chaleureux, ocre) et 1c (Bold fintech, sombre).

Fichier maquette : `App Commande M1` (Claude Design). -https://claude.ai/design/p/036a8544-22b4-4646-a72e-969ff18bd6c7?file=App+Commande+M1.dc.html&via=share.

### Écrans validés par la maquette (chemin nominal)
- **2a Magasins** — liste des magasins proches, entrée sans compte, statut « ouvert / retrait aujourd'hui ».
- **2b Catalogue** — liste dense, catégories, stock « Disponible / Épuisé » (jamais de chiffre), barre panier persistante.
- **2c Fiche produit** — photo, prix, « Disponible en magasin », mention retrait comptoir + paiement à la remise.
- **2d Panier** — récap, total à payer, mention « paiement au comptoir », teaser M2 (Wave/Orange Money).
- **2e Compte** — identifiant téléphone (+221), « compte créé et relié à la fidélité, aucun email requis ».
- **2f Vérification** — code SMS **ou WhatsApp**, renvoi minuté.
- **2g Créneau** — choix jour + heure de retrait, rappel magasin.
- **2h Suivi** — timeline reçue → en préparation → prête → retirée & payée, **code de retrait**.
- **2i Historique** — commandes multi-magasins, fidélité reliée.
- **2j Commerçant** — kanban de réception (Reçues → En préparation → Prêtes), note « le retrait décrémente le stock et alimente les stats (Phase 4) ».

### Décisions actées par la maquette (à respecter à l'implémentation)
- Stock affiché en **Disponible / Épuisé** uniquement (pas de quantité exacte).
- **Compte client global**, non lié à un magasin : c'est la *commande* qui porte l'établissement
  (l'historique montre des commandes chez plusieurs magasins).
- **Décrément du stock au RETRAIT** (pas à la commande) → cohérent avec la logique d'écart.
- Vérification téléphone par **SMS ou WhatsApp** (fournisseur externe à sourcer).

### Écrans d'EXCEPTION à concevoir avant implémentation (absents de la maquette)
- **Article devenu épuisé entre commande et préparation** → commande partielle / substitution /
  annulation de ligne (pendant « commande » de la survente offline).
- **Refus / annulation d'une commande** par le commerçant (le kanban n'a pas d'action « Refuser »).
- **Panier vide**, **échec d'envoi du code SMS**, **créneau expiré sans retrait**.
Ces cas déterminent la cohérence du stock hors chemin nominal — à traiter avec le même soin
que les écarts offline.