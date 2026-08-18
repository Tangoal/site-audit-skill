# site-audit

Un [skill Claude Code](https://docs.claude.com/en/docs/claude-code/skills) qui
audite un site public sur trois axes indépendants — TLS, headers de sécurité
HTTP, performance — en s'appuyant sur trois outils open source établis :

| Axe | Outil | Équivalent web connu |
|---|---|---|
| TLS / certificat | [testssl.sh](https://github.com/drwetter/testssl.sh) | ssllabs.com/ssltest |
| Headers de sécurité | [MDN HTTP Observatory](https://github.com/mdn/mdn-http-observatory) | developer.mozilla.org/observatory |
| Performance | [Lighthouse](https://github.com/GoogleChrome/lighthouse) | pagespeed.web.dev |

Le skill ne réinvente aucun de ces outils : il sait où les trouver, les
installe à la demande s'ils manquent (clone `testssl.sh`, télécharge un
Chrome pour Lighthouse si besoin), les lance, et transforme leur sortie brute
en un rapport comparable et actionnable — au lieu de trois JSON séparés à
interpréter à la main.

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
