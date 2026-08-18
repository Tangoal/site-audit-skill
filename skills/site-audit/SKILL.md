---
name: site-audit
description: Audite un site public (TLS, headers de sécurité, performance) via testssl.sh, MDN HTTP Observatory et Lighthouse, et produit un rapport comparable dans le temps. À utiliser quand l'utilisateur demande d'auditer/tester/checker la sécurité ou la perf d'un site, ou mentionne testssl/Observatory/Lighthouse/PageSpeed pour un de ses domaines.
---

# /site-audit — Audit sécurité + perf d'un site public

Lance trois outils tiers contre un ou plusieurs domaines et synthétise le
résultat dans un tableau. Lecture seule côté site audité — requêtes
HTTPS/TLS normales, jamais d'écriture ni de modification.

| Outil | Mesure | Équivalent web connu |
|---|---|---|
| [`testssl.sh`](https://github.com/drwetter/testssl.sh) | Config TLS/certif, ciphers, vulnérabilités connues (Heartbleed, POODLE, ROBOT...) | ssllabs.com/ssltest |
| [`@mdn/mdn-http-observatory`](https://github.com/mdn/mdn-http-observatory) | Headers de sécurité HTTP (CSP, HSTS, X-Frame-Options...) | developer.mozilla.org/observatory |
| [Lighthouse](https://github.com/GoogleChrome/lighthouse) | Performance, accessibilité, best-practices, SEO | pagespeed.web.dev |

Ces trois sujets sont indépendants et complémentaires : un site peut avoir un
A+ TLS et un D sur les headers, ou l'inverse. Toujours faire les trois, jamais
en déduire un score global unique.

## Dépendances : détection et installation à la demande

Ce skill ne suppose rien d'installé à l'avance. Avant de lancer un audit,
vérifier/installer dans cet ordre :

**1. `git` et `node`/`npx`** — prérequis de base, s'ils manquent le signaler
à l'utilisateur et arrêter (ne pas tenter d'installer des paquets système de
base sans confirmation explicite).

**2. `testssl.sh`** — chercher dans l'ordre :
   - `testssl.sh` dans le `PATH` (`command -v testssl.sh`)
   - un clone existant dans `~/.cache/site-audit/testssl.sh/testssl.sh`
   - sinon cloner : `git clone --depth 1 https://github.com/drwetter/testssl.sh.git ~/.cache/site-audit/testssl.sh`

**3. `@mdn/mdn-http-observatory`** — toujours via `npx --yes @mdn/mdn-http-observatory <domaine>`,
   npx le télécharge/cache lui-même, rien à gérer.

**4. Chrome/Chromium pour Lighthouse** — chercher dans l'ordre, et
   **mémoriser le chemin trouvé** pour ne pas refaire la détection à chaque
   domaine dans une même session :
   - variable d'env `CHROME_PATH` si déjà définie
   - un binaire système : `google-chrome`, `chromium`, `chromium-browser` dans le `PATH`
   - un chromium déjà téléchargé par Playwright, si le projet en dépend :
     `find ~/.cache/ms-playwright -maxdepth 1 -iname "chromium-*" ! -iname "*headless_shell*"`
     puis `<ce dossier>/chrome-linux64/chrome` (Linux) ou l'équivalent macOS/Windows
   - sinon télécharger un Chrome dédié : `npx --yes @puppeteer/browsers install chrome@stable --path ~/.cache/site-audit/chrome`,
     puis utiliser le chemin affiché en sortie

Si une installation à la demande est nécessaire (clone testssl.sh, download
Chrome), prévenir l'utilisateur en une phrase avant de le faire — c'est un
téléchargement, pas juste une lecture, même si c'est un dépôt public standard
et sans risque.

## Déroulé, pour chaque domaine demandé

1. **TLS** :
   ```
   <chemin testssl.sh> --quiet --color 0 --fast <domaine> 2>&1 | grep -E "^\s*(TLS 1|Overall Grade|Heartbleed|POODLE|ROBOT|FREAK|DROWN|LOGJAM|BEAST)"
   ```
2. **Headers** :
   ```
   npx --yes @mdn/mdn-http-observatory <domaine>
   ```
   Parser le JSON retourné (`scan.grade`, `scan.score`, `tests.*.pass`) plutôt
   que de coller le brut — ne lister que les tests en échec avec leur
   `scoreModifier`.
3. **Perf** :
   ```
   CHROME_PATH=<chemin chromium> npx --yes lighthouse https://<domaine> \
     --output=json --output-path=<fichier temporaire> \
     --chrome-flags="--headless=new --no-sandbox" \
     --only-categories=performance,accessibility,best-practices,seo --quiet
   ```
   Lire les scores dans `categories.*.score` (× 100).

## Rapport

Un tableau markdown : une ligne par domaine, colonnes TLS / Headers / Perf.
Sous le tableau, lister concrètement les points en échec (headers manquants,
score perf bas) avec la piste de correction si évidente (ex: middleware
reverse-proxy pour ajouter CSP/HSTS/X-Frame-Options). Ne pas juste balancer
le JSON brut — l'utilisateur veut un diagnostic actionnable, pas une donnée.

Ne pas proposer de corriger automatiquement la config du site dans ce skill —
l'audit est en lecture seule, la correction est une tâche à part que
l'utilisateur doit valider explicitement.
