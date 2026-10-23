# Vendored Frontend Libraries (exactly five files)

Download each file verbatim into `inventory-web/src/main/webapp/`. Do not modify, re-minify or truncate them, and
do not add glyphicon fonts, source maps or non-minified copies.

| Target path | Source URL | Size (bytes) | SHA-256 |
|---|---|---|---|
| `lib/jquery/jquery.min.js` | https://code.jquery.com/jquery-1.12.4.min.js | 97163 | `668b046d12db350ccba6728890476b3efee53b2f42dbb84743e5e9f1ae0cc404` |
| `lib/angular/angular.min.js` | https://ajax.googleapis.com/ajax/libs/angularjs/1.5.11/angular.min.js | 163188 | `363bd9f20220f732c5771977a1deb892d8c28452c10b1a899c54757fa52ab579` |
| `lib/angular-route/angular-route.min.js` | https://ajax.googleapis.com/ajax/libs/angularjs/1.5.11/angular-route.min.js | 4786 | `fb3fb2da59c1c385759cb3eb43d3058cf0c776dbfcc3199a9dcd3c980caf0180` |
| `lib/bootstrap/css/bootstrap.min.css` | https://cdn.jsdelivr.net/npm/bootstrap@3.3.7/dist/css/bootstrap.min.css | 121200 | `f75e846cc83bd11432f4b1e21a45f31bc85283d11d372f7b19accd1bf6a2635c` |
| `lib/bootstrap/js/bootstrap.min.js` | https://cdn.jsdelivr.net/npm/bootstrap@3.3.7/dist/js/bootstrap.min.js | 37045 | `53964478a7c634e8dad34ecc303dd8048d00dce4993906de1bacf67f663486ef` |

```bash
cd inventory-web/src/main/webapp
mkdir -p lib/jquery lib/angular lib/angular-route lib/bootstrap/css lib/bootstrap/js
curl -sSfL https://code.jquery.com/jquery-1.12.4.min.js -o lib/jquery/jquery.min.js
curl -sSfL https://ajax.googleapis.com/ajax/libs/angularjs/1.5.11/angular.min.js -o lib/angular/angular.min.js
curl -sSfL https://ajax.googleapis.com/ajax/libs/angularjs/1.5.11/angular-route.min.js -o lib/angular-route/angular-route.min.js
curl -sSfL https://cdn.jsdelivr.net/npm/bootstrap@3.3.7/dist/css/bootstrap.min.css -o lib/bootstrap/css/bootstrap.min.css
curl -sSfL https://cdn.jsdelivr.net/npm/bootstrap@3.3.7/dist/js/bootstrap.min.js -o lib/bootstrap/js/bootstrap.min.js
shasum -a 256 lib/*/*.js lib/bootstrap/*/*   # compare against the table above
```

Alternative mirrors, if a CDN is unreachable: `https://cdnjs.cloudflare.com/ajax/libs/jquery/1.12.4/jquery.min.js`,
`https://cdnjs.cloudflare.com/ajax/libs/angular.js/1.5.11/angular.min.js`,
`https://cdnjs.cloudflare.com/ajax/libs/angular.js/1.5.11/angular-route.min.js`,
`https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/{css/bootstrap.min.css,js/bootstrap.min.js}`.
A mirror's bytes may differ slightly (headers or line endings). In that case accept the file only if its first
line carries the expected version banner (`jQuery v1.12.4`, `AngularJS v1.5.11`, `Bootstrap v3.3.7`), and mention
the substitution in the final run report. If no source is reachable, stop and report it as a
blocker; never fabricate library contents.
