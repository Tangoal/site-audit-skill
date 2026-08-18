---
name: site-audit
description: Audite un site public (TLS, HTTP/2-3, headers de sécurité, SPF/DKIM/DMARC, performance) via testssl.sh, MDN HTTP Observatory et Lighthouse, compare apex et www, et produit un rapport comparable dans le temps. À utiliser quand l'utilisateur demande d'auditer/tester/checker la sécurité ou la perf d'un site, ou mentionne testssl/Observatory/Lighthouse/PageSpeed pour un de ses domaines.
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

## Déroulé

### 0. Apex vs www

Avant de lancer le trio d'outils, résoudre les deux formes du domaine
demandé (`exemple.fr` et `www.exemple.fr`) :

```
curl -sI https://exemple.fr/ | head -5
curl -sI https://www.exemple.fr/ | head -5
```

- Si une forme redirige vers l'autre (cas le plus fréquent), lancer le trio
  complet uniquement sur la forme **canonique** (celle qui répond 200), et se
  contenter de vérifier que la forme non-canonique redirige bien en HTTPS et
  que les headers de sécurité (HSTS notamment) sont déjà présents sur la
  réponse de redirection elle-même — sinon un visiteur qui tape l'apex sans
  jamais suivre la redirection (proxy, cache, bot) ne les reçoit pas.
- Si les deux formes répondent 200 indépendamment (pas de redirection), les
  traiter comme deux domaines distincts et lancer le trio complet sur
  chacune.

Pour chaque domaine canonique retenu :

1. **TLS + protocole** :
   ```
   <chemin testssl.sh> --quiet --color 0 --fast <domaine> 2>&1 | grep -E "^\s*(TLS 1|ALPN|HTTP2|Overall Grade|Heartbleed|POODLE|ROBOT|FREAK|DROWN|LOGJAM|BEAST)"
   ```
   Le grep élargi (`ALPN`, `HTTP2`) fait remonter le support HTTP/2 (h2) —
   pas besoin d'outil séparé. testssl.sh ne teste pas HTTP/3/QUIC (UDP) ; le
   header `alt-svc: h3=...` vu dans la réponse HTTP (étape 2) suffit à
   confirmer que HTTP/3 est *annoncé*, sans garantir qu'il fonctionne.
2. **Headers** :
   ```
   npx --yes @mdn/mdn-http-observatory <domaine>
   ```
   Parser le JSON retourné (`scan.grade`, `scan.score`, `tests.*.pass`) plutôt
   que de coller le brut — ne lister que les tests en échec avec leur
   `scoreModifier`. Noter aussi le header `alt-svc` de `scan.responseHeaders`
   pour le support HTTP/3.
3. **Perf** :
   ```
   CHROME_PATH=<chemin chromium> npx --yes lighthouse https://<domaine> \
     --output=json --output-path=<fichier temporaire> \
     --chrome-flags="--headless=new --no-sandbox" \
     --only-categories=performance,accessibility,best-practices,seo --quiet
   ```
   Lire les scores dans `categories.*.score` (× 100). Regarder aussi
   l'audit `audits["uses-http2"]` (liste les ressources encore servies en
   HTTP/1.1 malgré un site en h2).
4. **Email (SPF/DKIM/DMARC)** — un seul `dig`, aucun outil à installer :
   ```
   dig +short TXT <domaine> | grep spf
   dig +short TXT _dmarc.<domaine>
   ```
   Le DKIM ne peut pas être vérifié en aveugle (le sélecteur — ex.
   `google._domainkey` — n'est pas standard et dépend du fournisseur
   d'emailing) : le signaler comme non vérifiable plutôt que de deviner un
   sélecteur au hasard, sauf si l'utilisateur en connaît un.

### CMS / plugins vulnérables (optionnel, sur proposition seulement)

Ne jamais lancer automatiquement — plus intrusif (login attempts, énumération
de plugins) et plus long que le reste de l'audit. Mais la détection du CMS
est gratuite : elle vient des données déjà collectées à l'étape 2 (header
`link: <.../wp-json/>` ou meta `generator` dans le HTML → WordPress ; motifs
équivalents pour Drupal/Joomla). Si un CMS connu est détecté dans le rapport,
**proposer** à l'utilisateur un scan dédié (ex. WPScan pour WordPress) en
précisant que :
- l'outil n'est pas installé par défaut (nécessite `npx`/Docker + éventuel
  token API WPVulnDB pour la base de vulnérabilités complète, quota gratuit
  limité) — demander confirmation avant d'installer quoi que ce soit ;
- ce n'est utile que si le CMS détecté correspond à un scanner existant —
  ne pas en inventer un pour un CMS maison/obscur.

## Rapport

Un tableau markdown : une ligne par domaine canonique, colonnes TLS / HTTP2-3
/ Headers / Email / Perf. Sous le tableau, lister concrètement les points en
échec (headers manquants, SPF/DMARC absent, score perf bas) avec la piste de
correction si évidente (ex: middleware reverse-proxy pour ajouter
CSP/HSTS/X-Frame-Options). Mentionner explicitement le résultat de l'étape 0
(apex→www redirige correctement / les deux formes sont indépendantes). Si un
CMS a été détecté, le signaler avec la proposition de scan dédié — sans le
lancer. Ne pas juste balancer le JSON brut — l'utilisateur veut un diagnostic
actionnable, pas une donnée.

Ne pas proposer de corriger automatiquement la config du site dans ce skill —
l'audit est en lecture seule, la correction est une tâche à part que
l'utilisateur doit valider explicitement.
