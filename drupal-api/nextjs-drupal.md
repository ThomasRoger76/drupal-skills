---
name: drupal-api — next.js + drupal
description: Intégration complète Next.js + Drupal avec le module drupal/next et le package next-drupal - setup, preview mode, ISR, revalidation, et authentification.
---

# Next.js + Drupal — Intégration Complète

## Installation Côté Drupal

> **Côté Drupal : Docker natif (`docker compose exec php ...`), jamais DDEV.**

```bash
docker compose exec php composer require drupal/next drupal/simple_oauth drupal/decoupled_router
docker compose exec php drush en next next_jsonapi simple_oauth decoupled_router -y
```

```bash
# Générer les clés RSA pour Simple OAuth (commande dédiée, recommandée)
docker compose exec php drush simple-oauth:generate-keys ../oauth-keys
```

---

## Configuration Drupal (next module)

```yaml
# 1. Créer un "Next.js site" dans Drupal
# /admin/config/services/next/sites/add

# config/install/next.next_site.mon_frontend.yml
langcode: fr
status: true
id: mon_frontend
label: 'Frontend Next.js'
base_url: 'http://localhost:3000'     # URL du frontend en dev
preview_url: 'http://localhost:3000/api/preview'
preview_secret: 'MON_SECRET_PREVIEW'  # Partagé avec Next.js
```

```yaml
# 2. Créer le Consumer OAuth pour Next.js
# /admin/config/services/consumer/add

# Un consumer pour le backend (SSR/ISR) — client_credentials
# Un consumer pour le preview (avec utilisateur editor) — client_credentials
```

---

## Installation Côté Next.js

```bash
npx create-next-app@latest mon-frontend
cd mon-frontend
npm install next-drupal
```

### Configuration `next.config.js`

```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  images: {
    domains: [
      process.env.NEXT_PUBLIC_DRUPAL_BASE_URL?.replace('https://', '') || 'localhost',
    ],
  },
};

module.exports = nextConfig;
```

### Variables d'environnement `.env.local`

```env
NEXT_PUBLIC_DRUPAL_BASE_URL=https://mon-drupal.com
DRUPAL_CLIENT_ID=uuid-du-consumer
DRUPAL_CLIENT_SECRET=mon-secret-oauth
DRUPAL_PREVIEW_SECRET=MON_SECRET_PREVIEW
DRUPAL_REVALIDATE_SECRET=MON_SECRET_REVALIDATION
```

### Initialiser next-drupal `lib/drupal.ts`

```typescript
import { DrupalClient } from "next-drupal"

export const drupal = new DrupalClient(
  process.env.NEXT_PUBLIC_DRUPAL_BASE_URL!,
  {
    auth: {
      clientId: process.env.DRUPAL_CLIENT_ID!,
      clientSecret: process.env.DRUPAL_CLIENT_SECRET!,
    },
    frontPage: "/node",
    debug: process.env.NODE_ENV === "development",
  }
)
```

---

## Fetcher les Articles (Static Generation)

```typescript
// app/articles/[slug]/page.tsx (Next.js 14 App Router)
import { drupal } from "@/lib/drupal"
import { DrupalNode } from "next-drupal"
import { notFound } from "next/navigation"

interface ArticlePageProps {
  params: {
    slug: string[]
  }
}

export async function generateStaticParams() {
  const articles = await drupal.getResourceCollectionFromContext<DrupalNode[]>(
    "node--article",
    {
      params: {
        "filter[status]": 1,
        "filter[langcode]": "fr",
        "fields[node--article]": "title,path",
        "page[limit]": 50,
      },
    }
  )

  return articles.map((article) => ({
    slug: article.path?.alias?.split("/").filter(Boolean) || [article.id],
  }))
}

export default async function ArticlePage({ params }: ArticlePageProps) {
  const path = `/${params.slug.join("/")}`

  // Résoudre le chemin vers le node Drupal
  const resource = await drupal.translatePathFromContext({
    path,
  })

  if (!resource || resource.entity.type !== "node--article") {
    notFound()
  }

  // Charger l'article complet
  const article = await drupal.getResource<DrupalNode>("node--article", resource.entity.id, {
    params: {
      "fields[node--article]": "title,body,field_image,field_tags,created,langcode",
      "include": "field_image,field_tags",
    },
  })

  return (
    <article>
      <h1>{article.title}</h1>
      <div dangerouslySetInnerHTML={{ __html: article.body?.processed || "" }} />
    </article>
  )
}

// Revalidation ISR — reconstruire après N secondes
export const revalidate = 3600  // 1 heure
```

