# Module 4 — Durcissement et exposition

Contrôles complémentaires, tous passifs (`curl`, `dig`, lecture du JSON
Lighthouse déjà produit). Aucun outil à installer. Chaque contrôle donne un
statut : `ok`, `à renforcer` ou `absent`. Le rapport ne retient que ce qui
n'est pas `ok`.

## 1. `security.txt`

```
curl -s -o /dev/null -w "%{http_code} %{content_type}\n" https://<domaine>/.well-known/security.txt
```

- 200 en `text/plain` : lire le contenu, vérifier la présence de `Contact:` et
  `Expires:` (RFC 9116). Un `Expires` dans le passé est `à renforcer`.
- 404 : `absent`. Recommandation faible, une ligne `Contact:` suffit.
- Ne pas confondre avec une page HTML 200 (soft 404) : vérifier le
  `content-type`.

## 2. Cookies

Récupérer tous les `Set-Cookie` de la page d'accueil et d'une éventuelle
redirection :

```
curl -sI https://<domaine>/ | grep -i '^set-cookie'
```

Pour chaque cookie, relever : `Secure`, `HttpOnly`, `SameSite`, `Path`,
durée (`Max-Age`/`Expires`) et préfixe (`__Host-`, `__Secure-`).

| Constat | Verdict |
|---|---|
| Pas de `Secure` | à renforcer (critique si cookie de session) |
| Pas de `HttpOnly` sur un cookie de session/auth | à renforcer |
| Pas de `SameSite` | à renforcer (le navigateur applique `Lax` par défaut, mais l'expliciter) |
| `SameSite=None` sans `Secure` | invalide, rejeté par les navigateurs |
| Durée de vie > 1 an | à signaler |

Un cookie de tracking ou de consentement n'a pas besoin de `HttpOnly` :
distinguer par le nom (`session`, `sid`, `auth`, `token` → sensible) plutôt
que de tout signaler. Aucun `Set-Cookie` sur la page d'accueil est un bon
résultat, pas une absence.

## 3. Fuite d'information serveur

Dans les headers de réponse déjà collectés (`scan.responseHeaders` d'Observatory) :

- `Server` avec **version** (`nginx/1.18.0`, `Apache/2.4.41 (Ubuntu)`) : à
  renforcer. Le nom seul (`nginx`, `cloudflare`) est acceptable.
- `X-Powered-By` (`PHP/8.1`, `Express`, `ASP.NET`) : à renforcer.
- `X-AspNet-Version`, `X-Generator`, `X-Drupal-Cache` : même logique.

Une version précise permet de chercher les CVE correspondantes sans rien
tester : c'est la raison du signalement.

## 4. Mixed content et ressources tierces

Lire dans le JSON Lighthouse déjà produit (aucune requête de plus) :

- `audits["is-on-https"]` : ressources chargées en HTTP depuis une page HTTPS.
  Toute entrée est `à renforcer`.
- `audits["third-party-summary"]` : domaines tiers, volume et temps de
  blocage. Lister les trois plus lourds.
- `audits["csp-xss"]` et `audits["redirects-http"]` si présents.

Ces audits informatifs ont souvent un `score` vide (`null`). Ce n'est pas un
« ok » : lire `details.items`. Seul un tableau vide ou absent signifie « rien
à signaler ».

Chaque domaine tiers est une dépendance de confiance : le signaler comme
information, pas comme défaut, sauf s'il est lourd ou bloquant.

## 5. DNSSEC

```
dig +short DS <domaine>
dig +dnssec +short A <domaine> | grep -c RRSIG
```

- `DS` présent et `RRSIG` retourné : `ok`.
- `DS` absent : `absent`. À recommander mais pas en échec : c'est une option
  du registrar, pas toujours disponible selon le TLD.
- `DS` présent mais pas de `RRSIG` : zone mal signée, à signaler en priorité
  (le domaine peut devenir injoignable pour les résolveurs validants).

## 6. MTA-STS et TLS-RPT (complète SPF/DKIM/DMARC)

À ne vérifier que si le domaine reçoit du courrier (présence d'un `MX`).

```
dig +short MX <domaine>
dig +short TXT _mta-sts.<domaine>
dig +short TXT _smtp._tls.<domaine>
curl -s https://mta-sts.<domaine>/.well-known/mta-sts.txt
```

- `_mta-sts` TXT (`v=STSv1; id=...`) et la policy accessible avec
  `mode: enforce` : `ok`. `mode: testing` : à renforcer. `none` : absent.
- `_smtp._tls` TXT (`v=TLSRPTv1; rua=...`) : active les rapports d'échec TLS
  entre serveurs mail. Absent : à signaler, faible priorité.
- Pas de `MX` : sans objet, ne rien reporter.
