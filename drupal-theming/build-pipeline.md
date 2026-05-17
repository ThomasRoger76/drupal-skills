# Build Pipeline Node.js pour Thèmes Drupal

## Pourquoi Compiler en Local

| Besoin | Sans build pipeline | Avec build pipeline |
|--------|--------------------|--------------------|
| SCSS → CSS | Impossible (navigateurs ne lisent pas .scss) | `sass` + sourcemaps |
| JS ES6+ → bundle | Risques de compatibilité navigateurs | Babel / esbuild → bundle universel |
| Bootstrap custom | CDN uniquement, variables impossibles | `npm install bootstrap`, variables SCSS surchargées |
| Minification | À la main ou via Drupal (lent) | Build prod minifié automatiquement |
| Hot reload | Rechargement manuel | `--watch` → rechargement automatique |
| Linting CSS/JS | Aucun | ESLint + Stylelint dans le pipeline |

---

## Structure Type d'un Thème avec Build Pipeline

```
web/themes/custom/montheme/
├── package.json              # Dépendances Node.js et scripts npm
├── gulpfile.js               # (Option Gulp) Pipeline de compilation
├── webpack.config.js         # (Option Webpack) Alternative à Gulp
├── .babelrc                  # (si Babel) Config de transpilation JS
├── .stylelintrc.json         # Lint SCSS
├── .eslintrc.json            # Lint JS
├── .gitignore                # Ignorer node_modules, dist/ optionnel
│
├── src/                      # Sources non compilées
│   ├── scss/
│   │   ├── main.scss         # Point d'entrée SCSS
│   │   ├── base/
│   │   ├── layout/
│   │   ├── components/
│   │   └── theme/
│   └── js/
│       ├── main.js           # Point d'entrée JS
│       ├── behaviors/        # Drupal behaviors ES6
│       └── vendor/           # JS tiers non-npm
│
├── dist/                     # Assets compilés (référencés dans .libraries.yml)
│   ├── css/
│   │   └── main.css
│   └── js/
│       └── bundle.js
│
├── montheme.info.yml
├── montheme.libraries.yml    # Pointe vers dist/
└── montheme.theme
```

---

## Pipeline Gulp Complet pour Drupal

### `package.json`

```json
{
  "name": "montheme",
  "version": "1.0.0",
  "description": "Build pipeline pour le thème Drupal Bootstrap5",
  "scripts": {
    "watch": "gulp watch",
    "build": "NODE_ENV=production gulp build",
    "dev": "gulp default",
    "lint:css": "stylelint 'src/scss/**/*.scss'",
    "lint:js": "eslint 'src/js/**/*.js'",
    "lint": "npm run lint:css && npm run lint:js"
  },
  "devDependencies": {
    "gulp": "^4.0.2",
    "gulp-sass": "^5.1.0",
    "sass": "^1.75.0",
    "gulp-sourcemaps": "^3.0.0",
    "gulp-postcss": "^10.0.0",
    "autoprefixer": "^10.4.19",
    "cssnano": "^7.0.0",
    "gulp-uglify": "^3.0.2",
    "gulp-babel": "^8.0.0",
    "@babel/core": "^7.24.0",
    "@babel/preset-env": "^7.24.0",
    "gulp-concat": "^2.6.4",
    "gulp-rename": "^2.0.0",
    "gulp-plumber": "^1.2.1",
    "browser-sync": "^3.0.2"
  },
  "dependencies": {
    "bootstrap": "^5.3.3"
  }
}
```

### `gulpfile.js`