---

## Preview Mode (Brouillons en temps réel)

### Route API de preview `app/api/preview/route.ts`

```typescript
import { drupal } from "@/lib/drupal"
import { NextRequest, NextResponse } from "next/server"

export async function GET(request: NextRequest) {
  const searchParams = request.nextUrl.searchParams

  const secret = searchParams.get("secret")
  const slug = searchParams.get("slug")
  const resourceVersion = searchParams.get("resourceVersion")

  if (secret !== process.env.DRUPAL_PREVIEW_SECRET) {
    return NextResponse.json({ error: "Invalid token" }, { status: 401 })
  }

  // Activer le draft mode Next.js
  const draftModeEnabled = await import("next/headers").then(
    (mod) => mod.draftMode
  )
  draftModeEnabled().enable()

  // Rediriger vers la page de l'article en brouillon
  return NextResponse.redirect(new URL(slug || "/", request.url))
}
```

### Désactiver le preview `app/api/disable-draft/route.ts`

```typescript
import { draftMode } from "next/headers"
import { redirect } from "next/navigation"

export async function GET() {
  draftMode().disable()
  redirect("/")
}
```

---

## Revalidation on-demand depuis Drupal

```typescript
// app/api/revalidate/route.ts
import { revalidatePath, revalidateTag } from "next/cache"
import { NextRequest, NextResponse } from "next/server"

export async function POST(request: NextRequest) {
  const secret = request.headers.get("x-revalidate-secret")

  if (secret !== process.env.DRUPAL_REVALIDATE_SECRET) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 })
  }

  const body = await request.json()
  const { path, tag } = body

  if (tag) {
    revalidateTag(tag)
  }

  if (path) {
    revalidatePath(path)
  }

  return NextResponse.json({
    revalidated: true,
    now: Date.now(),
  })
}
```

**Depuis Drupal (webhook) :**
```php
// Déclencher la revalidation quand un article est sauvegardé
function mon_module_node_update(\Drupal\node\NodeInterface $node): void {
  if ($node->bundle() !== 'article') {
    return;
  }

  $frontend_url = \Drupal::config('mon_module.settings')->get('frontend_url');
  $secret = \Drupal::config('mon_module.settings')->get('revalidate_secret');

  $client = \Drupal::httpClient();
  $client->post("$frontend_url/api/revalidate", [
    'json' => [
      'path' => $node->toUrl()->toString(),
      'tag' => 'article-' . $node->id(),
    ],
    'headers' => ['X-Revalidate-Secret' => $secret],
    'timeout' => 5,
  ]);
}
```

---

## Commandes Utiles Next.js + Drupal

```bash
# Développement
npm run dev   # Next.js sur localhost:3000

# Build de production
npm run build && npm start

# Forcer une regénération des pages statiques
curl -X POST http://localhost:3000/api/revalidate \
  -H "x-revalidate-secret: MON_SECRET" \
  -H "Content-Type: application/json" \
  -d '{"path": "/articles/mon-article"}'

# Debug les requêtes JSON:API
NEXT_PUBLIC_DRUPAL_BASE_URL=... DRUPAL_CLIENT_ID=... \
  node -e "
const { DrupalClient } = require('next-drupal');
const d = new DrupalClient(process.env.NEXT_PUBLIC_DRUPAL_BASE_URL, {
  auth: { clientId: process.env.DRUPAL_CLIENT_ID, clientSecret: process.env.DRUPAL_CLIENT_SECRET }
});
d.getResourceCollection('node--article', { params: { 'page[limit]': 1 } })
  .then(r => console.log(JSON.stringify(r, null, 2)));
"
```
