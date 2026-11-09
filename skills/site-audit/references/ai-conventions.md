# Module 5 — Conventions IA

Vérifie ce qu'un site public expose pour les crawlers et agents IA. Tout
passe par `curl`. Ce module est une aide à la décision, pas une conformité :
la plupart de ces conventions sont volontaires ou encore émergentes, donc le
rapport emploie « absent » et « à envisager », jamais « échec ».

Ne s'applique qu'à ce qui s'audite depuis l'extérieur. `AGENTS.md`,
`DESIGN.md`, OKF et les serveurs MCP vivent dans un dépôt ou un runtime : hors
périmètre.

## 1. `robots.txt` et politique d'usage IA

```
curl -s https://<domaine>/robots.txt
```

- 200 en `text/plain` : présent. Un HTML ou un 404 déguisé : absent.
- Chercher une ligne `Content-signal:` (ex. `search=yes, ai-train=no`).
  Absente : signaler « pas de politique d'usage IA explicite », en précisant
  que l'absence ne restreint rien et n'autorise rien.
- Lister les `User-agent` d'IA bloqués (`GPTBot`, `ClaudeBot`, `CCBot`,
  `Google-Extended`, `PerplexityBot`…) : information, pas un jugement.
- Relever les lignes `Sitemap:`.

## 2. `sitemap.xml`

```
curl -s -o /dev/null -w "%{http_code} %{content_type}\n" https://<domaine>/sitemap.xml
```

- Tester aussi les URL déclarées dans `robots.txt`.
- 200 en `xml` : présent. Compter les `<loc>` (ou les sous-sitemaps d'un
  `sitemapindex`).
- Signaler si le sitemap n'est référencé dans aucun `robots.txt`.

## 3. `llms.txt`

```
curl -s -o /dev/null -w "%{http_code} %{content_type}\n" https://<domaine>/llms.txt
```

- Présent : vérifier que c'est du Markdown (titre `# ...` en première ligne,
  liens `[nom](url)`).
- Absent : à envisager **seulement** si le site est riche en documentation ou
  en contenu produit. Pour un site vitrine d'une page, ne rien recommander.
- Ne pas présenter comme un facteur SEO : ce n'est pas prouvé.

## 4. Schema.org / JSON-LD

Récupérer le HTML de la page d'accueil et extraire les blocs
`<script type="application/ld+json">` :

- Parser chaque bloc comme du JSON. Un bloc invalide est `à renforcer`.
- Lister les `@type` trouvés (`Organization`, `WebSite`, `LocalBusiness`,
  `Article`, `FAQPage`…).
- Aucun bloc : `absent`. À recommander pour une entreprise ou un commerce
  local (`Organization`/`LocalBusiness`), sans objet pour une app ou un outil.

## 5. Markdown for Agents

```
curl -sI -H "Accept: text/markdown" https://<domaine>/ | grep -iE "^(content-type|x-markdown-tokens|content-signal)"
```

- `content-type: text/markdown` : la fonctionnalité est active. Noter
  `x-markdown-tokens` s'il est présent.
- **Alerte** : si la fonctionnalité est active mais que la réponse n'a pas de
  header `content-signal`, la conversion côté Cloudflare applique par défaut
  `ai-train=yes, search=yes, ai-input=yes`. Recommander de poser le header
  `content-signal` sur l'origine avant d'activer la fonction.
- Réponse en `text/html` : fonctionnalité inactive, rien à signaler.

## 6. Agent card A2A (optionnel)

```
curl -s -o /dev/null -w "%{http_code} %{content_type}\n" https://<domaine>/.well-known/agent-card.json
```

Ne tester que si le site prétend exposer un agent. 404 est le résultat normal
pour un site classique : ne rien reporter.

## À ne pas auditer

- **WebMCP** : brouillon d'un groupe communautaire, API encore instable. Au
  plus, le mentionner comme « à surveiller ».
- **ACP / UCP / AP2** (commerce agentique) : seulement si le site vend en
  ligne, et sur la base des spécifications officielles.

## Source de référence

Ces conventions évoluent vite. En cas de doute sur un nom de fichier ou un
header, relire les spécifications primaires (llmstxt.org, RFC 9309, RFC 9116,
developers.cloudflare.com, a2a-protocol.org) plutôt que de se fier à ce skill.
