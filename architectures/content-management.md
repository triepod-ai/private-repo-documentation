# Content Management Architecture

## Overview

Scalable content processing system handling blog posts, MDX compilation, image optimization, and SEO metadata generation.

## System Design

```
┌─────────────────────────────────────────────────────────┐
│               Content Input Sources                     │
│  • Admin UI  • API  • Git  • CMS Integration           │
└────────────────────────┬────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────┐
│            Content Processing Pipeline                   │
│  ┌──────────┐  ┌───────────┐  ┌──────────────────┐    │
│  │ Validate │→ │  Process  │→ │   Optimize       │    │
│  │ Content  │  │  MDX/MD   │  │   Images         │    │
│  └──────────┘  └───────────┘  └──────────────────┘    │
│         │              │                │               │
│         ▼              ▼                ▼               │
│  ┌──────────┐  ┌───────────┐  ┌──────────────────┐    │
│  │ Generate │  │  Extract  │  │   Create         │    │
│  │   SEO    │  │  Metadata │  │   Static         │    │
│  └──────────┘  └───────────┘  └──────────────────┘    │
└────────────────────────┬────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────┐
│                  Storage Layer                           │
│  • Database (metadata)  • File System (content)         │
│  • CDN (images)  • Search Index                         │
└──────────────────────────────────────────────────────────┘
```

## Core Processing

### MDX Compilation

```typescript
// lib/mdx-compiler.ts

import { compile } from '@mdx-js/mdx'
import remarkGfm from 'remark-gfm'
import rehypeHighlight from 'rehype-highlight'
import rehypeSlug from 'rehype-slug'

export async function compileMDX(source: string) {
  const compiled = await compile(source, {
    remarkPlugins: [remarkGfm],
    rehypePlugins: [
      rehypeHighlight,
      rehypeSlug,
      [rehypeAutolinkHeadings, { behavior: 'wrap' }]
    ]
  })

  return {
    code: String(compiled),
    frontmatter: compiled.data.frontmatter
  }
}
```

### Image Optimization

```typescript
// lib/image-processor.ts

import sharp from 'sharp'
import path from 'path'

export async function optimizeImage(
  inputPath: string,
  outputDir: string
): Promise<OptimizedImage> {
  const image = sharp(inputPath)
  const metadata = await image.metadata()

  // Generate multiple sizes
  const sizes = [640, 750, 828, 1080, 1200, 1920]
  const outputs: ImageOutput[] = []

  for (const width of sizes) {
    if (width > metadata.width!) continue

    const outputPath = path.join(
      outputDir,
      `${path.basename(inputPath, path.extname(inputPath))}-${width}w.webp`
    )

    await image
      .resize(width, null, { withoutEnlargement: true })
      .webp({ quality: 85 })
      .toFile(outputPath)

    outputs.push({ width, path: outputPath })
  }

  return {
    original: inputPath,
    optimized: outputs,
    metadata: {
      width: metadata.width!,
      height: metadata.height!,
      format: metadata.format!
    }
  }
}
```

### SEO Generation

```typescript
// lib/seo-generator.ts

export function generateSEOMetadata(post: BlogPost) {
  return {
    title: `${post.title} | Company Blog`,
    description: post.excerpt || extractExcerpt(post.content, 160),
    openGraph: {
      title: post.title,
      description: post.excerpt,
      type: 'article',
      publishedTime: post.publishedAt?.toISOString(),
      authors: [post.author.name],
      images: [
        {
          url: post.coverImage || '/default-og-image.jpg',
          width: 1200,
          height: 630,
          alt: post.title
        }
      ]
    },
    twitter: {
      card: 'summary_large_image',
      title: post.title,
      description: post.excerpt,
      images: [post.coverImage || '/default-og-image.jpg']
    },
    alternates: {
      canonical: `https://example.com/blog/${post.slug}`
    }
  }
}

function extractExcerpt(content: string, maxLength: number): string {
  const text = content.replace(/[#*_\[\]]/g, '').trim()
  return text.length > maxLength
    ? text.substring(0, maxLength) + '...'
    : text
}
```

## Content API

```typescript
// api/posts.ts

export async function getPosts(options: GetPostsOptions = {}) {
  const { published = true, limit, offset, tags } = options

  return prisma.blogPost.findMany({
    where: {
      published,
      ...(tags && { tags: { some: { name: { in: tags } } } })
    },
    include: {
      author: { select: { name: true, email: true } },
      tags: true
    },
    orderBy: { publishedAt: 'desc' },
    take: limit,
    skip: offset
  })
}

export async function getPostBySlug(slug: string) {
  const post = await prisma.blogPost.findUnique({
    where: { slug },
    include: {
      author: true,
      tags: true
    }
  })

  if (!post) return null

  const { code } = await compileMDX(post.content)

  return {
    ...post,
    compiledContent: code
  }
}

export async function createPost(data: CreatePostInput) {
  // Validate content
  await validateMDXContent(data.content)

  // Generate slug
  const slug = await generateUniqueSlug(data.title)

  // Create post
  return prisma.blogPost.create({
    data: {
      ...data,
      slug,
      published: false
    }
  })
}
```

## Caching Strategy

```typescript
// lib/content-cache.ts

export class ContentCache {
  private redis: Redis

  async getPost(slug: string): Promise<BlogPost | null> {
    // Try cache first
    const cached = await this.redis.get(`post:${slug}`)
    if (cached) {
      return JSON.parse(cached)
    }

    // Fetch from database
    const post = await getPostBySlug(slug)
    if (!post) return null

    // Cache for 1 hour
    await this.redis.setex(
      `post:${slug}`,
      3600,
      JSON.stringify(post)
    )

    return post
  }

  async invalidatePost(slug: string) {
    await this.redis.del(`post:${slug}`)
  }

  async invalidateList(tags?: string[]) {
    const keys = tags
      ? tags.map(tag => `posts:tag:${tag}`)
      : ['posts:all']

    await this.redis.del(...keys)
  }
}
```

## Static Site Generation

```typescript
// app/blog/[slug]/page.tsx

export async function generateStaticParams() {
  const posts = await getPosts({ published: true })

  return posts.map((post) => ({
    slug: post.slug
  }))
}

export async function generateMetadata({ params }): Promise<Metadata> {
  const post = await getPostBySlug(params.slug)
  if (!post) return {}

  return generateSEOMetadata(post)
}

export default async function BlogPost({ params }) {
  const post = await getPostBySlug(params.slug)

  if (!post) notFound()

  return (
    <article>
      <h1>{post.title}</h1>
      <MDXContent code={post.compiledContent} />
    </article>
  )
}
```

## Related Patterns

- [Unified Ecosystem](unified-ecosystem.md)
- [Testing Strategies](../patterns/testing-strategies.md)

---

**Production Scale**: Processes 25+ blog posts with image optimization, SEO generation, and static site generation in <3 minutes.
