# Réponses – Partie 1 : prise en main

## 1. Commande httpie équivalente au `curl` de la route POST

```bash
http POST http://localhost:8080/api-v1/ url="https://perdu.com"
```

httpie envoie par défaut du JSON (`Content-Type: application/json`) ; `url="..."` (avec `=`) est
un champ de type chaîne. Pour forcer l'en-tête `Accept`, ajouter `Accept:application/json`.

## 2. Différences entre `npm run prod` et `npm run dev`

| | `npm run dev` | `npm run prod` |
|---|---|---|
| Commande | `cross-env NODE_ENV=development nodemon server.mjs` | `cross-env NODE_ENV=production node server.mjs` |
| Rechargement | nodemon relance le serveur à chaque modification de fichier | aucun, il faut relancer à la main |
| `NODE_ENV` | `development` | `production` |
| Niveau de log (loglevel) | `DEBUG` (très verbeux) | `WARN` (seulement avertissements et erreurs) |
| Morgan | actif : chaque requête est journalisée | désactivé |
| Pages d'erreur | la stack trace est affichée | pas de stack trace (ne pas révéler le code) |
| Performance | cache des templates EJS désactivé | Express active le cache des vues, et d'autres optimisations |

## 3. Script npm pour formatter tous les fichiers `.mjs`

Dans `package.json` :

```json
"format": "prettier --write \"**/*.mjs\""
```

Utilisation : `npm run format`.

## 4. Supprimer l'en-tête `X-Powered-By`

```js
app.disable("x-powered-by");
```

(à placer juste après `const app = express();`). C'est une bonne pratique de sécurité : on ne
révèle pas la technologie utilisée. Le module `helmet` fait aussi cela.

## 5. Middleware d'application qui ajoute `X-API-version`

```js
import { API_VERSION } from "./config.mjs";

app.use((_request, response, next) => {
  response.setHeader("X-API-version", API_VERSION);
  return next();
});
```

À enregistrer **avant** les routeurs pour que toutes les réponses aient l'en-tête.

## 6. Middleware pour `favicon.ico` → `static/logo_univ_16.png`

```bash
npm install serve-favicon
```

```js
import favicon from "serve-favicon";

app.use(favicon(path.join("static", "logo_univ_16.png")));
```

Le module répond aux requêtes `/favicon.ico` avec l'image (`Content-Type: image/x-icon`) et la
met en cache en mémoire.

## 7. Documentation du driver SQLite

L'application utilise deux paquets :

- `sqlite3` (le driver natif) : <https://github.com/TryGhost/node-sqlite3>, API :
  <https://github.com/TryGhost/node-sqlite3/wiki/API>
- `sqlite` (surcouche à base de promesses, permet `async/await`) :
  <https://github.com/kriasoft/node-sqlite>
- Le langage SQL de SQLite : <https://sqlite.org/docs.html>

## 8. Ouverture et fermeture de la connexion à la base

Dans `database/database.mjs`, la fonction `withDb()` :

- **ouvre** une connexion (`open(...)` dans `connect()`) au début de **chaque opération**
  (`countLinks`, `createLink`, ainsi que `initDatabase` au démarrage du serveur) ;
- la **ferme** (`db.close()`) dans un bloc `finally`, donc juste après l'opération, qu'elle
  réussisse ou échoue.

Aucune connexion n'est gardée ouverte entre deux requêtes HTTP. En mode dev, les messages
`open database` / `database closed` le montrent dans les logs.

## 9. Gestion du cache par Express

Mesuré avec `curl` sur la page d'accueil servie par `express.static` :

1. 1re visite : `200 OK`, avec `ETag: W/"..."`, `Last-Modified` et `Cache-Control: public, max-age=0`.
2. 2e visite : le navigateur renvoie `If-None-Match` avec l'ETag → `304 Not Modified` (pas de
   corps renvoyé, le navigateur réutilise sa copie).
3. `Ctrl+Shift+R` : le navigateur ajoute `Cache-Control: no-cache` et ignore sa copie
   → `200 OK` avec le contenu complet.

**Conclusion** : Express (via `serve-static`) gère un cache par **validation** : ETag et
`Last-Modified` sont générés automatiquement, mais `max-age=0` oblige le navigateur à
revalider à chaque fois. On économise la bande passante, pas la requête. Les réponses
dynamiques (`res.json`, `res.render`) reçoivent un ETag mais aucune politique de cache.

## 10. Deux instances (ports 8080 et 8081) : pourquoi les liens sont partagés ?

Les deux processus Node.js lisent le même fichier `database/database.sqlite`
(`DB_FILE` est un chemin relatif au dossier du projet, identique pour les deux instances).
Les données ne sont pas en mémoire dans le processus : elles sont dans ce fichier partagé,
et comme la connexion est rouverte à chaque requête, un lien créé par une instance est
immédiatement visible par l'autre. SQLite gère les accès concurrents avec des verrous sur
le fichier.

Conséquence : l'application est *stateless* (sans état en mémoire), ce qui permet de lancer
plusieurs instances derrière un répartiteur de charge tant qu'elles partagent la même base.
