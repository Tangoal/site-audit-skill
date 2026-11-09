---
name: site-audit
description: Audite un site public (TLS, HTTP/2-3, headers de sécurité, cookies, DNS/email, performance, conventions IA comme robots.txt/llms.txt) via testssl.sh, MDN HTTP Observatory et Lighthouse plus des contrôles passifs, compare apex et www, et mesure l'évolution depuis le dernier audit. À utiliser quand l'utilisateur demande d'auditer/tester/checker la sécurité, la perf ou la visibilité IA d'un site, ou mentionne testssl/Observatory/Lighthouse/PageSpeed pour un de ses domaines.
---

# /site-audit — Audit sécurité, perf et conventions IA d'un site public

Lance trois outils tiers puis deux modules de contrôles passifs contre un ou
plusieurs domaines, et synthétise le résultat dans un tableau avec l'évolution
depuis le dernier audit. Lecture seule côté site audité — requêtes
HTTPS/TLS/DNS normales, jamais d'écriture ni de modification.

| Outil | Mesure | Équivalent web connu |
|---|---|---|
| [`testssl.sh`](https://github.com/testssl/testssl.sh) | Config TLS/certif, ciphers, vulnérabilités connues (Heartbleed, POODLE, ROBOT...) | ssllabs.com/ssltest |
| [`@mdn/mdn-http-observatory`](https://github.com/mdn/mdn-http-observatory) | Headers de sécurité HTTP (CSP, HSTS, X-Frame-Options...) | developer.mozilla.org/observatory |
| [Lighthouse](https://github.com/GoogleChrome/lighthouse) | Performance, accessibilité, best-practices, SEO | pagespeed.web.dev |

Ces trois sujets sont indépendants et complémentaires : un site peut avoir un
A+ TLS et un D sur les headers, ou l'inverse. Toujours faire les trois, jamais
en déduire un score global unique.

Deux modules passifs s'y ajoutent, sans outil à installer (`curl`, `dig`).
Leur détail est dans des fichiers à part, à lire seulement à l'étape
concernée :

| Module | Contenu | Fichier |
|---|---|---|
| 4. Durcissement et exposition | security.txt, cookies, fuite d'info serveur, mixed content et tiers, DNSSEC, MTA-STS/TLS-RPT | `references/hardening.md` |
| 5. Conventions IA | robots.txt + Content-signal, sitemap, llms.txt, JSON-LD, Markdown for Agents | `references/ai-conventions.md` |

Le module 4 relève de la sécurité (un défaut y est un défaut). Le module 5
relève de la conformité à des conventions volontaires ou émergentes : ne
jamais le présenter comme un échec, seulement comme « absent » ou « à
envisager ».

## Dépendances : détection et installation à la demande

Ce skill ne suppose rien d'installé à l'avance. Avant de lancer un audit,
vérifier/installer dans cet ordre :

**1. `git` et `node`/`npx`** — prérequis de base, s'ils manquent le signaler
à l'utilisateur et arrêter (ne pas tenter d'installer des paquets système de
base sans confirmation explicite).

**2. `testssl.sh`** — chercher dans l'ordre :
   - `testssl.sh` dans le `PATH` (`command -v testssl.sh`)
   - un clone existant dans `~/.cache/site-audit/testssl.sh/testssl.sh`
   - sinon cloner : `git clone --depth 1 --branch 3.2 https://github.com/testssl/testssl.sh.git ~/.cache/site-audit/testssl.sh`
   - **Toujours vérifier le résultat du clone** (`test -x ~/.cache/site-audit/testssl.sh/testssl.sh`) :
     git peut réussir en silence sur un dépôt vide (« You appear to have
     cloned an empty repository »). Si le script est absent, supprimer le
     dossier vide (sinon la détection suivante croira à un clone valide) et
     appliquer le plan B ci-dessous.

**Plan B si `testssl.sh` est indisponible** — ne pas bloquer l'audit, ne pas
insister avec des sources de téléchargement de rechange non prévues :
   - dire à l'utilisateur en une phrase que testssl.sh n'a pas pu être
     installé et pourquoi ;
   - marquer le TLS « **non mesuré** » dans le tableau et dans l'historique
     (`"grade": null`, avec une note), jamais une note estimée ;
   - ajouter, à titre indicatif seulement, un sondage minimal à `openssl` :
     version négociée, refus de TLS 1.0/1.1 et date d'expiration du certificat
     (`echo | openssl s_client -connect <d>:443 -servername <d> | openssl x509 -noout -enddate`).
     Le présenter comme « sondage openssl, pas un audit TLS » : il ne teste ni
     les chiffrements ni les vulnérabilités connues.

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
- Si les deux formes répondent 200 indépendamment (pas de redirection),
  comparer d'abord leurs réponses : mêmes headers de sécurité (`strict-transport-security`,
  `content-security-policy`, `x-frame-options`, `referrer-policy`,
  `x-content-type-options`), même `server`, même `last-modified`/`etag` et même
  certificat (`openssl s_client ... | openssl x509 -noout -fingerprint`).
  - **Identiques** (cas d'un même site derrière le même proxy) : lancer le
    trio complet sur l'apex seulement, et indiquer dans le rapport que le
    `www` est un alias servi à l'identique. Signaler tout de même que deux
    URL servent le même contenu sans redirection (duplication pour le SEO,
    surface doublée), avec la piste : rediriger une forme vers l'autre.
  - **Différentes** (headers, contenu ou certificat qui divergent) : les
    traiter comme deux domaines distincts et lancer le trio complet sur
    chacune.

Pour chaque domaine canonique retenu :

1. **TLS + protocole + expiration certif** :
   ```
   <chemin testssl.sh> --quiet --color 0 --fast <domaine> 2>&1 | grep -E "^\s*(TLS 1|ALPN|HTTP2|Overall Grade|Certificate Validity|Heartbleed|POODLE|ROBOT|FREAK|DROWN|LOGJAM|BEAST)"
   ```
   Le grep élargi (`ALPN`, `HTTP2`) fait remonter le support HTTP/2 (h2) —
   pas besoin d'outil séparé. testssl.sh ne teste pas HTTP/3/QUIC (UDP) ; le
   header `alt-svc: h3=...` vu dans la réponse HTTP (étape 2) suffit à
   confirmer que HTTP/3 est *annoncé*, sans garantir qu'il fonctionne.
   `Certificate Validity` donne la date d'expiration — c'est le risque le
   plus bête et le plus fréquent (site qui tombe parce que personne n'a vu
   le renouvellement expirer) ; le signaler en alerte dans le rapport si
   l'échéance est proche (testssl.sh l'annonce déjà lui-même sous ~30-60j).
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
   Lire les scores dans `categories.*.score` (× 100). Relever aussi les
   métriques qui expliquent un score de performance bas : LCP, CLS, TBT
   (`audits["largest-contentful-paint"].displayValue`, etc.), et pour
   l'accessibilité les audits en échec (`score == 0`) de la catégorie.
   Regarder aussi `audits["uses-http2"]` (ressources encore servies en
   HTTP/1.1 malgré un site en h2).

   **Piège** : certains audits informatifs (`uses-http2`, `third-party-summary`,
   `csp-xss`, `redirects-http`) ont un `score` vide (`null`) alors qu'ils
   contiennent des données utiles. Un `score` vide ne veut jamais dire « rien
   à signaler » : lire `details.items` (ou `displayValue`) et ne conclure
   « rien à signaler » que si ce tableau est vide ou absent.
4. **Email (SPF/DKIM/DMARC) + CAA** — quelques `dig`, aucun outil à installer :
   ```
   dig +short TXT <domaine> | grep spf
   dig +short TXT _dmarc.<domaine>
   dig +short CAA <domaine>
   ```
   Le DKIM ne peut pas être vérifié en aveugle (le sélecteur — ex.
   `google._domainkey` — n'est pas standard et dépend du fournisseur
   d'emailing) : le signaler comme non vérifiable plutôt que de deviner un
   sélecteur au hasard, sauf si l'utilisateur en connaît un.
   Le CAA restreint qui a le droit d'émettre un certificat pour le domaine ;
   son absence n'est pas une faille en soi (n'importe quelle CA peut émettre
   par défaut) mais c'est un renforcement facile à recommander.
   **Piège derrière un proxy (Cloudflare, etc.)** : le certificat vu par les
   visiteurs est émis par le proxy, pas par l'origine. Un CAA qui ne liste que
   l'autorité de l'origine peut bloquer le renouvellement du certificat de
   bordure. Toujours recommander d'y inclure aussi les autorités du proxy
   (voir sa documentation), jamais un CAA « Let's Encrypt seul » à l'aveugle.
5. **Durcissement et exposition** — lire `references/hardening.md` et
   appliquer les six contrôles. Réutiliser les données déjà collectées aux
   étapes 2 et 3 (headers de réponse, JSON Lighthouse) au lieu de refaire des
   requêtes.
6. **Conventions IA** — lire `references/ai-conventions.md` et appliquer les
   contrôles. Les formuler comme des constats, pas comme des échecs.

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

## Historique et évolution

Chaque audit est enregistré, pour que le suivant puisse dire ce qui a changé.

**Avant** de lancer les outils sur un domaine, chercher le dernier audit :
`ls ~/.cache/site-audit/history/<domaine>/ | sort | tail -1`. S'il existe, le
lire : c'est la référence pour la colonne Δ.

**Après** l'audit, écrire `~/.cache/site-audit/history/<domaine>/<AAAA-MM-JJ>T<HHMM>.json`
(heure locale, créer le dossier au besoin). **Ne jamais écraser un fichier
existant** : deux audits le même jour (avant puis après une correction) doivent
rester comparables. Le tri alphabétique donne l'ordre chronologique, y compris
pour d'anciens fichiers nommés `<AAAA-MM-JJ>.json` :

```json
{
  "domain": "exemple.fr",
  "date": "2026-11-09",
  "tls": { "grade": "A+", "cert_expires": "2027-02-01", "cert_days_left": 84 },
  "http": { "h2": true, "h3_advertised": true },
  "headers": { "grade": "B+", "score": 75, "failed": ["content-security-policy"] },
  "perf": { "performance": 92, "accessibility": 100, "best_practices": 96, "seo": 100 },
  "email": { "spf": true, "dmarc": "p=quarantine", "caa": false, "mta_sts": false, "dnssec": false },
  "durcissement": { "issues": ["cookie-sans-samesite", "server-version"] },
  "conventions_ia": { "robots": true, "content_signal": false, "sitemap": true, "llms_txt": false, "jsonld": ["Organization"] }
}
```

Les champs sont indicatifs : écrire ce qu'on a mesuré, omettre ce qui ne
s'applique pas. Ne jamais y mettre de donnée sensible (valeur de cookie,
contenu de `security.txt`).

