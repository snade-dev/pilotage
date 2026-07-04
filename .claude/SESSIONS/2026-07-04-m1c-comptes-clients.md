# Session 2026-07-04 — M1-c : comptes clients (auth téléphone vérifiée) — BACKEND

Repo : `all-in-one-backend`. Branche : `feat/m1c-comptes-clients` (partant de
`feat/m1d-commandes-retrait`). **Non commitée** (attente relecture).
Design validé : `.claude/DESIGN/2026-07-04-m1c-comptes-clients.md`.

## Fait
- **Modèle** : `Client.email`/`motDePasse` rendus nullables (compte grand public
  par téléphone, sans email/mdp) + `telephoneVerifie` / `dateVerificationTelephone`.
  Nouveau `CodeVerificationTelephone` (codeHash bcrypt, expiration, essais, usage
  unique). `ClientFidelite.clientId` (lien optionnel). Migration additive
  `20260705120000_add_compte_client_telephone`.
- **Module auth client séparé** (`src/auth/client-auth/`) : routes `/v1/auth/client/*`
  (`demarrer`, `renvoyer-code`, `verifier` en `@Public` throttlées ; `profil`
  GET/PATCH en `@ClientOnly`). Flux téléphone-first unique (inscription = connexion),
  réponses neutres.
- **Session client** : réutilise l'infra JWT (`AuthService.creerSessionClient`,
  `userType: CLIENT`). Aucun second secret. `@CurrentClientId` inchangé → M1-d
  consomme le token tel quel.
- **`VerificationCodeSender`** : interface + `LogVerificationCodeSender` (dev = log)
  avec garde-fou `NODE_ENV=production` (throw au démarrage → fail-fast). Impl réelle
  SMS/WhatsApp branchable plus tard sous le même token.
- **Rattachement fidélité** (`FideliteLinkService`) : à la création de commande de
  retrait, relie/crée la fiche `ClientFidelite` de l'établissement par téléphone,
  sans écraser les champs commerçant ni dupliquer.
- **Anti-abus** : throttle HTTP par route ; cooldown 60 s/numéro + cap quotidien +
  essais bornés + code haché + expiration 5 min, appliqués côté service (BD).

## Tests
- **Unit : 291/291 verts** (`npx jest`), dont : génération/hachage/vérif de code
  (expiration, essais épuisés, code faux, normalisation) ; rattachement fidélité
  (crée / relie / idempotent / scope) ; séparation client/employé (`UserTypeGuard`).
- **e2e-BD** : 1er run **48/49** (flux nominal, réponses neutres, cooldown, verrou
  d'essais, fidélité ×2, garde-fou prod, + M1-d intact après ajout d'arg au service).
  L'unique échec était un **bug du test** (Reflector factice renvoyant les types pour
  toute clé, y compris `IS_PUBLIC_KEY` → route vue comme publique) — **corrigé** et
  reprouvé hors Docker par un spec unit dédié.

## Piège rencontré
- **Docker Desktop passé en lecture seule** en cours de session (overlay2
  « read-only file system ») : impossible de créer/supprimer un conteneur, donc
  `pnpm test:e2e:db` bloqué (conteneur `aio-e2e-pg` figé, non supprimable). Le
  conteneur de dev `aio-db` (5432) est resté intact. **À rejouer après redémarrage
  de Docker Desktop.**

## Reste
- Relecture + commit + déploiement migration.
- Impl réelle `VerificationCodeSender` (fournisseur SMS/WhatsApp à sourcer).
- Écrans app cliente (compte + commande/suivi).