```javascript
'use strict';

const gulp        = require('gulp');
const sass        = require('gulp-sass')(require('sass'));
const sourcemaps  = require('gulp-sourcemaps');
const postcss     = require('gulp-postcss');
const autoprefixer = require('autoprefixer');
const cssnano     = require('cssnano');
const babel       = require('gulp-babel');
const uglify      = require('gulp-uglify');
const concat      = require('gulp-concat');
const rename      = require('gulp-rename');
const plumber     = require('gulp-plumber');
const browserSync = require('browser-sync').create();

const isProd = process.env.NODE_ENV === 'production';

// ─── Chemins ──────────────────────────────────────────────────────────────────
const paths = {
  scss: {
    src:  'src/scss/**/*.scss',
    main: 'src/scss/main.scss',
    dest: 'dist/css',
  },
  js: {
    src:  'src/js/**/*.js',
    dest: 'dist/js',
  },
  images: {
    src:  'src/images/**/*',
    dest: 'dist/images',
  },
};

// ─── CSS : SCSS → CSS ─────────────────────────────────────────────────────────
function styles() {
  const postPlugins = [autoprefixer()];
  if (isProd) {
    postPlugins.push(cssnano({ preset: 'default' }));
  }

  let stream = gulp.src(paths.scss.main)
    .pipe(plumber({
      errorHandler: function(err) {
        console.error('SCSS Error:', err.message);
        this.emit('end');
      }
    }));

  if (!isProd) {
    stream = stream.pipe(sourcemaps.init());
  }

  stream = stream
    .pipe(sass({
      includePaths: ['node_modules'],   // Accès à node_modules/bootstrap/scss
      outputStyle: isProd ? 'compressed' : 'expanded',
    }).on('error', sass.logError))
    .pipe(postcss(postPlugins));

  if (!isProd) {
    stream = stream.pipe(sourcemaps.write('.'));
  }

  return stream
    .pipe(rename({ basename: 'main' }))
    .pipe(gulp.dest(paths.scss.dest))
    .pipe(browserSync.stream());
}

// ─── JS : ES6+ → bundle ────────────────────────────────────────────────────────
function scripts() {
  let stream = gulp.src(paths.js.src)
    .pipe(plumber({
      errorHandler: function(err) {
        console.error('JS Error:', err.message);
        this.emit('end');
      }
    }));

  if (!isProd) {
    stream = stream.pipe(sourcemaps.init());
  }

  stream = stream
    .pipe(babel({
      presets: [['@babel/preset-env', {
        targets: '> 0.5%, last 2 versions, not dead',
        useBuiltIns: 'usage',
        corejs: 3,
      }]]
    }))
    .pipe(concat('bundle.js'));

  if (isProd) {
    stream = stream.pipe(uglify());
  } else {
    stream = stream.pipe(sourcemaps.write('.'));
  }

  return stream
    .pipe(gulp.dest(paths.js.dest))
    .pipe(browserSync.stream());
}

// ─── Images ────────────────────────────────────────────────────────────────────
function images() {
  return gulp.src(paths.images.src)
    .pipe(gulp.dest(paths.images.dest));
}

// ─── Watch ─────────────────────────────────────────────────────────────────────
function watch() {
  // URL de l'environnement local Docker Compose
  browserSync.init({
    proxy: 'https://mon-projet.docker compose exec php.site',
    https: false,
    open: false,
    notify: false,
  });

  gulp.watch(paths.scss.src, styles);
  gulp.watch(paths.js.src, scripts);
  gulp.watch(paths.images.src, images);

  // Recharger le navigateur si les templates Twig changent
  gulp.watch('templates/**/*.twig').on('change', browserSync.reload);
}

// ─── Tâches exposées ──────────────────────────────────────────────────────────
const build = gulp.parallel(styles, scripts, images);

exports.styles  = styles;
exports.scripts = scripts;
exports.images  = images;
exports.watch   = gulp.series(build, watch);
exports.build   = build;
exports.default = build;
```

### `.babelrc`

```json
{
  "presets": [
    ["@babel/preset-env", {
      "targets": "> 0.5%, last 2 versions, not dead",
      "useBuiltIns": "usage",
      "corejs": 3
    }]
  ]
}
```

---

## Pipeline Webpack pour Thèmes Modernes

### `package.json` (variante Webpack)

```json
{
  "name": "montheme",
  "version": "1.0.0",
  "scripts": {
    "watch": "webpack --watch --mode development",
    "build": "webpack --mode production",
    "dev": "webpack serve --mode development"
  },
  "devDependencies": {
    "webpack": "^5.91.0",
    "webpack-cli": "^5.1.4",
    "webpack-dev-server": "^5.0.4",
    "sass": "^1.75.0",
    "sass-loader": "^14.2.0",
    "css-loader": "^7.1.1",
    "mini-css-extract-plugin": "^2.9.0",
    "postcss-loader": "^8.1.1",
    "autoprefixer": "^10.4.19",
    "babel-loader": "^9.1.3",
    "@babel/core": "^7.24.0",
    "@babel/preset-env": "^7.24.0"
  },
  "dependencies": {
    "bootstrap": "^5.3.3"
  }
}
```

