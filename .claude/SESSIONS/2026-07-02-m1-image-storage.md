# M1 (app client) — sous-chantier 1 : stockage d'images (backend)

Date : 2026-07-02
Repo : `all-in-one-backend` — **PR #58 MERGÉE sur main (`6de2bfb`)**.

## Objectif
Remplacer le stockage des images en **base64 en base** par un **vrai stockage de fichiers**,
derrière une interface abstraite basculable (dossier local aujourd'hui, stockage objet demain),
avec génération de miniatures. Flux de saisie commerçant inchangé (base64 en JSON).

## Décision d'archi (Étape A validée)
Interface `ImageStorageService` (abstraite) + 1re impl **dossier local** servi en statique.
Choix retenus : URL **absolue via `PUBLIC_MEDIA_BASE_URL`** ; rendus **thumb 200² + display 800px
webp** (pas d'original) ; service statique via **`useStaticAssets`** (0 dép) ; **`sharp`** pour
les miniatures ; migration = **fonction pure + commande standalone + spec e2e-BD**, `--dry-run`
par défaut.

## Réalisé (Étape B)
Module profond `src/common/image-storage/` (`@Global`, patron StockModule/AuditModule) :
- `image-storage.service.ts` : interface abstraite (`save` / `delete`).
- `local-image-storage.service.ts` : impl dossier local. Nommage
  `<root>/<TYPE>/<etabId>/<uuid>.webp` (+ `_thumb.webp`) → isolation Phase 4 sur disque, zéro
  collision. `delete` anti-traversal (refuse hors-racine), best-effort.
- `image-processing.service.ts` : `sharp` (rotate EXIF, display max 800 `inside`, thumb 200²
  `cover`, webp).
- `image-persistence.service.ts` : **point commun SM/resto** `persistImages`/`persistOne`
  (data URL → URL ; URL gérée conservée → idempotent). Zéro duplication.
- `image-storage.util.ts` : `parseDataUrl` / `isDataUrl` / `isManagedUrl`.
- `image-storage.module.ts` : binding `{ provide: ImageStorageService, useClass: Local… }`.
- Branché dans `AppModule` ; `main.ts` bascule en `NestExpressApplication` +
  `useStaticAssets('/media', index:false, cache immutable)`.

Bascule des créations (avant la transaction, I/O hors tx) :
- `produit-magasin.service` (SM) : `dto.images` + `variante.imageUrl` → `persist*`.
- `plat.service` (resto) : `dto.images` (le DTO variante resto ne porte pas d'`imageUrl`).

Migration (livrée, NON exécutée) :
- `src/common/image-storage/migration/migrate-images.ts` : fonction pure idempotente,
  **dry-run = zéro effet de bord** (aucune écriture fichier ni base), sauvegarde JSON en mode réel.
- `scripts/migrate-images.ts` : commande standalone (`NestFactory.createApplicationContext`),
  **dry-run par défaut**, `--confirm` pour exécuter.

Tests :
- Unitaires (jest) : util, `LocalImageStorageService` (save→fichiers+miniatures, isolation,
  delete, anti-traversal), `ImagePersistenceService` (idempotence). **Suite 241/241 verte.**
- e2e-BD (Docker) : `test/e2e-db/image-migration.db-e2e-spec.ts` — dry-run sans effet,
  conversion réelle → `/media/...` + fichiers, idempotence. **`pnpm test:e2e:db` = 17/17.**

Notes techniques :
- `esModuleInterop` off dans le repo → override **scopé ts-jest** (package.json + jest-e2e-db.json)
  pour l'import default de `sharp` ; le build runtime (SWC) gère déjà l'interop.
- `uploads/` et `scripts/backup/` ajoutés au `.gitignore`.
- Fichier mort repéré (hors périmètre) : `produit-magasin.service copy.txt`.

## En attente
- ✅ **Mergé** (PR #58 → `6de2bfb`).
- Migration à exécuter **séparément et après validation** (tester d'abord via l'e2e-BD, puis
  `--dry-run` sur copie, puis `--confirm`).
- Non-régression front : les data URLs déjà en base continuent de s'afficher pendant la
  transition (mix base64 + URLs). Vérif visuelle caisse à faire côté front.
