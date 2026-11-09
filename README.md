# site-audit

Un [skill Claude Code](https://docs.claude.com/en/docs/claude-code/skills) qui
audite un site public sur trois axes indépendants — TLS, headers de sécurité
HTTP, performance — en s'appuyant sur trois outils open source établis, puis
complète avec deux modules de contrôles passifs et un suivi dans le temps :

| Axe | Outil | Équivalent web connu |
|---|---|---|
| TLS / certificat | [testssl.sh](https://github.com/drwetter/testssl.sh) | ssllabs.com/ssltest |
| Headers de sécurité | [MDN HTTP Observatory](https://github.com/mdn/mdn-http-observatory) | developer.mozilla.org/observatory |
| Performance | [Lighthouse](https://github.com/GoogleChrome/lighthouse) | pagespeed.web.dev |

Le skill ne réinvente aucun de ces outils : il sait où les trouver, les
installe à la demande s'ils manquent (clone `testssl.sh`, télécharge un
Chrome pour Lighthouse si besoin), les lance, et transforme leur sortie brute
en un rapport actionnable — au lieu de trois JSON séparés à interpréter à la
main.

## Modules complémentaires

Sans outil à installer (`curl` et `dig` suffisent), lecture seule :

| Module | Contrôles |
|---|---|
| Durcissement et exposition | `security.txt`, flags des cookies (`Secure`, `HttpOnly`, `SameSite`), fuite de version serveur (`Server`, `X-Powered-By`), mixed content et ressources tierces, DNSSEC, MTA-STS et TLS-RPT |
| Conventions IA | `robots.txt` et `Content-signal`, `sitemap.xml`, `llms.txt`, JSON-LD (Schema.org), Markdown for Agents |

Le premier relève de la sécurité. Le second mesure l'adoption de conventions
volontaires ou émergentes : ses constats sont présentés comme des options, pas
comme des échecs. Le détail de chaque contrôle est dans
`skills/site-audit/references/`.

## Historique

Chaque audit est enregistré dans `~/.cache/site-audit/history/<domaine>/<date>.json`.
Au lancement suivant, le rapport ajoute une colonne Δ qui signale les
régressions (grade qui baisse, score perf qui chute, certificat proche de
l'expiration, nouveau point en échec) puis les améliorations. Les fichiers
restent sur votre machine. Supprimez le dossier pour repartir de zéro.

## Rapport HTML et PDF

En plus du tableau markdown, le skill génère un rapport HTML sobre et
autonome (un seul fichier, thème clair/sombre, adapté à l'impression) dans
`~/.cache/site-audit/reports/<domaine>/<date>.html`. Un PDF peut en être tiré
sur demande avec le Chrome déjà utilisé par Lighthouse. Le gabarit est dans
`skills/site-audit/references/report-template.html`.

## Si un outil est indisponible

Le skill ne bloque pas l'audit. Si `testssl.sh` ne peut pas être installé
(par exemple un clone vide), le TLS est marqué « non mesuré » et complété par
un sondage `openssl` indicatif, jamais par une note inventée.

## Installation

```
/plugin marketplace add <url-de-ce-repo>
/plugin install site-audit
```

## Usage

```
/site-audit exemple.com
/site-audit exemple.com autredomaine.fr
```

## Pourquoi pas un seul outil tout-en-un ?

Aucun outil ne couvre les trois axes correctement. SSL Labs et testssl.sh
ignorent les headers de sécurité applicatifs (CSP, HSTS...) ; Observatory
ignore le TLS et la perf ; PageSpeed/Lighthouse ignore la sécurité. Un site
peut avoir un A+ TLS et un D sur les headers — les trois signaux sont
nécessaires et aucun ne remplace les deux autres.

## Licence

MIT pour le code de ce skill. Les outils invoqués (`testssl.sh` en GPLv2,
`mdn-http-observatory` en MPL-2.0, Lighthouse en Apache-2.0) restent sous
leur propre licence — ce skill les appelle en ligne de commande sans en
redistribuer ni modifier le code.