### `webpack.config.js`

```javascript
const path = require('path');
const MiniCssExtractPlugin = require('mini-css-extract-plugin');

module.exports = (env, argv) => {
  const isDev = argv.mode === 'development';

  return {
    entry: {
      main: './src/js/main.js',         // JS + SCSS Bootstrap importé ici
    },
    output: {
      filename: 'js/[name].bundle.js',
      path: path.resolve(__dirname, 'dist'),
      clean: true,
    },
    devtool: isDev ? 'source-map' : false,
    module: {
      rules: [
        // JS — transpilation Babel
        {
          test: /\.js$/,
          exclude: /node_modules/,
          use: {
            loader: 'babel-loader',
            options: {
              presets: [['@babel/preset-env', { targets: '> 0.5%, last 2 versions' }]],
            },
          },
        },
        // SCSS → CSS extrait dans dist/css/
        {
          test: /\.scss$/,
          use: [
            MiniCssExtractPlugin.loader,
            { loader: 'css-loader', options: { sourceMap: isDev } },
            { loader: 'postcss-loader', options: {
              postcssOptions: { plugins: ['autoprefixer'] },
              sourceMap: isDev,
            }},
            { loader: 'sass-loader', options: {
              sourceMap: isDev,
              sassOptions: { includePaths: ['node_modules'] },
            }},
          ],
        },
      ],
    },
    plugins: [
      new MiniCssExtractPlugin({
        filename: 'css/[name].css',
      }),
    ],
  };
};
```

```javascript
// src/js/main.js — point d'entrée qui importe tout
import '../scss/main.scss';   // Bootstrap SCSS + styles custom

// Drupal behaviors ES6
(function (Drupal, once) {
  'use strict';

  Drupal.behaviors.montheme = {
    attach: function (context, settings) {
      once('montheme-init', '[data-montheme]', context).forEach(function (el) {
        // Logique custom ici
      });
    }
  };
})(Drupal, once);
```

---

## Intégration Docker — Service Node.js

### `docker-compose.yml` — Service `webpack_theming`

```yaml
services:
  # ... autres services PHP, MariaDB, Nginx ...

  webpack_theming:
    image: node:22-alpine
    working_dir: /app/web/themes/custom/montheme
    volumes:
      - .:/app
    command: npm run watch
    # Pas de port exposé — BrowserSync optionnel
    # Pour BrowserSync avec proxy, exposer le port 3000 :
    # ports:
    #   - "3000:3000"
    environment:
      - NODE_ENV=development
    # Redémarrer automatiquement si le conteneur s'arrête
    restart: unless-stopped
    # Attendre que PHP soit prêt (optionnel mais propre)
    depends_on:
      - php
```

### Commandes utiles avec Docker

```bash
# Lancer le watch en arrière-plan (via docker compose)
docker compose up webpack_theming -d

# Voir les logs de compilation en temps réel
docker compose logs -f webpack_theming

# Lancer un build de production depuis Docker
docker compose run --rm webpack_theming npm run build

# Installer les dépendances dans le conteneur
docker compose run --rm webpack_theming npm install

# Mettre à jour une dépendance
docker compose run --rm webpack_theming npm update bootstrap
```

### Service Node.js additionnel (Docker Compose)

```yaml
# .docker compose exec php/docker-compose.node.yml
services:
  node:
    image: node:22-alpine
    container_name: docker compose exec php-${DDEV_SITENAME}-node
    volumes:
      - ../web/themes/custom/montheme:/app
    working_dir: /app
    command: sh -c "npm install && npm run watch"
    restart: unless-stopped
    labels:
      com.docker compose exec php.site-name: ${DDEV_SITENAME}
      com.docker compose exec php.approot: ${DDEV_APPROOT}
```

```bash
# Dans le Makefile ou Taskfile pour installer les dépendances
# .docker compose exec php/config.yaml
hooks:
  post-start:
    - exec-host: "docker compose -f .docker compose exec php/docker-compose.node.yml exec node npm install"
```

---

## `.libraries.yml` Pointant vers les Assets Compilés

