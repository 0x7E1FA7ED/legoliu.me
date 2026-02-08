# AI Copilot Instructions for legoliu.me

## Project Overview
**AstroPaper** is a performance-optimized Astro blog theme built with TypeScript and Tailwind CSS. The architecture centers on Astro's Content Collections API for type-safe Markdown blog posts, with dynamic OG image generation and full-text search via Pagefind.

## Critical Architecture

### Content & Data Flow
- **Blog posts**: `src/data/blog/*.md` with Astro Content Collections schema in [content.config.ts](../src/content.config.ts)
- **Schema**: Requires `title`, `description`, `pubDatetime`, `tags`; supports `draft`, `featured`, `ogImage`, `modDatetime`, `timezone`
- **Drafts**: Set `draft: true` in frontmatter to exclude from builds
- **Post exclusion**: Files prefixed with `_` (e.g., `_releases/`) are ignored (glob pattern: `**/[^_]*.md`)
- **Dynamic routing**: Post slugs generated via [getPath()](../src/utils/getPath.ts) from `post.id` and `filePath`

### Layout Structure
- **Layouts**: [Layout.astro](../src/layouts/Layout.astro) (base), [PostDetails.astro](../src/layouts/PostDetails.astro) (post pages)
- **Pages**: Dynamic routes in `src/pages/posts/[...slug]/index.astro` generate paths for all non-draft posts
- **Components**: Async/sync `.astro` components in `src/components/` support view transitions and class:list binding

### OG Image Generation
- **Enabled by default**: Controlled by `SITE.dynamicOgImage` in [config.ts](../src/config.ts)
- **Process**: [generateOgImages.ts](../src/utils/generateOgImages.ts) uses Satori (SVG to image) + Resvg (SVG render)
- **Templates**: `src/utils/og-templates/post.js` and `site.js` render custom SVG layouts
- **Fallback**: If `ogImage` frontmatter is set (string or ImageMetadata), it uses that instead

### Site Configuration
[config.ts](../src/config.ts) is the single source of truth:
```typescript
export const SITE = {
  website, author, desc, title, // basic metadata
  postPerIndex, postPerPage, // pagination
  dynamicOgImage, // OG strategy
  showArchives, showBackButton, // feature flags
  editPost: { enabled, url }, // GitHub edit links
  lang, timezone, // i18n/localization
}
```

## Build & Development Workflow

| Command | Purpose |
|---------|---------|
| `npm run dev` | Start Astro dev server (HMR enabled) |
| `npm run build` | Type-check → Astro build → Pagefind index → Copy pagefind to public/ |
| `npm run format:check` | Verify Prettier compliance |
| `npm run lint` | Run ESLint (fails on `console` usage) |

**Critical**: Build includes `astro check` (TypeScript) before bundling; failing type checks block the build.

## Code Patterns & Conventions

### Post Processing
```typescript
// Pattern: Filter & sort posts
import getSortedPosts from "@/utils/getSortedPosts";
const posts = await getCollection("blog", ({ data }) => !data.draft);
const sorted = getSortedPosts(posts); // filters + sorts by modDatetime|pubDatetime DESC
```

### URL Generation
Use [getPath()](../src/utils/getPath.ts) for consistent post URLs:
```typescript
getPath(post.id, post.filePath, false) // Returns slugified path
```
Do **not** hardcode post paths; this utility handles nested directory structures.

### Component Props
Astro components use `type Props` with spread destructuring:
```typescript
type Props = {
  variant?: "h2" | "h3";
} & CollectionEntry<"blog">; // Spreads id, data, filePath, collection

const { variant, data, filePath, ...rest } = Astro.props;
```

### Styling
- **Tailwind CSS** (v4) with `@tailwindcss/typography` plugin
- **Prettier plugin**: Auto-sorts classes
- **No inline styles** unless absolutely necessary (use `class:list` instead)
- **View transitions**: Use `viewTransitionName` on unique elements to enable smooth transitions

### Search & Navigation
- **Pagefind**: Generated at build time to `dist/pagefind/`; copied to `public/` post-build
- **Archives**: Conditional rendering via `SITE.showArchives` flag
- **Tag filtering**: [getUniqueTags.ts](../src/utils/getUniqueTags.ts) and [getPostsByTag.ts](../src/utils/getPostsByTag.ts)

## Critical Dependencies
- `@astrojs/rss`, `@astrojs/sitemap` → RSS/sitemap generation
- `satori`, `@resvg/resvg-js` → OG image rendering
- `@shikijs/transformers` → Code syntax highlighting (file names, diffs, highlights)
- `pagefind` → Full-text search indexing

## Linting & Formatting
- **ESLint**: Forbids `console.log()` (dev-friendly; use during debugging but remove for commits)
- **Prettier**: Enforces format across TS/Astro/CSS/Markdown with 2-space indents
- **Pre-commit check**: Run `npm run format:check && npm run lint` to validate

## Common Pitfalls
1. **Frontmatter required fields**: Posts without `title`, `description`, or `pubDatetime` will fail schema validation
2. **Draft posts in navigation**: Always filter with `!data.draft` in collection queries (built-in utility does this)
3. **OG image caching**: If modifying templates, delete `dist/` and rebuild to refresh images
4. **Timezone mismatches**: Use IANA timezone format (e.g., `Asia/Hong_Kong`) in post frontmatter for accurate display
5. **Import aliases**: Use `@/` for `src/` (configured in Astro); do not use relative imports across directories

## Files to Reference When Contributing
- Configuration: [astro.config.ts](../astro.config.ts), [content.config.ts](../src/content.config.ts), [config.ts](../src/config.ts)
- Utilities: [src/utils/](../src/utils/) (post filtering, slug generation, path resolution)
- Layout: [PostDetails.astro](../src/layouts/PostDetails.astro) (post rendering), [Layout.astro](../src/layouts/Layout.astro) (base template)
- Example post: [src/data/blog/chat-with-gemini3.md](../src/data/blog/chat-with-gemini3.md)
