# Next.js 16 - Nouvelles Fonctionnalités et APIs

Guide d'utilisation des nouvelles fonctionnalités de Next.js 16 dans ce projet.

## 1. Partial Pre-Rendering (PPR)

PPR combine le meilleur du rendu statique et dynamique pour une navigation instantanée.

### Configuration

Dans `next.config.ts` :
```typescript
// Cache Components (includes Partial Pre-Rendering)
cacheComponents: true,
```

**Note** : Dans Next.js 16, `experimental.ppr` a été fusionné dans `cacheComponents` qui est maintenant une option stable.

### Utilisation dans les Pages

```typescript
// app/dashboard/page.tsx
export const experimental_ppr = true; // Active PPR pour cette page

export default async function DashboardPage() {
  return (
    <div>
      {/* Partie statique - pré-rendue */}
      <Header />

      {/* Partie dynamique - streamed */}
      <Suspense fallback={<Skeleton />}>
        <DynamicContent />
      </Suspense>
    </div>
  );
}
```

### Avantages
- ✅ Time to First Byte (TTFB) ultra-rapide
- ✅ Navigation instantanée
- ✅ Meilleure expérience utilisateur
- ✅ SEO optimisé

## 2. Improved Caching APIs

### `revalidateTag()` - Amélioré

Revalide le cache pour un tag spécifique.

```typescript
// app/actions.ts
'use server'

import { revalidateTag } from 'next/cache';

export async function updatePost(id: string) {
  // Mettre à jour le post en base de données
  await db.post.update({ where: { id }, data: { ... } });

  // Revalider le cache pour ce tag
  revalidateTag(`post-${id}`);
  revalidateTag('posts-list');
}
```

### `updateTag()` - Nouvelle API

Mise à jour du cache par tag sans revalidation complète.

```typescript
// app/actions.ts
'use server'

import { updateTag } from 'next/cache';

export async function incrementPostViews(id: string) {
  await db.post.increment({ where: { id }, views: 1 });

  // Mise à jour du cache sans revalidation complète
  updateTag(`post-${id}-views`);
}
```

### Utilisation avec fetch

```typescript
// app/lib/api.ts
export async function getPost(id: string) {
  const res = await fetch(`https://api.example.com/posts/${id}`, {
    next: {
      tags: [`post-${id}`, 'posts-list'],
      revalidate: 3600, // 1 heure
    },
  });

  return res.json();
}
```

### Cache Granulaire

```typescript
// app/posts/[id]/page.tsx
import { unstable_cache } from 'next/cache';

const getCachedPost = unstable_cache(
  async (id: string) => {
    return await db.post.findUnique({ where: { id } });
  },
  ['post'], // Cache key
  {
    tags: ['posts'],
    revalidate: 3600,
  }
);

export default async function PostPage({ params }: { params: { id: string } }) {
  const post = await getCachedPost(params.id);

  return <Post data={post} />;
}
```

## 3. Enhanced Routing

### Layout Deduplication

Next.js 16 optimise automatiquement les layouts partagés pour éviter les re-renders inutiles.

```typescript
// app/layout.tsx
export default function RootLayout({ children }: { children: React.ReactNode }) {
  // Ce layout ne sera pas re-rendu lors de la navigation
  // entre les pages qui partagent le même layout
  return (
    <html lang="fr">
      <body>
        <Header />
        {children}
        <Footer />
      </body>
    </html>
  );
}
```

### Incremental Prefetching

Préchargement intelligent des routes pour une navigation plus rapide.

```typescript
// app/components/Navigation.tsx
import Link from 'next/link';

export function Navigation() {
  return (
    <nav>
      {/* Prefetch automatique au hover */}
      <Link href="/dashboard" prefetch={true}>
        Dashboard
      </Link>

      {/* Prefetch uniquement au hover */}
      <Link href="/settings" prefetch={false}>
        Settings
      </Link>
    </nav>
  );
}
```

### Optimisation des Navigations

```typescript
// app/lib/navigation.ts
import { useRouter } from 'next/navigation';

export function useOptimizedNavigation() {
  const router = useRouter();

  const navigate = (path: string) => {
    // Next.js 16 optimise automatiquement :
    // - Cache des layouts
    // - Prefetch des routes
    // - Transitions fluides
    router.push(path);
  };

  return { navigate };
}
```

## 4. use cache Directive

Nouvelle directive pour le caching de composants.

```typescript
// app/components/ExpensiveComponent.tsx
'use cache';