```yaml
# montheme.libraries.yml — assets compilés dans dist/

global:
  version: 1.0
  css:
    theme:
      dist/css/main.css: { minified: true }   # minified: true = pas de traitement Drupal
  js:
    dist/js/bundle.js: { minified: true }
  dependencies:
    - core/drupal
    - core/once

# En développement — avec sourcemaps non minifiés
# Changer minified: false pour profiter des sourcemaps dans les devtools
global-dev:
  version: 1.0
  css:
    theme:
      dist/css/main.css: {}    # Pas minified: true → Drupal peut lire les sourcemaps
  js:
    dist/js/bundle.js: {}
  dependencies:
    - core/drupal
    - core/once
```

> **Conseil :** utiliser la même librairie `global` en dev et prod. En dev, désactiver l'agrégation Drupal (`/admin/config/development/performance`) pour que les sourcemaps fonctionnent.

---

## `.gitignore` pour un Thème avec Build Pipeline

```gitignore
# Dépendances Node.js — NE JAMAIS commiter
node_modules/

# Assets compilés — 2 stratégies :
# Option A : dist/ non commité (rebuilder en CI)
# dist/

# Option B : dist/ commité (déploiement sans CI Node.js)
# Dans ce cas, ne pas ignorer dist/

# Fichiers de debug/cache npm
npm-debug.log*
yarn-debug.log*
yarn-error.log*
.npm

# Sourcemaps (optionnel — utiles en dev, inutiles en prod)
# dist/**/*.map

# Fichiers d'environnement local
.env.local
.env.development.local
```

**Recommandation pour Drupal :** commiter `dist/` dans le dépôt git. Cela simplifie le déploiement (pas besoin de Node.js sur le serveur ou en CI), au prix d'un dépôt légèrement plus lourd. Pour les équipes avec CI, exclure `dist/` et rebuilder en pipeline.

---

## Scripts npm Utiles

```json
{
  "scripts": {
    "watch":       "gulp watch",
    "build":       "NODE_ENV=production gulp build",
    "dev":         "gulp default",
    "lint:css":    "stylelint 'src/scss/**/*.scss' --fix",
    "lint:js":     "eslint 'src/js/**/*.js' --fix",
    "lint":        "npm run lint:css && npm run lint:js",
    "clean":       "rm -rf dist/",
    "reinstall":   "rm -rf node_modules/ && npm install",
    "bs-version":  "node -e \"console.log(require('bootstrap/package.json').version)\""
  }
}
```

```bash
# Développement quotidien
npm run watch        # Lancer le watch (SCSS+JS → dist/)

# Avant un commit / déploiement
npm run lint         # Vérifier qualité CSS et JS
npm run build        # Build de production (minifié, pas de sourcemaps)

# En cas de problème
npm run clean        # Vider dist/
npm run reinstall    # Réinstaller node_modules proprement
```

---

## Pipeline Gulp pour un Module Custom avec SCSS

Ce pattern isole le build CSS au niveau du module — utile quand un module apporte ses propres composants visuels :

```javascript
// modules/custom/dg_interactive_map/gulpfile.js
const gulp   = require('gulp');
const sass   = require('gulp-sass')(require('sass'));
const maps   = require('gulp-sourcemaps');
const post   = require('gulp-postcss');
const auto   = require('autoprefixer');

const paths = {
  scss: 'src/scss/**/*.scss',
  main: 'src/scss/dg-interactive-map.scss',
  dest: 'css',
};

function styles() {
  return gulp.src(paths.main)
    .pipe(maps.init())
    .pipe(sass({ outputStyle: 'expanded' }).on('error', sass.logError))
    .pipe(post([auto()]))
    .pipe(maps.write('.'))
    .pipe(gulp.dest(paths.dest));
}

exports.watch   = () => gulp.watch(paths.scss, styles);
exports.build   = styles;
exports.default = styles;
```

```yaml
# dg_interactive_map.libraries.yml
map-styles:
  version: 1.0
  css:
    component:
      css/dg-interactive-map.css: {}
```

---

## Pipeline Vite — Recommandé pour les Nouveaux Projets (2025+)

Vite est devenu le bundler de référence : démarrage instantané en dev (ESM natif), build de production ultra-rapide via Rollup. Recommandé pour tout nouveau thème.

### `package.json` (variante Vite)