**Lecture des deltas** : comparer champ à champ avec la dernière entrée.
Signaler en priorité les **régressions** (grade qui baisse, score perf qui
chute de plus de 5 points, nouveau point en échec, certif qui passe sous 30
jours, DMARC affaibli) puis les améliorations. Sans historique, le dire
(« premier audit de ce domaine ») au lieu de laisser la colonne vide. Les
fichiers restent sur la machine de l'utilisateur ; ne jamais les envoyer
ailleurs.

## Rapport

Un tableau markdown : une ligne par domaine canonique, colonnes TLS
(avec date d'expiration certif) / HTTP2-3 / Headers / Email+CAA / Perf /
Durcissement / Conventions IA / Δ depuis le dernier audit. Sous
le tableau, lister concrètement les points en échec (headers manquants,
SPF/DMARC/CAA absent, certif proche de l'expiration, score perf bas, cookies
ou fuite de version serveur) avec la piste de
correction si évidente (ex: middleware reverse-proxy pour ajouter
CSP/HSTS/X-Frame-Options). Traiter les constats du module Conventions IA dans
une section à part, plus courte, présentés comme des options. Mentionner explicitement le résultat de l'étape 0
(apex→www redirige correctement / les deux formes sont indépendantes). Si un
CMS a été détecté, le signaler avec la proposition de scan dédié — sans le
lancer. Ne pas juste balancer le JSON brut — l'utilisateur veut un diagnostic
actionnable, pas une donnée.

### Rapport HTML et PDF

Après le tableau markdown, **générer aussi un rapport HTML** : un fichier
unique, sans dépendance externe, qui s'ouvre hors ligne et s'envoie tel quel.

- Partir de `references/report-template.html` (le lire, remplacer les
  `{{PLACEHOLDER}}`, dupliquer ou supprimer les blocs `REPEAT`). Ne pas
  réinventer le design : page sobre, une colonne, un point de couleur par
  ligne, jamais de score global.
- Écrire dans `~/.cache/site-audit/reports/<domaine>/<AAAA-MM-JJ>T<HHMM>.html`
  (créer le dossier, même convention de nom que l'historique, ne rien écraser).
  Donner le chemin complet à l'utilisateur.
- Le contenu est celui du rapport markdown, pas un autre : mêmes constats,
  même ordre de gravité, mêmes deltas. Pas de donnée sensible dedans.
- Vérifier le rendu avant de le présenter : capture avec le Chrome déjà
  détecté pour Lighthouse
  (`<chrome> --headless=new --no-sandbox --screenshot=<png> --window-size=800,1500 file://<rapport>.html`),
  puis la regarder.

**PDF, seulement si l'utilisateur le demande** (même Chrome, aucun outil de
plus) :

```
<chrome> --headless=new --no-sandbox --no-pdf-header-footer \
  --print-to-pdf=<rapport>.pdf file://<rapport>.html
```

Le gabarit contient déjà les règles d'impression (pas de coupure au milieu
d'une ligne ou d'un constat).

Ne pas proposer de corriger automatiquement la config du site dans ce skill —
l'audit est en lecture seule, la correction est une tâche à part que
l'utilisateur doit valider explicitement.