export async function ExpensiveComponent({ userId }: { userId: string }) {
  // Ce composant sera mis en cache
  const data = await fetchExpensiveData(userId);

  return <div>{data}</div>;
}
```

### Avec Revalidation

```typescript
'use cache';

export async function CachedData() {
  const data = await fetch('https://api.example.com/data', {
    next: { revalidate: 60 }, // Revalide après 60 secondes
  });

  return <DataDisplay data={data} />;
}
```

## 5. Server Actions avec Cache

```typescript
// app/actions/posts.ts
'use server'

import { revalidateTag, revalidatePath } from 'next/cache';

export async function createPost(formData: FormData) {
  const title = formData.get('title') as string;
  const content = formData.get('content') as string;

  const post = await db.post.create({
    data: { title, content },
  });

  // Revalide le cache
  revalidateTag('posts-list');
  revalidatePath('/posts');

  return { success: true, post };
}

export async function deletePost(id: string) {
  await db.post.delete({ where: { id } });

  // Revalide plusieurs tags
  revalidateTag(`post-${id}`);
  revalidateTag('posts-list');
  revalidatePath('/posts');

  return { success: true };
}
```

## Meilleures Pratiques

### 1. Utiliser PPR pour les Pages Mixtes

```typescript
// Activer PPR pour les pages avec contenu statique + dynamique
export const experimental_ppr = true;

export default async function Page() {
  return (
    <>
      <StaticHeader />
      <Suspense fallback={<Loading />}>
        <DynamicUserContent />
      </Suspense>
    </>
  );
}
```

### 2. Tags de Cache Cohérents

```typescript
// Créer des conventions de nommage pour les tags
const CACHE_TAGS = {
  posts: {
    list: 'posts-list',
    detail: (id: string) => `post-${id}`,
    byUser: (userId: string) => `user-${userId}-posts`,
  },
  users: {
    list: 'users-list',
    detail: (id: string) => `user-${id}`,
  },
};

// Utiliser dans les Server Actions
revalidateTag(CACHE_TAGS.posts.detail(postId));
```

### 3. Revalidation Stratégique

```typescript
// Combiner revalidateTag et revalidatePath
export async function updateUser(id: string, data: UserData) {
  await db.user.update({ where: { id }, data });

  // Revalide le tag pour les requêtes API
  revalidateTag(`user-${id}`);

  // Revalide le path pour les pages
  revalidatePath(`/users/${id}`);
  revalidatePath('/users'); // Liste des utilisateurs
}
```

### 4. Cache Granulaire avec unstable_cache

```typescript
// Créer des fonctions de cache réutilisables
import { unstable_cache } from 'next/cache';

export const getCachedUser = unstable_cache(
  async (id: string) => db.user.findUnique({ where: { id } }),
  ['user'],
  { tags: ['users'], revalidate: 300 }
);

export const getCachedPosts = unstable_cache(
  async (userId: string) => db.post.findMany({ where: { userId } }),
  ['user-posts'],
  { tags: ['posts'], revalidate: 60 }
);
```

## Performance Tips

1. **PPR** : Activez `experimental_ppr` pour les pages avec mix statique/dynamique
2. **Tags** : Utilisez des tags cohérents pour un contrôle précis du cache
3. **Revalidation** : Revalidez uniquement ce qui est nécessaire
4. **Suspense** : Utilisez Suspense pour streamer le contenu dynamique
5. **unstable_cache** : Pour les opérations de base de données coûteuses

## Migration depuis Next.js 15

### Avant (Next.js 15)
```typescript
// Revalidation globale
export const revalidate = 60;

export default async function Page() {
  const data = await fetch('https://api.example.com/data');
  return <div>{data}</div>;
}
```

### Après (Next.js 16)
```typescript
// Cache granulaire avec tags
export const experimental_ppr = true;

export default async function Page() {
  const data = await fetch('https://api.example.com/data', {
    next: {
      tags: ['api-data'],
      revalidate: 60,
    },
  });

  return <div>{data}</div>;
}
```

## Ressources

- [Next.js 16 Documentation](https://nextjs.org/docs)
- [PPR Guide](https://nextjs.org/docs/app/building-your-application/rendering/partial-prerendering)
- [Caching Guide](https://nextjs.org/docs/app/building-your-application/caching)
- [Data Fetching](https://nextjs.org/docs/app/building-your-application/data-fetching)