```json
{
  "name": "montheme",
  "version": "1.0.0",
  "scripts": {
    "dev": "vite build --watch",
    "build": "vite build",
    "preview": "vite preview"
  },
  "devDependencies": {
    "vite": "^5.2.0",
    "sass": "^1.75.0",
    "autoprefixer": "^10.4.19",
    "postcss": "^8.4.38"
  },
  "dependencies": {
    "bootstrap": "^5.3.3"
  }
}
```

### `vite.config.js`

```javascript
import { defineConfig } from 'vite';
import { resolve } from 'path';

export default defineConfig({
  // Pas de server dev (Drupal gère le serving)
  build: {
    outDir: 'dist',
    emptyOutDir: true,
    rollupOptions: {
      input: {
        main: resolve(__dirname, 'src/js/main.js'),  // JS + SCSS importé ici
      },
      output: {
        // Noms de fichiers fixes pour .libraries.yml
        entryFileNames: 'js/[name].bundle.js',
        chunkFileNames: 'js/[name].chunk.js',
        assetFileNames: (assetInfo) => {
          if (/\.css$/.test(assetInfo.name)) {
            return 'css/[name][extname]';
          }
          return 'assets/[name][extname]';
        },
      },
    },
    sourcemap: process.env.NODE_ENV !== 'production',
    minify: process.env.NODE_ENV === 'production' ? 'esbuild' : false,
  },
  css: {
    preprocessorOptions: {
      scss: {
        includePaths: ['node_modules'],  // Accès à Bootstrap SCSS
      },
    },
    postcss: {
      plugins: [
        (await import('autoprefixer')).default(),
      ],
    },
  },
});
```

### `src/js/main.js` — Point d'entrée

```javascript
// Importer Bootstrap SCSS + styles custom
import '../scss/main.scss';
import 'bootstrap/dist/js/bootstrap.bundle.min.js';  // Bootstrap JS (optionnel)

// Drupal behaviors ES6 — Vite bundlera correctement
(function (Drupal, once) {
  'use strict';

  Drupal.behaviors.montheme = {
    attach(context, settings) {
      once('montheme-init', '[data-montheme]', context).forEach((el) => {
        // Logique custom
      });
    },
  };
})(Drupal, once);
```

### Comparatif Gulp vs Webpack vs Vite

| | Gulp | Webpack 5 | Vite 5 |
|---|---|---|--------|
| Vitesse dev (watch) | Lente (rebuild complet) | Lente (HMR) | **Instantanée** (ESM natif) |
| Config | Verbose (code JS) | Très verbose | **Minimale** |
| Build prod | OK | OK | **Très rapide** (esbuild) |
| Adoption 2025 | En déclin | Stable | **En forte croissance** |
| SCSS | Via plugins | Via loaders | **Natif** (plugin sass) |
| Recommandation | Projets legacy | Projets complexes | **Nouveaux projets** |

> **Migration depuis Webpack :** remplacer `webpack.config.js` par `vite.config.js`, adapter les points d'entrée, supprimer les loaders Webpack. La configuration Vite est ~5× plus courte.

---

## Troubleshooting Build Pipeline

| Problème | Cause probable | Solution |
|----------|---------------|---------|
| `sass: command not found` | `sass` non installé | `npm install sass` |
| Bootstrap SCSS non trouvé | `includePaths` manquant | Ajouter `includePaths: ['node_modules']` dans la config sass |
| Variables Bootstrap non surchargées | Import avant les variables | Définir les variables AVANT `@import "bootstrap/scss/bootstrap"` |
| Sourcemaps absents dans les devtools | Agrégation Drupal active | Désactiver l'agrégation sur `/admin/config/development/performance` |
| CSS non rechargé en watch | Fichier `dist/css/` non mis à jour | Vérifier les logs du watcher, vérifier le chemin dans gulp |
| Docker node_modules manquants | Volume monté AVANT `npm install` | `docker compose run --rm node npm install` puis relancer |
| `dist/` vide après `npm run build` | Erreur silencieuse du build | `npm run build` sans `2>/dev/null` pour voir les erreurs |
| `minified: true` mais Drupal modifie le fichier | Agrégation active ignore ce flag | Désactiver l'agrégation en dev ou vérifier la config Drupal |
